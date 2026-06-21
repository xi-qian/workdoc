# 多租户 Host-direct 隔离架构设计

**日期**: 2026-06-16
**状态**: Design (pending implementation plan)
**取代**: ADR-002, ADR-005, ADR-008, ADR-012, ADR-019 (parts), ADR-004 of `docs/runtime-rework/ADR.md`
**保留**: ADR-001, ADR-003, ADR-006, ADR-007, ADR-009, ADR-010, ADR-011, ADR-013, ADR-015, ADR-016, ADR-017, ADR-018, ADR-020

## 背景

现有 `docs/runtime-rework/` 计划的目标是“每个 agent 服务一个 Docker container，并在每个 container 内为每个 group 使用 Linux users”。本设计反转该方向：主要部署形态是一个 NanoClaw Docker image（或一个等价 Kubernetes pod），其中包含控制平面、helper 和运行进程。该部署内部的 Linux users 成为隔离单元；Docker 是打包和运维边界，不是每个 agent 或 group 的安全边界。

这一转向来自三个判断：

1. 在 Linux user 分离之上，Docker 不再额外提供凭据隔离。进程内存隔离、`/proc/<pid>/environ` 限制和文件权限已经让不同 UID 的进程处于不同安全域。Docker 的剩余价值是运维性质的（image 可移植性、cgroups、defence in depth），而不是安全性。
2. 部署便利性很重要：一个 NanoClaw 实例支持多个 tenants 和多个 agents，比 N 个 Docker services 更容易运维。
3. 真正需要保护的是渠道凭据（Feishu、Slack 等）和 tenant runtime data，而不是 LLM 凭据。当前部署把 agents 路由到内部 LLM gateway，使用内部 endpoint 和内部 credential；该 credential 不被公共 provider API 接受，在内部网络之外也没有用处，因此 LLM credential 的威胁模型更低。

## 目标

- 一个 NanoClaw 控制平面进程支持多个 tenants、每个 tenant 多个 agents、每个 agent 多个 groups。
- 隔离单元：每个 `(tenant, agent, group)` 元组一个 Linux user。Usernames 使用 `ncg-` 前缀加稳定 tuple hash，避免 sanitisation collisions。
- Per-(tenant, agent) external identity：每个 agent 可以拥有自己的 Feishu app、Slack bot 等。
- 形如 `/<tenant>/<agent>/<channel>/event` 的 Webhook URLs 将 inbound events 路由到正确的 channel instance。
- 干净替换现有 `docker-per-group` runtime。没有 fallback mode。
- 主要部署目标：每个 NanoClaw 实例一个 Docker image。Kubernetes 使用一个等价 pod。Docker/Kubernetes 是打包边界；per-group 隔离仍是 image 内部映射的 `ncg-*` Linux user。

## 非目标

- 默认不提供 per-group kernel namespace、per-group network namespace。
- 不防御 kernel exploits 或 container escapes（与 Docker 威胁面相同，因为两者都共享宿主 kernel）。
- 不在过渡期间兼容运行现有 `groups/<name>/` layout。一次性迁移脚本负责切换。
- NanoClaw 内部不做 multi-tenant billing 或 quota enforcement（由内部 LLM gateway 或外部 channel platforms 处理）。
- Approach A（独立 supervisor process）列在 “Future extensions” 下，但不属于初始实现。

## 架构概览

```text
NanoClaw deployment (single Docker container OR equivalent pod)
├── Control plane process (uid=nanoclaw-svc, no capabilities)
│   ├── HTTP webhook server: POST /<tenant>/<agent>/<channel>/event
│   ├── channels: per-(tenant, agent) Feishu / Slack / Telegram / ... clients
│   ├── router, scheduler, sender/trigger policy
│   ├── tool workers: Feishu / approval / file / task APIs (host-side)
│   ├── tenant config loader
│   ├── run lifecycle manager: prepare, spawn, monitor, reap, kill, idle-reap
│   └── run spawner: invokes nc-setuid-helper
├── nc-setuid-helper (SUID root binary, ~500 lines C)
│   └── privileged operations: prepare, spawn, kill, cgroup setup
└── Run processes (uid=mapped ncg-* user for tenant/agent/group)
    └── agent-runner: Claude / OpenCode / mock provider adapter
```

### 权限模型

控制平面从不持有任何 Linux capability。所有特权操作都通过 `nc-setuid-helper`，这是一个安装在 `/usr/lib/nanoclaw/nc-setuid-helper` 的小型 SUID root binary（mode 4750，owner=root，group=nc-priv）。只有 `nanoclaw-svc` user 在 `nc-priv` group 中，因此只有控制平面可以调用 helper。

### 部署设置：Docker image

参考生产部署是单个 NanoClaw Docker image。运行一个包含控制平面、helper 和 run processes 的容器。容器配置如下：

- 从 `nanoclaw` image 启动为长期运行的 service container，不是每个 tenant、agent 或 group 一个 container。
- 使用能让 helper 完成窄职责的 privilege profile。默认运维形态是 `--privileged`；hardened profile 仍必须允许 SUID execution、`setuid/setgid`、signal mapped `ncg-*` processes、POSIX ACL changes 和 cgroup v2 writes。
- 不要启用 `no_new_privileges`；`/usr/lib/nanoclaw/nc-setuid-helper` 必须能以 SUID root 执行。
- 挂载支持 ACL 的持久数据，例如 `-v /srv/nanoclaw:/var/lib/nanoclaw`。底层 filesystem 必须支持 POSIX ACLs。
- 挂载可写 cgroup v2 view，或委派 `/sys/fs/cgroup/nanoclaw/`，让 helper 可以创建 per-run cgroups 并设置 memory/pid/cpu limits。
- 启用 container-local user/group management。Image 必须包含 `prepare` 用来创建 mapped `ncg-*` users 的 local NSS/user/group 机制。
- 当 operators 需要独立 backup 和 rotation policies 时，将 tenant repositories 和 auth storage 保持为独立 mounts。

基础 Docker run 形态：

```bash
docker run -d --name nanoclaw \
  --privileged \
  --security-opt no-new-privileges:false \
  --cgroupns=host \
  -v /srv/nanoclaw:/var/lib/nanoclaw \
  -v /sys/fs/cgroup:/sys/fs/cgroup:rw \
  -e NANOCLAW_DATA_DIR=/var/lib/nanoclaw \
  nanoclaw:<version>
```

Operator 只有在证明 helper 仍能执行 `prepare`、`spawn`、`kill`、`cgroup` 操作，且 isolation test suite 通过后，才可以用更收紧的 runtime profile 替换 `--privileged`。

在 Kubernetes 中，部署遵循相同形态：每个 NanoClaw 实例一个 pod，带等价 `securityContext`、persistent volume、POSIX ACL support 和 writable/delegated cgroup v2 subtree。Pod 仍只是打包形态；pod 内部的 Linux UID separation 仍是隔离边界。

Helper 暴露五个操作，每个操作都有严格参数校验：

```
nc-setuid-helper prepare --tenant=<t> --agent=<a> --group=<g> --runtime-dir=<dir>
nc-setuid-helper spawn  --uid=<u> --gid=<g> --cgroup=<path> --runtime-dir=<dir> -- <cmd...>
nc-setuid-helper kill   --pid=<p> --signal=<sig> --uid=<u> --runtime-dir=<dir> --cgroup=<path> --start-time=<ticks>
nc-setuid-helper cgroup --path=<path> --mem=<mb> --pids=<n> --cpu=<shares>
nc-setuid-helper status --pid=<p> --uid=<u> --runtime-dir=<dir> --cgroup=<path> --start-time=<ticks>
```

校验规则：

- `prepare` 校验 `(tenant, agent, group)`，推导抗碰撞 Linux username，缺失时创建 user/group，创建 runtime directory tree，应用 ACLs 和 modes，预创建 runtime DB files，并在 `/var/lib/nanoclaw/users.db` 中记录 mapping。
- `spawn` 的 `--uid` 和 `--gid` 必须匹配 `^ncg-`，并对应 helper 通过 `prepare` 创建的精确 `(tenant, agent, group)` mapping。
- `--runtime-dir` 必须位于 `/var/lib/nanoclaw/runtime/` 下，匹配记录的 mapping，并具有预期 owner/mode/ACLs。
- `kill` 和 `status` 必须匹配控制平面记录的 active-run identity：PID、`/proc/<pid>/stat` start time、expected real UID、runtime dir 和 cgroup path。Start time 或 cgroup 不匹配的 PID 会被视为 stale PID reuse，绝不会被 signal。
- `cgroup` paths 必须位于 `/sys/fs/cgroup/nanoclaw/` 下。

任何违规都会让 helper 以非零状态退出，且不执行操作。控制平面会看到失败并报告。

### 进程生命周期

| Operation | Owner | Mechanism |
|-----------|-------|-----------|
| Prepare user/runtime | control plane | `nc-setuid-helper prepare` 创建/修复 Linux user、group、runtime dirs、ACLs 和 runtime DB files |
| Spawn | control plane | `posix_spawn` → child execs `nc-setuid-helper spawn` → helper setuids、设置 cgroup、execs agent-runner |
| Liveness check | control plane | `stat("/proc/<pid>")` 加 helper `status`，校验 PID start time、expected UID、runtime dir 和 cgroup |
| Stop (SIGTERM/SIGKILL) | helper | `nc-setuid-helper kill --pid --signal --uid --runtime-dir --cgroup --start-time` |
| Cgroup / resource limits | helper | Spawn time 设置；后续通过 `cgroup` command 调整 |
| Zombie reaping | control plane | Run process 是控制平面直接子进程（helper 只桥接 exec）；正常 SIGCHLD + `waitpid` |
| Idle reap / timeout kill | control plane | 定时器扫描 active runs；超过阈值时调用 helper kill |
| Host-restart reconcile | control plane | DB-listed active PIDs 通过 `/proc/` 和 helper `status` 检查；缺失或 identity mismatch 的 PIDs 标记为 crashed/stale |

每条 active-run DB record 存储 `pid`、`/proc/<pid>/stat` start time ticks、expected uid、runtime dir、cgroup path、tenant、agent、group 和 run id。Helper 永远不会只从 PID 推断 ownership。

两条 Linux 规则决定了这个拆分：

- `waitpid` 忽略 UID：控制平面（uid=nanoclaw-svc）可以 reap 由 helper 改为 `ncg-...` UID 的子进程。
- `kill` 要求同 UID（或 privilege）：控制平面不能 signal `ncg-*` 进程，因此 kill 一律通过 helper。

这样可以把特权面限制到单个可审计 binary，同时让控制平面拥有所有 lifecycle policy。

## 用户模型

### 命名约定

概念 Linux user identity：

```
ncg-<tenant>-<agent>-<group>
```

实际 Linux username：

```
ncg-<tenant8>-<agent8>-<hash10>
```

规则：

- Tenant 和 agent IDs 在 tenant-config load time 转为小写并校验，只在 human-readable username prefix 中截断。
- `<hash10>` 在 sanitisation 之前从规范 tuple `(tenant, agent, group)` 推导，因此 `a_b`、`a-b`、`a/b` 和大小写变体不会折叠成同一个 Linux user。
- `/var/lib/nanoclaw/users.db` 存储从规范 tuple 到 Linux uid/gid/username 的权威 mapping。Helper 拒绝任何 username collision 或 tuple remap。
- Username 是实现细节；authorization 和 routing 始终使用规范 `(tenant, agent, group)` tuple。

每个 user 获得：

- 一个与 mapped `ncg-*` username 匹配的 primary group。
- Home directory 就是 runtime directory 本身（`/var/lib/nanoclaw/runtime/<t>/<a>/<g>/`）。没有单独的 `/home/ncg-*`。
- 没有 supplementary groups。`/opt/nanoclaw/agent-runner/` 下的 platform runner code 是 world-readable（mode 0755，owner=nanoclaw-svc），因此 run process 不需要特殊 group 也能读取。Tenant 和 agent skills 不存放在 world-readable paths。

### 控制平面对 runtime directories 的访问

控制平面（`uid=nanoclaw-svc`）需要读写每个 runtime directory 内的 IPC databases，同时不能破坏 `ncg-*` users 之间的隔离。机制是由 `nc-setuid-helper prepare` 安装的 per-runtime POSIX ACLs：

- 每个 runtime directory 都是 `owner=ncg-<mapped-user>`、`group=ncg-<mapped-user>`、`mode=0700`。
- ACL 授予 `nanoclaw-svc` 对 directory tree 的 rwx。
- Default ACL 授予 group user 和 `nanoclaw-svc` 对新建 directories 的 rwx，以及对新建 files 的 rw。
- Helper 预创建的 runtime DB files（`inbound.db`、`outbound.db`、`state.db`、`tools.db`）以及在 default ACL 下创建的 SQLite sidecars 可由 group user 和 `nanoclaw-svc` 共同 read/write；其他 `ncg-*` users 没有 ACL entry，也无法访问。

这样 `nanoclaw-svc` 无需通过 helper 提权即可透明 read/write 每个 runtime dir 的 IPC files，同时保持 `ncg-*` users 彼此隔离。刻意避免 shared runtime group，因为它要么会在文件由错误 owner 创建时失败，要么会把所有 run users 对所有 runtime directories 的访问权授给彼此。

### 生命周期

Users 和 runtime directories 在给定 `(tenant, agent, group)` tuple 的第一次 run 前，由 `nc-setuid-helper prepare` 懒创建，并且**不会自动删除**。Idle runs 会停止进程，但保留 user 和 runtime directory 在磁盘上，让下一条消息走 warm path。独立的 `nanoclaw-user-gc` admin command 可以清理已从 config 移除的 tenants 对应 users；该动作由 operator 驱动，不自动执行。

State file `/var/lib/nanoclaw/users.db`（SQLite，owned by root，mode 0600）跟踪 helper 创建过的每个 user，因此 helper 可以按历史记录校验 `spawn --uid` requests。

## 租户配置

### 来源

所有 tenant 和 agent configuration 都在启动时从 tenant repository 加载。路径通过 `NANOCLAW_TENANTS_DIR` 设置。Loader 校验 schemas，解析 skill references，并生成内存中的 `RegisteredTenant` / `RegisteredAgent` objects。不支持 ad-hoc `groups/<name>/` directory。

### 布局

```text
nanoclaw-tenants/
  tenants/
    <tenant>/
      tenant.json                      # tenant metadata, enabled flag
      skills/                          # tenant-level skills (read-only inputs)
        <skill>/
          SKILL.md
          manifest.json
      agents/
        <agent>/
          agent.json                   # provider, model, instructions, skill refs, limits
          instructions.md
          skills/                      # agent-local skills
          channels/                    # per-(tenant, agent) external identity config
            feishu.json                # mode (websocket/webhook), webhook settings, secret refs
            slack.json                 # (optional)
            telegram.json              # (optional)
```

### `agent.json` example

```json
{
  "id": "finance",
  "tenant": "acme",
  "name": "Finance Bot",
  "provider": "claude",
  "model": "claude-sonnet-4",
  "instructions": "./instructions.md",
  "skills": [
    "builtin:welcome",
    "tenant:acme-approval",
    "agent:finance-local"
  ],
  "channels": ["feishu"],
  "envRefs": ["llm:ANTHROPIC_API_KEY"],
  "limits": {
    "memoryMb": 1024,
    "pids": 256,
    "concurrentTasksPerGroup": 1
  }
}
```

### `channels/feishu.json` example

```json
{
  "mode": "websocket",
  "appId": "cli_acme_finance",
  "appSecretRef": "channel:FEISHU_APP_SECRET",
  "webhook": {
    "encryptKeyRef": "channel:FEISHU_WEBHOOK_ENCRYPT_KEY",
    "verificationTokenRef": "channel:FEISHU_WEBHOOK_VERIFICATION_TOKEN"
  }
}
```

实际 secret values 绝不进入 tenant repo。规范 auth root 是 `/var/lib/nanoclaw/auth/tenants/<tenant>/<agent>/`。Repo 只携带 typed references：

- `llm:<name>` 可以解析到 run environment，因为它指向内部 LLM gateway。
- `channel:<name>` 只能在控制平面内解析，绝不写入 runtime DBs、run envs、logs 或 skill bundles。
- 未知或无类型 refs 会导致 config validation 失败。

Channel secrets 位于 `/var/lib/nanoclaw/auth/tenants/<tenant>/<agent>/<channel>/credentials.json`（mode 0600，owner=nanoclaw-svc）。

## 渠道注册表和 webhook 路由

### 注册表 key

Channel registry 使用复合 key `(tenant_id, agent_id, channel_type)`。对于每个声明 channel 的 `agent.json`，loader 使用该 agent 的外部身份构造 channel instance，并在复合 key 下注册。Lookup 通过 `getChannel(tenantId, agentId, channelType)`；现有 singleton `getFeishuChannel()` accessor 被移除。

### 入站 webhook server

控制平面运行**一个** HTTP server。每个 channel instance 注册自己的 URL prefix：

```
POST /<tenant>/<agent>/feishu/event       → FeishuChannel(tenant, agent).handleWebhook
POST /<tenant>/<agent>/slack/event        → SlackChannel(tenant, agent).handleWebhook
GET  /<tenant>/<agent>/<channel>/verify   # challenge / verification responses
```

Feishu developer console 将 webhook URL 配置为 `https://nanoclaw.example.com/<tenant>/<agent>/feishu/event`。不同 `(tenant, agent)` 元组使用不同 URL。

对于使用 outbound connections 的 channels（Feishu WebSocket mode、Slack Socket Mode、Telegram long-poll），每个 channel instance 拥有自己的 connection。NanoClaw 根据 event 到达的 connection 识别源 tenant/agent，而不是根据 URL path。

### 渠道隔离

每个 `FeishuClient`（以及其他 channels 的等价实现）都以自己的 credentials 构造，拥有自己的 `Lark.Client` / WebSocket connection / event handler map，并且在一个进程内对多个 instances 状态安全。当前代码已经支持这一点；需要的改动只有：

1. `src/channels/feishu.ts:1026`：registry key 从 string literal 改为复合 `(tenant, agent, type)`。
2. `src/feishu/auth.ts:13-14`：credential file path 从 legacy `store/auth/feishu/credentials.json` 改为 `/var/lib/nanoclaw/auth/tenants/<tenant>/<agent>/feishu/credentials.json`。
3. `src/feishu/auth.ts:72-88`：移除 env-var overrides；webhook settings 来自 `channels/feishu.json`。
4. `src/index.ts:899` / `src/ipc.ts:242`：用 `getChannel(tenantId, agentId, 'feishu')` 替换 `getFeishuChannel()`。

另有架构改动：webhook HTTP server 从 per-`FeishuClient` 移到控制平面中的共享 server。

## 凭据模型

两级威胁模型：

### LLM 凭据（低风险）

Anthropic API credentials 指向 NanoClaw 的内部 LLM gateway，run process 只调用该内部 gateway endpoint。这些凭据不是公共 Anthropic credentials；它们只被内部服务接受，从内部网络外不可用。交付方式：spawn time env injection。

```
ANTHROPIC_BASE_URL=<internal-gateway-url>      # from tenant/agent config
ANTHROPIC_API_KEY=<internal-gateway-credential> # from /var/lib/nanoclaw/auth/tenants/<tenant>/<agent>/llm/credentials.json
```

只有 `llm:` refs 可以进入 run environment。不使用 credential proxy，不使用 scoped tokens，不使用 `SO_PEERCRED`。出于通用卫生要求，文件仍是 0600 / owned by `nanoclaw-svc`。

### 渠道凭据（高风险）

Feishu `app_secret`、Slack bot tokens、Telegram bot tokens、Discord bot tokens 是真实外部凭据。它们只存在于：

- `/var/lib/nanoclaw/auth/tenants/<tenant>/<agent>/<channel>/credentials.json`：0600，owner=`nanoclaw-svc`
- 加载后的控制平面进程内存

运行进程**永远**看不到 channel credentials。它们通过 tool IPC 请求 channel operations（写入 `tools.db`），控制平面 tool worker 对 channel client 执行操作，并把结果写回 `tools.db`。

这个模式继承自当前架构，新设计中保持不变。

### 运行时数据（高风险）

聊天历史、provider continuation、生成的 skills、下载的 files。由 Linux user ownership、0700 directory modes 和给 `nanoclaw-svc` 的 per-runtime ACLs 保护。跨 group、跨 agent、跨 tenant 读取会因 permission denied 失败。

## 运行时目录

### 宿主侧布局

```text
/var/lib/nanoclaw/                          # NANOCLAW_DATA_DIR (configurable)
  users.db                                  # state: users created by helper (root:root 0600)
  tenants -> /path/to/nanoclaw-tenants      # symlink, or direct path
  auth/
    tenants/
      <tenant>/<agent>/llm/credentials.json         # internal gateway credential, 0600, owner=nanoclaw-svc
      <tenant>/<agent>/<channel>/credentials.json   # external channel credential, 0600, owner=nanoclaw-svc
  runtime/
    <tenant>/<agent>/<group>/
      live/
        inbound.db
        outbound.db
        state.db
        tools.db
        files/
        downloads/
      runs/<runId>/                         # isolated task runs
        inbound.db
        outbound.db
        state.db
        tools.db
        files/
        downloads/
      skills/
        generated/                          # group-created skills (writable by group user only)
  logs/
    <tenant>/<agent>/<group>/
      live.log
      runs/<runId>.log
```

运行时目录所有权：

- Owner/group：mapped `ncg-*` user for `(tenant, agent, group)`
- Mode：0700 加 `nanoclaw-svc` 的 POSIX ACL
- Default ACL：授予 mapped `ncg-*` user 和 `nanoclaw-svc` 对 runtime DB files 和 SQLite sidecars 的 read/write access

控制平面通过 per-runtime ACLs 透明访问 IPC files（inbound、outbound、tools DBs）。理由见 User model 下的“控制平面对 runtime directories 的访问”。

### 运行进程视图

运行进程的 cwd 和 home 是 `/var/lib/nanoclaw/runtime/<t>/<a>/<g>/{live|runs/<runId>}`。它可以看到：

- 自己的 `inbound.db` / `outbound.db` / `state.db` / `tools.db`（read/write）
- 自己的 `skills/generated/`（read/write）
- 自己 runtime directory 下的 per-run read-only resolved skill bundle（`skills/resolved/<revision>/`）。该 bundle 只包含此 run 选中的 builtin、tenant 和 agent skills。
- `/opt/nanoclaw/agent-runner/`（read-only platform code）

它不能看到：

- 其他 tenants/agents/groups 的 runtime directories（由其他 users 拥有，mode 0700，并且 ACL 只授予 `nanoclaw-svc`）
- `auth/` 和 `users.db`（owned by nanoclaw-svc 或 root，mode 0600）
- Tenant repo source files 和其他 tenants 的 resolved skill bundles（只加载到控制平面内存；run process 只获得自己 runtime dir 下的 resolved skill bundle 和 manifest）

## 数据流和 IPC

DB-backed IPC 保持继承自 `docs/runtime-rework/03-db-backed-ipc.md` 的设计不变。只有路径前缀从 `/runtime/groups/<group>/` 改为 `/var/lib/nanoclaw/runtime/<tenant>/<agent>/<group>/`。

控制平面 host DB 按 `(tenant, agent)` 分区，例如 `/var/lib/nanoclaw/data/tenants/<tenant>/<agent>/messages.db`。每个 per-agent DB 内，channel-derived tables 和 cursors 的 identity 必须包含 `channel_type`：

- `chats`：`(channel_type, jid)`
- `messages`：`channel_type` 加 channel message identity
- `registered_groups`：`(channel_type, jid)`
- `router_state`：`last_timestamp[channel_type]` 和 `last_agent_timestamp[channel_type, chat_jid]`

这让 per-agent SQLite files 保持较小，同时防止一个 multi-channel agent 比较或覆盖无关 channel cursors。

| File | Writer | Reader |
|------|--------|--------|
| `inbound.db` | control plane | run process |
| `outbound.db` | run process | control plane |
| `state.db` | run process | run process (continuation, audit) |
| `tools.db` | run process (request) / control plane tool worker (result) | both |

Tool workers 从 request row 的 source identity 接收 `(tenant, agent, group, runId)`，然后通过 `getChannel(tenantId, agentId, channelType)` 查找 channel instance。没有全局 singleton lookup。

### 审计日志

每个 run 在执行过程中把自己的 API usage（model、tokens、latency、status）写入 `state.db` 或 `outbound.db`。控制平面按需聚合 per-(tenant, agent) view，供 status dashboard 使用。这样不需要 central proxy，也仍可做 per-tenant reporting。

## 决策记录变更

### 已反转

- **ADR-002**（保留 `docker-per-group` fallback）→ reversed。干净替换。
- **ADR-005**（每个 agent service 一个 Docker）→ reversed。新 ADR-005'：一个 Docker image 是部署单元；image 内映射的 Linux users 是隔离单元。
- **ADR-008**（supervisor 拥有 group process lifecycle）→ revised。控制平面拥有 lifecycle *policy*；特权 *mechanism*（setuid、signal、cgroup）封装在 `nc-setuid-helper`。控制平面不持有 capabilities。

### 已删除

- **ADR-004**（legacy file IPC compatibility）→ deleted。干净替换，无兼容层。
- **ADR-019**（runtime code 默认 immutable）→ narrowed。原则仍适用于 `/opt/nanoclaw/agent-runner/` 下的平台代码，但“没有 per-group writable runner source”的理由现在是结构性的（Linux user 无法写出自己的 runtime dir），而不是 policy。

### 新增

- **ADR-021**：每个 `(tenant, agent)` tuple 可以声明自己的外部 channel identity。Channel registry key 是 `(tenant_id, agent_id, channel_type)`。
- **ADR-022**：Inbound webhook URL `/<tenant>/<agent>/<channel>/event` 是 routing key。控制平面中的一个共享 HTTP server 按 path 分发。
- **ADR-023**：特权操作只能通过 `nc-setuid-helper`（SUID root）。控制平面不持有 Linux capabilities。
- **ADR-024**：两级 credential threat model。LLM credentials 是只被内部 LLM gateway 接受的 internal-gateway credentials，typed `llm:` refs 可以注入 run envs。Channel credentials 是真实外部凭据，typed `channel:` refs 仅限 host-side，run processes 使用 tool IPC。Runtime data 由 Linux user isolation 保护。
- **ADR-025**：Channel registry 使用 composite key。所有 `getFeishuChannel()` 风格 singleton accessors 都移除，改用 `getChannel(tenantId, agentId, channelType)`。

### 保持不变

ADR-001（tenant/skill boundary）、ADR-003（DB IPC）、ADR-006（isolated task uses same group user）、ADR-007（isolated task does not reuse live continuation）、ADR-009（provider differences behind `AgentProvider`）、ADR-010（OpenCode optional）、ADR-011（secrets host-side，由 ADR-024 sharpened）、ADR-012（no world-writable IPC）、ADR-013（registered ≠ active）、ADR-015（tenant skills read-only inputs）、ADR-016（generated skills group-local）、ADR-017（partitioned host DBs remain the control-plane source of truth）、ADR-018（host-side authz）、ADR-020（additional mount scope explicit）。

## 迁移

一次性脚本：`npm run migrate:to-multi-tenant -- --tenant <default-tenant-id> --source ./legacy-backup --target ./nanoclaw-tenants`

脚本：

1. 先以 `--dry-run` 运行，生成列出每项转换的 migration report。等待显式确认后才修改任何内容。
2. 写入前先把所有 source data 备份到 `legacy-backup-<timestamp>/`。
3. 转换 layout：

   ```text
   legacy                                          new
   ─────────────────────────────────────────────────────────────────────────
   groups/<name>/CLAUDE.md                  →     tenants/<t>/agents/<name>/instructions.md
                                                                                   + agent.json (provider=claude)
   store/auth/feishu/credentials.json       →     /var/lib/nanoclaw/auth/tenants/<t>/<name>/feishu/credentials.json
   data/sessions/<group>/                          → /var/lib/nanoclaw/runtime/<t>/<name>/<group>/live/
     (Claude SDK state repacked into state.db)
     agent-runner-src/                             ← dropped (platform code is shared in new model)
   data/ipc/<group>/                               → dropped (file IPC not supported; backed up only)
   data/nanoclaw.db, data/messages.db,
   store/messages.db                               → split into per-(tenant, agent) host DBs under
                                                    /var/lib/nanoclaw/data/tenants/<t>/<agent>/
                                                    (legacy default tenant=<default-tenant-id>, agent=<folder>)
                                                    and populate channel_type in chat/message/group/cursor keys
   ```

4. 通过 `npm run verify:migration` 验证结果：
   - 每个 legacy group 都有到新 `(tenant, agent, group)` tuple 的 1:1 mapping。
   - Source 和 target DBs 的 message counts 匹配。
   - `/var/lib/nanoclaw/auth/` 下每个 credentials file 都是 mode 0600，owner=nanoclaw-svc。
   - Tenant config loader 可以成功加载新 repo，且没有 diagnostics。

迁移设计为**不可逆**（没有 `docker-per-group` fallback）。Operators 必须在确认前验证 dry-run report。

## 测试

### 权限隔离（`npm run test:isolation`）

在已创建多个 `ncg-*` users 的 test host 内运行：

- `(a, x, y)` 的 mapped run user 不能读取或写入 `(a, x, z)` 的 `state.db`。
- `(a, x, y)` 的 mapped run user 不能读取 `auth/` 下任何文件。
- `(a, x, y)` 的 mapped run user 不能读取 `(b, x, y)` 的 runtime directory（cross-tenant）。
- `(a, x, y)` 的 mapped run user 不能读取另一个 tenant/agent 的 resolved skill bundle。
- `nanoclaw-svc` 和 mapped `ncg-*` user 都可以 read/write 由任一进程创建的 runtime DB files 和 SQLite sidecars。
- Run process environment 只包含 run config 和 typed `llm:` credentials，例如内部 LLM gateway 的 `ANTHROPIC_BASE_URL` / `ANTHROPIC_API_KEY`，不包含 `channel:` credentials，也不包含其他 tenant 的 data。

### Webhook 路由

- POST `/t1/a1/feishu/event` 只触发 `FeishuChannel(t1, a1)`。
- POST 到 `/t1/a1/...` 不影响 `FeishuChannel(t1, a2)` 或 `FeishuChannel(t2, a1)`。
- WebSocket-mode channel events 根据 connection identity 路由到正确的 `(tenant, agent)`。

### 渠道凭据隔离

- `(t1, a1, g1)` 的 run process 调用 feishu tool → tool worker 使用 `FeishuChannel(t1, a1)` credentials，绝不使用 `(t2, a1)` 或 `(t1, a2)`。
- 对 run process memory dump 做 grep（test-only）找不到 `app_secret` strings。

### 生命周期

- 控制平面 kill 一个 run → process 退出，audit row 写入，runtime directory 保留。
- Run process crash → 控制平面通过 SIGCHLD 和 `/proc/<pid>` absence 检测，并 reconcile DB state。
- Idle-reap timer 触发 → idle runs 通过 helper 被 kill。
- Host restart → 控制平面用 `/proc/` 加 helper `status` reconcile DB-listed active PIDs；缺失或 identity mismatch 的 PIDs 标记为 crashed/stale。
- PID reuse test：同 PID 但 `/proc/<pid>/stat` start time 或 cgroup 不同的进程，绝不会被视为旧 run，也绝不会被 helper `kill` signal。

### Setuid helper 测试

Helper binary 的 standalone C test suite：

- `prepare` 为新的 `(tenant, agent, group)` tuple 创建预期 Linux user/group mapping、runtime tree、ACLs 和 runtime DB files。
- `prepare` 对已有 tuple 幂等，并拒绝任何 tuple-to-username remap 或 username collision。
- 拒绝不匹配 `^ncg-` 的 `spawn --uid`。
- 拒绝 `users.db` 中不存在 users 的 `spawn --uid`。
- 拒绝不匹配记录 tuple/user mapping 或缺少预期 ACLs 的 `spawn --runtime-dir`。
- 拒绝 real UID 不是 expected mapped `ncg-*` user 的 `kill --pid`。
- 当 PID start time、runtime dir、cgroup 或 expected UID 不匹配 active-run record 时，拒绝 `kill` 和 `status`。
- 拒绝来自 `nanoclaw-svc` 以外任何 UID 的调用。
- 成功 `spawn` 后，被 exec 的进程已经丢弃所有 capabilities，并以请求的 UID/GID 运行。

## 未来扩展

### 方案 A: 独立 supervisor process

如果控制平面增长到需要在 NanoClaw 内部做 privilege separation 的程度，引入一个长期运行的 supervisor process，拥有 user lifecycle 和 run spawning。控制平面通过 Unix socket 与 supervisor 通信。这会：

- 进一步缩小 privilege surface（控制平面不再直接调用 helper）。
- 让 supervisor 成为唯一需要 `nc-priv` group membership 的进程。
- 增加一个 IPC boundary 和一个需要运维的长期进程。

不属于初始实现。当控制平面 binary 超过复杂度阈值，或 audit requirements 需要更清晰地区分 routing logic 与 privileged operations 时触发。

### 每次运行的 Unix socket 凭据 proxy

如果 NanoClaw 将来运行 untrusted tenants（例如 multi-customer SaaS），将 credential delivery 从 env injection 升级为 per-run Unix socket proxy：

- 每个 run 在 `/var/run/nanoclaw/cred-proxy.<runId>.sock` 获得自己的 Unix socket。
- Socket file access 只授予 `nanoclaw-svc` 和该 run 对应的 mapped `ncg-*` user。
- Kernel-enforced access control：只有该 run 可以连接。
- Anthropic SDK 配置 custom HTTP agent 指向该 socket。
- Proxy 在 server-side 注入 credentials。

当前内部部署不需要，但作为升级路径记录。

### Skill 热加载

基础 runtime 稳定后，在 run lifecycle 中加入 `skills.reload`，让 tenant skill changes 无需重启 active runs 即可生效。需要 provider-specific adapter support。

## 待定问题

推迟到实施计划：

- `tenant.json`、`agent.json`、`channels/<channel>.json`、skill manifests 的精确 JSON schemas。
- Per-`(tenant, agent)` host DB 是保持 SQLite，还是为更大的 multi-tenant query patterns 迁移到 embedded Postgres。
- 每个 agent 的 resource limit defaults（memoryMb、pids、cpuShares），很可能允许 tenant override。
- `/var/lib/nanoclaw/users.db` 应使用 SQLite（当前方案）还是更简单的 append-only format。
- Cgroup v2 delegation：控制平面是否获得自己的 delegated cgroup subtree，或由 helper 以 root 管理完整 `/sys/fs/cgroup/nanoclaw/` tree。
