# Group Isolation & Sandbox Security Design

> **分析对象**：[Hermes Agent](../hermes-agent/) — 开源多通道 AI Agent 运行时平台
> **分析日期**：2026-04-14
> **目标**：为群聊场景实现按 chat group 隔离聊天记录、记忆、技能，并增强沙箱安全。

---

## Part 1: 现状分析

### 1.1 当前隔离现状

| 资源 | 隔离方式 | 群聊安全 |
|------|---------|---------|
| 对话历史 (session) | `platform:chat_type:chat_id:thread_id:user_id` | 按 user_id 隔离 ✓ |
| 内置记忆 (MEMORY.md/USER.md) | 按 profile 隔离，profile 内所有 session 共享 | **泄露 ✗** |
| 外部 Provider (Honcho/Hindsight) | 部分隔离（Honcho 有 user_id peer，Hindsight 共享 bank） | **部分泄露** |
| 技能 (skills/) | 按 profile 隔离，profile 内所有 session 共享 | **无隔离** |
| 定时任务 (cron/) | `skip_memory=True`，完全无状态 | **不适用** |
| 沙箱 (execute_code) | 进程隔离或 Docker 容器 | **按 profile 隔离** |

### 1.2 核心问题

1. **内置记忆泄露**：所有 session 共享同一份 MEMORY.md/USER.md，用户 A 写入的信息用户 B 可见
2. **技能无隔离**：所有用户共享同一套技能，任何用户都可修改
3. **外部 Provider 不完整隔离**：Hindsight 全局 memory bank，Honcho 显式配置 peerName 时跨用户共享
4. **沙箱按 profile 隔离**：所有群组共享同一个 Docker 容器/执行环境

### 1.3 设计目标

```
                    共享层 (管理员维护)              私有层 (群组/用户可修改)
                    ─────────────────              ─────────────────────
记忆            全局 MEMORY.md                  群组专属记忆 (读写)
用户画像        全局 USER.md                    群组用户画像 (读写)
技能            全局 skills (只读)               群组专属 skills (读写)
对话历史        不共享                        按 session 隔离 (已有)
配置            全局 config.yaml (API keys 等)    可选 per-group 覆盖
定时任务        全局 cron (不涉及用户数据)        群组专属 cron
沙箱            按群组隔离执行环境              按群组隔离文件系统
```

### 1.4 两种隔离架构对比

本设计提供两种方案，适用于不同场景：

| | 方案 A: 单进程 + ContextVar | 方案 B: per-group 进程 + Router |
|---|---|---|
| 隔离强度 | 逻辑隔离 | **进程级隔离**（最强） |
| 改动量 | 小（~9 文件, ~180 行） | 中（~5 文件, ~250 行） |
| 资源开销 | 低 | 每个 Worker 一个进程 |
| 故障隔离 | 一个 crash 影响全部 | **互不影响** |
| HERMES_HOME | 需要 ContextVar hack | **天然隔离**，零改动 |
| 并发安全 | 依赖 asyncio ContextVar | **天然安全**，进程隔离 |
| 适用场景 | 群组数量少、资源有限 | 群组数量多、隔离要求高 |
| 详细设计 | Part 2 | Part 2B |

---

## Part 2: 方案 A — 基于 Profile 复用的单进程隔离

### 2.1 核心思路

复用现有 Profile 机制（`hermes_cli/profiles.py`）的目录结构和管理工具，通过在 AIAgent 级别传入 `hermes_home_override` 参数实现**单进程内动态切换**，避免进程级 profile 切换的限制。

**为什么复用 Profile 而不是自建 Namespace**：

- Profile 已有完整隔离的目录结构（memories/skills/sessions/cron）
- Profile 管理工具开箱即用（create/delete/export/clone）
- 外部 Provider 配置天然隔离（各自从 HERMES_HOME 下读取 honcho.json 等）
- 改动量从 ~11 文件降到 ~6 文件，~100 行代码

### 2.2 目录结构

```
~/.hermes/
├── config.yaml                     ← 全局配置（API keys、model 等）
├── .env                            ← 全局密钥（始终从这里读取）
├── memories/
│   ├── MEMORY.md                   ← 全局共享记忆（只读）
│   └── USER.md                     ← 全局用户画像（只读）
├── skills/                          ← 全局共享技能（只读，symlink 或 copy）
│
├── profiles/                        ← 群组 profile 自动创建在这里
│   ├── grp-telegram--1001234567/
│   │   ├── memories/
│   │   │   ├── MEMORY.md           ← 群组专属记忆（agent 可读写）
│   │   │   └── USER.md             ← 群组用户画像（agent 可读写）
│   │   ├── skills/                  ← 群组专属技能（agent 可读写）
│   │   ├── sessions/               ← 群组会话历史（隔离）
│   │   ├── cron/                   ← 群组定时任务（隔离）
│   │   ├── state.db                ← 群组 SQLite（隔离）
│   │   └── honcho.json            ← 群组 Honcho 配置（隔离）
│   │
│   └── grp-discord-1234567890/
│       └── ...
```

### 2.3 各组件改造

#### 2.3.1 MemoryStore — 支持 hermes_home_override

**文件**: `tools/memory_tool.py`

```python
class MemoryStore:
    def __init__(self, hermes_home_override: str = None,
                 memory_char_limit=2200, user_char_limit=1375):
        self._root = Path(hermes_home_override) if hermes_home_override else get_hermes_home()

    def _path_for(self, target: str) -> Path:
        mem_dir = self._root / "memories"
        if target == "user":
            return mem_dir / "USER.md"
        return mem_dir / "MEMORY.md"

    def load_from_disk(self):
        # 加载群组专属记忆
        # 同时加载全局记忆作为只读基础（用于 system prompt）
        ...
```

**双层记忆加载逻辑**：

```python
# run_agent.py — _build_system_prompt 中
# 1. 全局共享记忆（只读，注入到系统提示词）
global_memory = self._load_global_memory()

# 2. 群组专属记忆（可读写，注入到系统提示词）
group_memory = self._memory_store.format_for_system_prompt("memory")

# system prompt = ... + global_memory + group_memory
```

#### 2.3.2 AIAgent — 透传 override 参数

**文件**: `run_agent.py`

```python
class AIAgent:
    def __init__(self, ..., hermes_home_override: str = None, ...):
        self._hermes_home_override = hermes_home_override

        # MemoryStore 使用 override 路径
        self._memory_store = MemoryStore(
            hermes_home_override=hermes_home_override,
            memory_char_limit=...,
            user_char_limit=...,
        )
```

MemoryManager 也需要传递 override：

```python
if self._memory_manager:
    _init_kwargs["hermes_home"] = (
        self._hermes_home_override or str(get_hermes_home())
    )
    self._memory_manager.initialize_all(**_init_kwargs)
```

#### 2.3.3 技能加载 — 共享只读 + 私有可写

**文件**: `agent/skill_utils.py`, `agent/prompt_builder.py`

```python
# agent/skill_utils.py
def get_skills_dir(hermes_home_override: str = None) -> Path:
    root = Path(hermes_home_override) if hermes_home_override else get_hermes_home()
    return root / "skills"

def get_shared_skills_dir() -> Path:
    """全局共享技能目录（始终可读）。"""
    return get_hermes_home() / "skills"
```

```python
# agent/prompt_builder.py — build_skills_system_prompt
# 扫描两个目录：
# 1. 全局 shared skills（只读，标记为 "shared"）
# 2. 群组 skills（可读写，标记为 "group"）
```

**写权限检查**（`tools/skill_tools.py`）：

```python
def skill_manage(action, name, ...):
    if action in ("patch", "create"):
        target_dir = resolve_skill_dir(name)
        if target_dir == get_shared_skills_dir():
            return error("Cannot modify shared skills. "
                         "Create a group-specific version instead.")
```

#### 2.3.4 Gateway — 自动创建群组 Profile

**文件**: `gateway/run.py`

```python
def _resolve_group_profile(source: SessionSource) -> Optional[str]:
    """从 session source 推导群组 profile 路径。DM 不隔离。"""
    if source.chat_type in ("dm",):
        return None
    if source.chat_id:
        name = f"grp-{source.platform.value}-{source.chat_id}"
        return str(get_profile_dir(name))
    return None

def _ensure_group_profile(source: SessionSource) -> Optional[str]:
    """按需创建群组 profile（首次消息时触发）。"""
    profile_path = _resolve_group_profile(source)
    if not profile_path:
        return None

    profile_dir = Path(profile_path)
    if not profile_dir.exists():
        profile_dir.mkdir(parents=True)
        for d in _PROFILE_DIRS:
            (profile_dir / d).mkdir(exist_ok=True)
        # 可选：从全局 skills/ symlink 共享技能
        logger.info("Auto-created group profile: %s", profile_dir.name)

    return profile_path
```

`_run_agent()` 中传入：

```python
namespace = _ensure_group_profile(source)
agent = AIAgent(
    ...,
    hermes_home_override=namespace,
)
```

Agent cache 按 `(session_key, namespace)` 键缓存。

#### 2.3.5 配置继承

群组 profile 的 `config.yaml` 只需包含需要覆盖的字段，其余从全局继承：

```python
def _load_group_config(group_profile_dir: str, global_config: dict) -> dict:
    group_config_path = Path(group_profile_dir) / "config.yaml"
    group_config = {}
    if group_config_path.exists():
        group_config = yaml.safe_load(group_config_path.read_text()) or {}
    return deep_merge(global_config, group_config)
```

API keys 始终从全局 `~/.hermes/.env` 读取，群组 profile 不复制。

#### 2.3.6 外部 Provider — 零改造

Honcho 和 Hindsight 的配置从 `hermes_home` 参数解析：

- Honcho 读 `$HERMES_HOME/honcho.json`
- Hindsight 读 `$HERMES_HOME/hindsight/config.json`

当 `MemoryManager.initialize_all()` 传入群组 profile 路径时，Provider 自动加载群组专属配置，天然隔离。无需修改 provider 代码。

#### 2.3.7 Cron 支持

**文件**: `cron/scheduler.py`, `tools/cronjob_tools.py`

Cron job 定义增加 `namespace` 字段：

```yaml
- name: "群组日报"
  schedule: "0 9 * * *"
  namespace: "grp-telegram--1001234567"
  prompt: "生成今日群组活跃度报告"
  deliver: "telegram:-1001234567"
```

`run_job()` 创建 agent 时：

```python
if job.get("namespace"):
    agent = AIAgent(..., hermes_home_override=profile_path, ...)
else:
    agent = AIAgent(..., skip_memory=True, ...)
```

### 2.4 共享技能管理

**初始创建**：群组 profile 创建时从全局 skills/ symlink 需要共享的技能。

**批量同步**：新增 CLI 命令用于将全局技能更新同步到所有群组：

```bash
hermes skills sync --to-groups research/arxiv  # 更新所有群组的 arxiv 技能
```

**写保护**：`skill_manage` 的 patch/create 操作检查目标路径。全局 skills/ 目录（或 symlink 指向它的路径）标记为只读。

### 2.5 改动文件清单

| 文件 | 改动 | 行数估计 |
|------|------|---------|
| `tools/memory_tool.py` | MemoryStore 加 `hermes_home_override`，双层加载 | ~30 |
| `run_agent.py` | AIAgent 加参数，透传给 MemoryStore/Manager | ~25 |
| `agent/skill_utils.py` | `get_skills_dir` 支持 override | ~5 |
| `agent/prompt_builder.py` | 技能索引支持共享+私有目录 | ~15 |
| `tools/skill_tools.py` | `skill_manage` 写权限检查 | ~15 |
| `gateway/run.py` | `_resolve_group_profile` + 自动创建 + 配置继承 | ~50 |
| `cron/scheduler.py` | 支持 `hermes_home_override` | ~15 |
| `hermes_cli/profiles.py` | `_PROFILE_DIRS` 加 `skills`（已有），可选过滤 | ~5 |
| `hermes_cli/commands.py` | 可选：新增 `skills sync` 命令 | ~20 |
| **总计** | **9 个文件** | **~180 行** |

### 2.6 向后兼容

- 无 `hermes_home_override` 时行为完全不变（DM、CLI、默认模式）
- 全局 `MEMORY.md` / `USER.md` 继续作为共享基础
- 外部 Provider 不配置 namespace 时行为不变
- 已有 cron job 无 namespace 字段时保持 `skip_memory=True`
- Profile 管理命令（list/delete/export/clone）直接复用

### 2.7 局限性

| 局限 | 说明 | 缓解 |
|------|------|------|
| ContextVar 并发安全 | asyncio 协程安全，但线程不安全 | Gateway 是纯 async，可接受 |
| 磁盘占用 | 每个群组一个 profile 目录 | 大部分是空目录，实际占用小 |
| 共享技能同步 | 全局 skill 更新后需手动同步 | 提供 `hermes skills sync` 命令 |
| Profile 列表混杂 | `hermes profile list` 会混入群组 profile | 加 `--type group` 过滤 |
| 全局记忆只读 | 群组 agent 不能修改全局 MEMORY.md | 设计意图如此，避免互相污染 |
| 故障隔离不足 | 一个群组的 agent crash 可能影响同进程其他群组 | 需要异常捕获和 agent 隔离 |

### 2.8 ContextVar 改造（单进程并发关键）

`get_hermes_home()` 当前读取进程级环境变量，单进程内多协程共享：

```python
# hermes_constants.py — 当前实现
def get_hermes_home() -> Path:
    return Path(os.getenv("HERMES_HOME", Path.home() / ".hermes"))
```

需要改为协程安全版本：

```python
import contextvars

_hermes_home_ctx: contextvars.ContextVar[Optional[Path]] = contextvars.ContextVar(
    "hermes_home", default=None
)

def get_hermes_home() -> Path:
    ctx_val = _hermes_home_ctx.get()
    if ctx_val is not None:
        return ctx_val
    return Path(os.getenv("HERMES_HOME", Path.home() / ".hermes"))

def set_hermes_home(path: Path):
    _hermes_home_ctx.set(path)

def reset_hermes_home(token):
    _hermes_home_ctx.reset(token)
```

Gateway session 创建时设置：

```python
# gateway/run.py — _run_agent 中
token = set_hermes_home(group_profile_path)
try:
    agent = AIAgent(...)
    await agent.run(...)
finally:
    reset_hermes_home(token)
```

---

## Part 2B: 方案 B — per-group 进程 + Router

### 2B.1 核心思路

每个群组一个独立 Worker 进程，各进程拥有独立的 `HERMES_HOME` 环境变量，天然实现进程级隔离。一个轻量 Router 进程负责接收飞书消息并按 `chat_id` 分发到对应 Worker。

**优势**：
- `HERMES_HOME` 是进程级环境变量，每个 Worker 天然隔离，**不需要 ContextVar**
- Worker crash 不影响其他群组
- 所有依赖 `get_hermes_home()` 的组件（MemoryStore、skills、sandbox、providers）零改造
- 故障隔离、资源隔离最强

**当前 Gateway 的消息流**：

```
FeishuAdapter → BaseAdapter.handle_message() → GatewayRunner._handle_message() → AIAgent
                     ↑ 紧耦合：同一个进程内收消息、跑 agent、回消息
```

**目标架构**：

```
FeishuAdapter → Router（序列化+分发） → IPC → Worker 进程（独立 HERMES_HOME）
                     ↑ 收发分离                      ↑ 完全隔离的 agent 执行
```

### 2B.2 架构图

```
                     ┌──────────────┐
                     │   飞书 Bot    │ (同一个 app_id / app_secret)
                     └──────┬───────┘
                            │ WebSocket / Webhook
                     ┌──────▼───────┐
                     │  Router 进程  │  只做消息收发+路由，不跑 agent
                     │  (Gateway)   │
                     └──┬───┬───┬───┘
                        │   │   │     IPC: Unix Socket / stdin+stdout
               ┌────────▼┐ │ ┌▼────────┐
               │ Worker A │ │ │Worker B │ ... 每个 group 一个
               │ HERMES_  │ │ │HERMES_  │
               │ HOME=    │ │ │HOME=    │
               │ group-a  │ │ │ group-b │
               └──────────┘ │ └─────────┘
                            │
                     ┌──────▼───────┐
                     │ Worker N     │
                     │ HERMES_HOME= │
                     │ group-n      │
                     └──────────────┘
```

### 2B.3 各组件改造

#### 2B.3.1 Router 进程（新增）

**文件**: `gateway/router.py`（新增，~150 行）

```python
class GroupRouter:
    """按 chat_id 将消息分发到独立 Worker 进程。"""

    def __init__(self):
        self.workers: Dict[str, WorkerClient] = {}  # chat_id → worker
        self._lock = asyncio.Lock()

    async def route(self, event: MessageEvent) -> Optional[str]:
        chat_id = event.source.chat_id

        if chat_id not in self.workers:
            await self._spawn_worker(chat_id)

        return await self.workers[chat_id].send_and_wait(event)

    async def _spawn_worker(self, chat_id: str):
        """启动群组 Worker 进程。"""
        profile_path = self._ensure_group_profile(chat_id)
        cmd = [
            sys.executable, "-m", "hermes_cli",
            "gateway", "--mode", "worker",
            "--group-id", chat_id,
            "--ipc", "stdio",  # 或 unix:/tmp/hermes-worker-{chat_id}.sock
        ]
        env = {**os.environ, "HERMES_HOME": str(profile_path)}
        proc = await asyncio.create_subprocess_exec(
            *cmd, stdin=asyncio.subprocess.PIPE,
            stdout=asyncio.subprocess.PIPE, stderr=asyncio.subprocess.PIPE,
            env=env,
        )
        self.workers[chat_id] = WorkerClient(proc)

    async def _ensure_group_profile(self, chat_id: str) -> Path:
        """按需创建群组 profile 目录。"""
        profile_dir = Path(f"~/.hermes/profiles/grp-feishu-{chat_id}").expanduser()
        if not profile_dir.exists():
            create_profile(profile_dir.name, clone_from=None)
        return profile_dir
```

**Worker 生命周期管理**：

| 事件 | 行为 |
|------|------|
| 首次收到某 group 消息 | 自动 `spawn_worker()` |
| Worker 进程 crash | 检测 stderr EOF，自动重启 |
| 长时间无消息 | 可选：idle timeout 后回收进程，下次消息重新启动 |
| Gateway 关闭 | 向所有 Worker 发送 shutdown 信号，等待退出 |

#### 2B.3.2 FeishuAdapter — 收发分离

**文件**: `gateway/platforms/feishu.py`

当前 `FeishuAdapter` 内部直接调用 `self._message_handler(event)` 跑 agent。需要拆分为两种运行模式：

```python
class FeishuAdapter(BasePlatformAdapter):
    def __init__(self, config: PlatformConfig, mode: str = "standalone"):
        super().__init__(config, Platform.FEISHU)
        self._mode = mode  # "standalone" | "router"

    async def connect(self):
        if self._mode == "router":
            # 只建立飞书连接，消息通过 router 分发
            self._event_handler = self._create_router_event_handler()
        else:
            # 原有逻辑：消息直接走 _handle_message → AIAgent
            self._event_handler = self._create_standalone_event_handler()
```

**Router 模式**下，adapter 只负责：
1. 接收飞书消息 → 序列化为 `MessageEvent`
2. 将 `MessageEvent` 传给 `GroupRouter.route()`
3. 收到 Worker 回复后调 `self.send_text()` 发回飞书

**不影响 standalone 模式**：非飞书平台和 DM 场景继续走原有 `_handle_message()` 路径。

#### 2B.3.3 Worker 进程

Worker 不直接连接飞书，通过 IPC 接收序列化消息，执行后通过 IPC 返回结果。

```python
# gateway/worker.py（新增，~100 行）
class GatewayWorker:
    """独立进程，处理单个群组的消息。"""

    def __init__(self, group_id: str, ipc_mode: str = "stdio"):
        self.group_id = group_id
        self._ipc = self._create_ipc(ipc_mode)

    async def run(self):
        """主循环：从 IPC 读消息 → 跑 agent → 写回结果。"""
        while True:
            event_data = await self._ipc.read()
            event = self._deserialize(event_data)

            # 复用现有 GatewayRunner._handle_message 的 agent 逻辑
            response = await self._process_with_agent(event)

            await self._ipc.write(self._serialize(response))

    async def _process_with_agent(self, event: MessageEvent):
        """调用 AIAgent 处理消息。复用 gateway/run.py 中的逻辑。"""
        # 此进程的 HERMES_HOME 已在启动时设置
        agent = AIAgent(...)  # 使用进程的 HERMES_HOME，天然隔离
        return await agent.run_conversation(...)
```

#### 2B.3.4 IPC 协议

推荐使用 Unix Socket（比 stdin/stdout 更灵活，支持多路复用和 Worker 重连）：

```
协议: JSONL (每行一个 JSON 消息)

Router → Worker:
  {"type": "message", "event": {MessageEvent 序列化}}

Worker → Router:
  {"type": "response", "text": "回复文本", "files": [...]}
  {"type": "stream", "token": "增量文本"}
  {"type": "error", "message": "错误信息"}
  {"type": "status", "state": "thinking|tool_call|done"}
```

#### 2B.3.5 命令行入口

`hermes gateway` 新增 `--mode` 参数：

```bash
# Router 模式（推荐生产使用）
hermes gateway --mode router --platform feishu

# Worker 模式（通常由 Router 自动启动，也可手动调试）
hermes gateway --mode worker --group-id "oc_xxxxx" --ipc unix:/tmp/hermes-worker.sock

# Standalone 模式（默认，当前行为不变）
hermes gateway
```

**文件**: `cli.py` — 解析 `--mode` 参数并分发到对应入口。

### 2B.4 记忆/技能/沙箱 — 零改造

由于每个 Worker 是独立进程且 `HERMES_HOME` 独立，以下组件**不需要任何代码改动**：

| 组件 | 为什么不需要改 |
|------|--------------|
| `hermes_constants.get_hermes_home()` | 读进程环境变量，Worker 天然隔离 |
| `tools/memory_tool.py` | `get_memory_dir()` → `get_hermes_home()` → Worker 的 HOME |
| `agent/skill_utils.py` | skills 目录随 HERMES_HOME 变化 |
| `tools/environments/docker.py` | sandbox 目录随 HERMES_HOME 变化 |
| `plugins/memory/honcho/` | 读 `$HERMES_HOME/honcho.json`，Worker 天然隔离 |
| `plugins/memory/hindsight/` | 配置路径随 HERMES_HOME 变化 |
| `cron/scheduler.py` | cron 文件随 HERMES_HOME 变化 |

**共享技能实现**（与方案 A 相同思路）：

群组 profile 创建时，将全局 skills 作为只读基础：

```python
def create_group_profile(chat_id: str):
    profile_dir = get_profile_dir(f"grp-feishu-{chat_id}")
    create_profile(profile_dir.name)
    # 全局 skills 作为只读层
    shared_skills = get_hermes_home() / "skills"
    group_skills = profile_dir / "skills"
    if shared_skills.exists():
        # 方式 1: symlink（轻量，但不能在 group 中创建同名 skill）
        # 方式 2: 初始复制（独立，但更新需手动同步）
        shutil.copytree(shared_skills, group_skills / "_shared", dirs_exist_ok=True)
```

### 2B.5 共享飞书连接的问题

飞书 SDK 是否支持同 `app_id` 多进程共享连接？

| 方案 | 说明 | 推荐 |
|------|------|------|
| **Router 收发，Worker 不连飞书** | Router 持有唯一飞书 WebSocket/Webhook，Worker 通过 IPC 回传回复 | 推荐 |
| 多进程各自连飞书 | 每个 Worker 独立建 WebSocket，飞书服务端可能限制同 token 连接数 | 不推荐 |
| 共享 WebSocket fd | 通过 Unix Domain Socket 传递 fd，复杂且飞书 SDK 不一定支持 | 不推荐 |

**推荐方案**：Worker 完全不接触飞书 SDK。所有消息收发由 Router 负责。

### 2B.6 改动文件清单

| 文件 | 改动 | 行数估计 |
|------|------|---------|
| `gateway/router.py` | **新增** — GroupRouter + WorkerClient + IPC | ~150 |
| `gateway/worker.py` | **新增** — Worker 主循环 + 序列化/反序列化 | ~100 |
| `gateway/platforms/feishu.py` | 支持 router 模式（收发分离） | ~40 |
| `gateway/run.py` | GatewayRunner 支持 `--mode router`，启动 Router | ~30 |
| `cli.py` | `hermes gateway --mode` 参数解析 | ~15 |
| `hermes_cli/profiles.py` | 无改动（`create_profile` 直接复用） | 0 |
| `tools/memory_tool.py` | 无改动 | 0 |
| `run_agent.py` | 无改动 | 0 |
| `tools/environments/docker.py` | 无改动 | 0 |
| **总计** | **4 个新增/修改文件** | **~335 行** |

### 2B.7 向后兼容

- `hermes gateway`（无 `--mode`）走原有 standalone 模式，行为不变
- 非 feishu 平台不受影响
- DM 场景不受影响（Router 检测 chat_type == "dm" 时走原有单进程路径）
- Worker 进程的 HERMES_HOME 通过环境变量设置，所有现有代码路径自动兼容

### 2B.8 局限性

| 局限 | 说明 | 缓解 |
|------|------|------|
| 进程数 | 每个 group 一个进程，group 多时内存开销大 | idle timeout 回收 + 进程池 |
| IPC 延迟 | 多一跳序列化/反序列化 | Unix Socket 延迟 < 1ms，可忽略 |
| 流式输出 | IPC 需支持 streaming token 传输 | IPC 协议定义 `stream` 类型 |
| Worker crash 恢复 | 需要检测 crash 并重启 | Router 监听 stderr EOF，自动 respawn |
| 横向扩展 | Router 是单点 | 后续可替换为消息队列（Redis/nats） |

---

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

## Part 4: 实施路线

### 选型建议

| 场景 | 推荐方案 | 理由 |
|------|---------|------|
| 群组数 < 10，资源有限 | 方案 A（单进程 + ContextVar） | 改动最小，180 行 |
| 群组数 10~100，有隔离要求 | 方案 B（per-group 进程） | 进程级隔离，故障不扩散 |
| 群组数 > 100，需要弹性扩展 | 方案 B + 消息队列 | Router 替换为 Redis/nats 分布式路由 |

### Route A: 方案 A 实施路径

#### Phase 1: 核心隔离（~180 行，9 个文件）

目标：群组记忆和技能隔离。

1. `hermes_constants.py` — `get_hermes_home()` 加 ContextVar 支持
2. `tools/memory_tool.py` — MemoryStore 加 `hermes_home_override` + 双层加载
3. `run_agent.py` — AIAgent 透传 override 参数
4. `agent/skill_utils.py` — get_skills_dir 支持 override
5. `agent/prompt_builder.py` — 技能索引双目录
6. `tools/skill_tools.py` — 写权限检查
7. `gateway/run.py` — 自动创建 + 配置继承 + ContextVar set/reset

#### Phase 2: Provider 隔离（零代码改动）

目标：外部 Provider 天然隔离（依赖 Phase 1 的 override 透传）。

验证 Honcho 和 Hindsight 在传入群组 profile 路径时的行为。

#### Phase 3: 沙箱隔离（~30 行，3 个文件）

目标：每个群组独立 Docker 容器和 execute_code 环境。

1. `tools/environments/docker.py` — sandbox 目录按 override 路径
2. `tools/terminal_tool.py` — 透传 override 到环境创建
3. `tools/code_execution_tool.py` — 群聊模式白名单调整 + 环境变量净化

#### Phase 4: Cron + CLI（~35 行，3 个文件）

1. `cron/scheduler.py` — 支持 hermes_home_override
2. `tools/cronjob_tools.py` — cron job 定义增加 namespace 字段
3. `hermes_cli/commands.py` — 可选：`skills sync` 命令

### Route B: 方案 B 实施路径

#### Phase 1: Router + Worker 框架（~250 行，4 个文件）

目标：消息收发分离，per-group 进程隔离。

1. `gateway/router.py` — **新增** — GroupRouter + WorkerClient + IPC 协议
2. `gateway/worker.py` — **新增** — Worker 主循环 + 消息序列化
3. `gateway/platforms/feishu.py` — 支持 router 模式
4. `cli.py` — `hermes gateway --mode` 参数

#### Phase 2: Worker 生命周期管理（~50 行，1 个文件）

目标：crash 恢复、idle 回收、优雅关闭。

1. `gateway/router.py` — Worker 健康检查 + 自动重启 + 超时回收

#### Phase 3: 共享技能（~20 行，1 个文件）

目标：全局 skills 作为只读基础注入群组 profile。

1. `hermes_cli/profiles.py` — `create_group_profile()` 增加 shared skills 初始化
2. `hermes_cli/commands.py` — 可选：`skills sync` 命令

#### Phase 4: 沙箱增强（群聊安全，Part 3 内容）

1. `tools/code_execution_tool.py` — 群聊模式工具白名单 + 环境变量净化
2. `tools/environments/docker.py` — 群聊默认 `--network=none`

> 注意：方案 B 的沙箱隔离是天然实现的（Worker 进程 HERMES_HOME 独立），不需要像方案 A 那样透传 `hermes_home_override`。Phase 4 仅是群聊场景下的安全加固。
