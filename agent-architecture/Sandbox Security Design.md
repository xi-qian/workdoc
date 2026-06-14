# Sandbox Security Design

> 分析对象：Hermes Agent 的沙箱安全机制
> 属于 [[Group Isolation Design]] 的配套安全方案

## Part 3: 沙箱安全设计与群组隔离

### 3.1 当前沙箱架构

Hermes 有三层沙箱机制，安全性逐层递增：

```
LLM 要执行操作
    │
    ├── execute_code tool?
    │     └── 进程隔离 + 环境变量剥离 + 工具白名单 + 输出脱敏
    │         └── terminal backend 是 Docker?
    │               └── 容器隔离 + cap-drop ALL + PID 限制 + 网络隔离
    │
    └── terminal tool (直接执行)?
          └── approval.py 模式匹配 → 危险命令拦截/人工审批
```

### 3.2 三层沙箱详解

#### 3.2.1 execute_code 沙箱（轻量，本地进程隔离）

**文件**: `tools/code_execution_tool.py`

LLM 写 Python 脚本，通过 UDS socket 回调父进程的有限工具子集执行。

**隔离措施**：

| 措施 | 实现 |
|------|------|
| 环境变量剥离 | 含 `KEY/TOKEN/SECRET/PASSWORD/CREDENTIAL` 的变量全部移除 |
| 安全前缀白名单 | 只传递 `PATH/HOME/USER/LANG/LC_/TERM/TMPDIR/PYTHONPATH` |
| 工具白名单 | 仅 7 个: `web_search/web_extract/read_file/write_file/search_files/patch/terminal` |
| terminal 参数限制 | 禁止 `background/pty/notify_on_complete/watch_patterns` |
| 工具调用次数限制 | 默认 50 次 |
| 执行超时 | 默认 300 秒 |
| 输出截断 | stdout 50KB, stderr 10KB |
| ANSI 清理 | `strip_ansi()` |
| 敏感信息脱敏 | `redact_sensitive_text()` |
| Profile HOME 隔离 | 子进程 `$HOME` → `{HERMES_HOME}/home/` |
| PID 隔离 | `os.setsid()` 独立进程组 |

**不是 OS 级沙箱** — 子进程共享内核和文件系统。通过环境变量剥离实现网络不可达（无法联网），但可通过 `terminal` stub 执行任意 shell 命令。

#### 3.2.2 Docker 容器沙箱（重量，完整 OS 隔离）

**文件**: `tools/environments/docker.py`

当 terminal backend 配置为 Docker 时，所有命令在容器内执行。

**安全加固**（`_SECURITY_ARGS`）：

```python
[
    "--cap-drop", "ALL",                    # 丢弃所有 Linux capabilities
    "--cap-add", "DAC_OVERRIDE",           # 仅保留: root 可写 bind mount
    "--cap-add", "CHOWN",                  # 仅保留: 包管理器 chown
    "--cap-add", "FOWNER",                 # 仅保留: 包管理器 fchown
    "--security-opt", "no-new-privileges", # 禁止 setuid/setgid 提权
    "--pids-limit", "256",                 # 限制进程数 (防 fork bomb)
    "--tmpfs", "/tmp:rw,nosuid,size=512m",  # /tmp 限制大小, 禁止 suid
    "--tmpfs", "/var/tmp:rw,noexec,nosuid,size=256m",
    "--tmpfs", "/run:rw,noexec,nosuid,size=64m",
    "--init",                              # tini/catatonit PID 1
]
```

| 措施 | 说明 |
|------|------|
| `cap-drop ALL` | 丢弃全部 capabilities，仅加回 3 个最小必要项 |
| `no-new-privileges` | 禁止通过 setuid/setgid 提权 |
| `pids-limit 256` | 最多 256 个进程 |
| `nosuid tmpfs` | /tmp 禁止执行 setuid 程序 |
| `noexec /run` | /run 完全禁止执行 |
| `--network=none` | 可选：完全断网 |
| `--init` | tini 作为 PID 1 回收僵尸 |

**持久化控制**：

```
持久模式:  bind mount ~/.hermes/sandboxes/docker/<id> → /root, /workspace
非持久:    tmpfs（内存文件系统，容器销毁后消失）
```

#### 3.2.3 命令审批（运行时防护）

**文件**: `tools/approval.py`

不依赖沙箱，对所有 terminal 调用的统一防护层：

- 正则匹配危险模式（`rm -rf`, `chmod 777`, `curl` 外传 secrets, 写 `~/.ssh/authorized_keys` 等）
- Per-session 审批状态（contextvars, 线程安全）
- 永久 allowlist（`config.yaml` 中的 `terminal.allow`）
- 智能审批（auxiliary LLM 判断低风险命令自动通过）

### 3.3 群组隔离场景下的沙箱需求

#### 3.3.1 问题：当前沙箱按 profile 隔离，不按群组隔离

当前 Docker 容器是按 profile 创建的（`~/.hermes/sandboxes/docker/`），所有群组共享同一个容器。这意味着：

- 群组 A 的 cron job 创建的临时文件在 `/workspace` 中对群组 B 可见
- 群组 A 执行的命令（pip install 的包）影响后续所有群组
- 如果群组 A 的命令污染了容器状态，群组 B 受影响

#### 3.3.2 方案：按群组 profile 使用独立容器

**核心改动**：DockerEnvironment 创建容器时使用群组 profile 的 sandbox 目录：

```python
# tools/environments/docker.py
class DockerEnvironment(BaseEnvironment):
    def __init__(self, ..., hermes_home_override: str = None, ...):
        # 群组专属 sandbox 目录
        sandbox_base = Path(hermes_home_override) / "sandboxes" if hermes_home_override else get_sandbox_dir()
        ...
```

```python
# tools/terminal_tool.py
def _create_environment(..., hermes_home_override: str = None):
    env = DockerEnvironment(
        ...,
        hermes_home_override=hermes_home_override,
    )
```

透传路径：`AIAgent → terminal tool → _create_environment → DockerEnvironment`

#### 3.3.3 完整隔离后的效果

```
群组 A 发送消息 "安装 xxx 包"
  └── Gateway → _resolve_group_profile → grp-telegram--100xxx
        └── AIAgent(hermes_home_override="~/.hermes/profiles/grp-telegram--100xxx")
              ├── MemoryStore → grp-xxx/memories/MEMORY.md (隔离)
              ├── skills → grp-xxx/skills/ (隔离)
              ├── execute_code
              │     └── 子进程 HOME=grp-xxx/home/ (隔离)
              │     └── terminal backend
              │           └── Docker sandbox
              │                 └── bind mount grp-xxx/sandboxes/docker/ → /workspace (隔离)
              └── external provider → grp-xxx/honcho.json (隔离)
```

#### 3.3.4 沙箱附加安全措施（群聊场景增强）

| 措施 | 说明 | 优先级 |
|------|------|--------|
| 按群组独立容器 | 每个群组 profile 有自己的 Docker 容器 | P0 |
| 网络隔离 `--network=none` | 群聊场景默认断网，防止外连 exfil | P0 |
| 环境变量进一步净化 | 群聊模式下移除 `HOME` 中的用户路径信息 | P1 |
| 输出额外脱敏 | 检测并移除其他群组用户的消息片段 | P1 |
| 文件写入审计 | 记录群组 agent 的所有文件写入操作 | P2 |
| 资源配额 | 按群组限制 CPU/内存/disk 用量 | P2 |

#### 3.3.5 环境变量净化增强

当前 `execute_code` 的环境变量过滤：

```python
_SECRET_SUBSTRINGS = ("KEY", "TOKEN", "SECRET", "PASSWORD", "CREDENTIAL", "PASSWD", "AUTH")
```

群聊场景需要增加：

```python
# 移除可能泄露其他用户信息的变量
_PRIVACY_SUBSTRINGS = _SECRET_SUBSTRINGS + (
    "USERNAME", "USER", "DISPLAY", "REALNAME",
    "TELEGRAM_CHAT_ID", "DISCORD_USER_ID",
    "HONCHO_PEER_NAME",
)
```

### 3.4 execute_code 安全加固建议

当前 execute_code 的 `terminal` stub 允许 LLM 在沙箱内执行任意 shell 命令（因为 `terminal` 在白名单中）。在群聊场景下需要更严格的限制：

#### 3.4.1 方案 A：从白名单移除 terminal

```python
# 群聊模式下 execute_code 的工具白名单
GROUP_SANDBOX_TOOLS = frozenset([
    "web_search",
    "web_extract",
    "read_file",
    "write_file",
    "search_files",
    "patch",
    # "terminal",  ← 群聊模式下移除
])
```

LLM 可以读写文件、搜索文件、打补丁、联网搜索，但不能执行任意 shell 命令。需要 shell 操作时使用 `terminal` tool（走 approval 审批）。

#### 3.4.2 方案 B：terminal stub 添加命令过滤

保留 `terminal` 在白名单中，但在 RPC dispatch 层过滤危险命令：

```python
# code_execution_tool.py — _rpc_server_loop
_TERMINAL_BLOCKED_COMMANDS = re.compile(
    r'(?:(?:curl|wget|nc|ncat|socat|ssh|scp|rsync)\b'
    r'|(?:(?:sudo|su|doas)\b)'
    r'|(?:(?:chmod|chown|chgrp)\s+[0-7]{3,4}\b'
)
```

#### 3.4.3 方案 C：不可变文件系统（Docker only）

Docker 容器内使用 overlayfs 的只读层：

```python
# 工具白名单工具写入到 /workspace（可写层）
# 系统目录和 /home 保持只读
"--read-only",  # 整个根文件系统只读
"--tmpfs", "/workspace:rw,exec,size=10g",  # 仅工作目录可写
```

### 3.5 改动文件清单（沙箱部分，叠加在 Part 2 之上）

| 文件 | 改动 | 行数估计 |
|------|------|---------|
| `tools/environments/docker.py` | 支持 `hermes_home_override` 隔离 sandbox 目录 | ~10 |
| `tools/terminal_tool.py` | `_create_environment` 透传 override | ~5 |
| `tools/code_execution_tool.py` | 群聊模式工具白名单移除 terminal | ~5 |
| `tools/code_execution_tool.py` | 环境变量净化增强（群聊隐私） | ~10 |
| **沙箱增量总计** | **3 个文件** | **~30 行** |

---


## Related

* [[Group Isolation Design]]
* [[Multi Agent Isolation]]
* [[Plugin Mechanism]]
* [[Hermes Distributed Architecture]]

## Tags

#sandbox #security #docker #isolation
