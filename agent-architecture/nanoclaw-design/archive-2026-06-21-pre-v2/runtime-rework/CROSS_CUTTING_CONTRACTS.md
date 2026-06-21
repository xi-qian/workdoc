# 跨领域契约

本文记录跨实现阶段的行为。阶段计划定义什么时候做；本文件定义从 V1.x runtime 迁移到 V2.0 host-direct multi-tenant runtime 时不能丢失的行为。

实现阶段由实施计划生成。实现任何阶段前，都应先阅读本文。

## 当前源码清单

重构会触及或替换的代码：

**控制平面（host-side，当前进程）**：

- `src/index.ts`：orchestrator，负责 state、message loop、agent invocation。包含 channel callbacks、sender filtering、auto-registration、message cursors、typing indicators、scheduler/IPC/reporter wiring。
- `src/db.ts`：当前用于 chats、messages、scheduled tasks、task run logs、router cursors、sessions、registered groups 的中央 SQLite。目标形态是每个 `(tenant, agent)` 一个控制平面 SQLite DB，并在每个 DB 内使用 channel-scoped keys。
- `src/router.ts`：channel ownership lookup 和 XML prompt formatting（sender IDs、attachments、card actions、timezone context）。
- `src/group-queue.ts`：per-group concurrency、follow-up message reuse、retry backoff、idle close、active process tracking。
- `src/task-scheduler.ts`：`context_mode` 语义和 isolated task runs。
- `src/sender-allowlist.ts`、`src/approval-allowlist.ts`：host-side authorization gates。
- `src/mount-security.ts`：external mount allowlist 和 non-main read-only policy。
- `src/remote-control.ts`：main-group host process，用于 Claude remote control。
- `src/reporter/*`：monitor/local API、group memory 和 skill editing。

**Runtime launch 和 filesystem（将由 host-direct + SUID helper 替代）**：

- `src/container-runner.ts`：Docker args、mounts、per-group `.claude`、task/group snapshots、output sentinel parsing、idle timeout handling。
- `src/container-runtime.ts`：Docker runtime selection、host gateway、read-only mount helpers、orphan cleanup。
- `src/group-folder.ts`：folder validation 和 runtime path resolution。
- `src/credential-proxy.ts`：Anthropic credential proxy（两级模型下不再需要，将删除或改作他用）。

**Channels 和 Feishu（将变为每个 `(tenant, agent)` 多实例）**：

- `src/channels/registry.ts`：singleton-keyed registry。将迁移到复合 `(tenant, agent, type)` keys。
- `src/channels/feishu.ts`：singleton Feishu channel。将变为 multi-instance。
- `src/feishu/auth.ts`：硬编码凭据文件路径。将解析 `/var/lib/nanoclaw/auth/tenants/<tenant>/<agent>/` 下的 typed refs。
- `src/feishu/client.ts`：per-instance client。已经具备 multi-instance 状态安全性。

**IPC、tools 和 host-only capabilities**：

- `src/ipc.ts`：legacy file IPC watcher，负责 messages、scheduling、group registration、session reset、Feishu tools、approvals、uploads/downloads、P2P chat auto-registration。
- `container/agent-runner/src/ipc-mcp-stdio.ts`：MCP tool definitions，写 legacy IPC files 并等待 result files。
- `container/agent-runner/src/index.ts`：container entrypoint。将变成 run process entrypoint，由 SUID helper 调用。

**Setup 和 operations**：

- `setup/*`：setup、status、service、verify、environment、container、group registration flows。
- Docker image entrypoint、Kubernetes manifests、`deploy.sh`、legacy launchd/systemd wrappers：service management。
- Root config：`.env`、`approval-allowlist.json`、`~/.config/nanoclaw/*.json`。

## 数据库：Host DB 和 Runtime DB

控制平面事实源按 `(tenant, agent)` 分区：每个 agent service 拥有自己的 host DB，例如 `/var/lib/nanoclaw/data/tenants/<tenant>/<agent>/messages.db`。Per-run runtime DBs 是单个 live run 或 isolated task 的持久队列和状态；它们不是 host routing state 的替代品。

Tenant 和 agent 是物理 DB 分区。`channel_type` 仍是每个 DB 内必需的逻辑 key，因为一个 agent 可能绑定多个 channels，而这些 channels 的 chat IDs、message IDs、cursor formats 和 timestamp ordering 彼此独立。

### 所有权：Central DB

- `chats`：channel/chat discovery 和 names。Identity 是 per-agent DB 内的 `(channel_type, jid)`。
- `messages`：权威 inbound history、bot-message filtering、attachments、card actions、scheduled task linkage。唯一性包含 `channel_type` 以及 channel message identity。
- `scheduled_tasks`、`task_run_logs`：task 事实源和审计。
- `router_state`：per-channel `last_timestamp` 和 per-`(channel_type, chat_jid)` `last_agent_timestamp`。
- `sessions`：legacy Claude session IDs，直到 provider state 完全 runtime-scoped。Keys 不得是裸 JID；必须使用 runtime group identity。
- `registered_groups`：group-folder routing、trigger config、JID/channel ownership。Identity 是 per-agent DB 内的 `(channel_type, jid)`。
- `active_runs`：用于 restart reconciliation 和 kill/status validation 的 run lifecycle identity：`pid`、`/proc/<pid>/stat` start time ticks、expected uid、runtime dir、cgroup path、tenant、agent、group 和 run id。

### 所有权：Runtime DB

- `inbound.db`：一个 run 已领取的工作。
- `outbound.db`：一个 run 发出的 delivery requests。
- `state.db`：run-local control keys 和 provider continuation。也用于 per-run audit（model、tokens、latency、status）。
- `tools.db`：一个 run 生成的 tool requests。

不要让两个 DB 层各自独立决定 routing progress。Host 必须在一个地方推进和回滚 cursors。

### 规则：Cursor

- `last_timestamp[channel_type]` 表示“host message loop 已看到此 channel 的消息”。
- `last_agent_timestamp[channel_type, chat_jid]` 表示“此 channel/chat 最新一条用户消息已成功交给 agent run 或 active run”。
- 如果 agent run 在交付任何用户可见输出前失败，回滚 `last_agent_timestamp[channel_type, chat_jid]`，让下一次 run 可以重试相同消息。
- 如果已经交付输出，之后发生 provider error，不回滚 cursor（否则有重复回复风险）。
- Runtime inbound rows 应携带 `channel_type` 和 source message IDs，以便幂等重试。

## 消息元数据契约

DB-backed IPC 必须保留当前能到达 prompt 或 delivery layer 的全部 metadata。

### 记录：Inbound

- tenant ID、agent ID、group folder、chat JID、channel type/name（必需 routing identity）
- source message ID 或 scheduled task ID
- sender ID、sender name、`is_from_me`、trigger reason
- content、timestamp、message type、attachment JSON、card action JSON
- dedupe key、attempt count

### 记录：Outbound

- tenant ID、agent ID、group folder、chat JID、channel type/name（必需 routing identity）
- source inbound IDs 或 tool request ID
- text content、optional sender/persona、message type、attachment path/key
- delivery status、channel message ID、error、retry count、idempotency key

Host outbound poller 负责 channel delivery、typing indicator cleanup，以及审计所需的任何 central DB updates。

## 路由和授权

这些 gates 留在 host-side。Run process 可以请求 work 或 tools，但控制平面必须先校验 identity 和 policy，之后才执行。

### 入站消息 gates

- 通过复合 key `(tenant, agent, channelType)` 判断 channel ownership。
- Registered group lookup（现在按 tenant scoped）。
- 启用时 auto-registration（tenant-scoped）。
- Sender allowlist drop mode 在存储 content 前执行。
- Non-main groups 的 trigger allowlist。
- Card actions 绕过普通 trigger delay，并立即 enqueue。

### 权限：Group

- Main group 可以注册 groups、刷新 group metadata、在其 tenant 内跨 groups 操作。
- Non-main groups 只能为自己 send/schedule/update/cancel，除非 tenant policy 显式授予更多权限。
- 由 `send_to_user` 创建的 P2P chats 保留 `source_group`，用于 authorization 和 audit。

### Gates：Tool

- Scheduling tools 强制执行 main/self authorization。
- Approval tools 强制执行 `approval-allowlist.json`。
- Feishu 和其他 channel tools 在 host-side 执行，因为 channel credentials 位于 host。
- Additional mounts 使用 external mount allowlist 和 blocked-pattern policy。

## Tenant、Agent 和 Group 配置

### 要求：Tenant config

`tenant.json` 必须提供：

- `id`：tenant identifier（最长 16 字符，`[a-z][a-z0-9-]*`）
- `name`：display name
- `enabled`：boolean

### 要求：Agent config

`agent.json` 必须提供：

- `id`：agent identifier（最长 16 字符，`[a-z][a-z0-9-]*`）
- `tenant`：parent tenant ID
- `provider`：`claude`、`opencode` 或 `mock`
- `model`：provider-specific model identifier
- `instructions`：instructions file 路径
- `skills`：skill references 列表（`builtin:`、`tenant:`、`agent:`）
- `channels`：此 agent 使用的 channel types 列表
- `envRefs`：typed LLM secret references 列表（`llm:<name>`），在 load time 解析；channel secrets 不是合法 run env refs
- `limits`：resource limits（memoryMb、pids、concurrentTasksPerGroup）

### 表示：Group

Group 是 chat 的 runtime representation，由 `(tenant, agent, channel_type, chat_jid)` 标识。Host DBs 已按 `(tenant, agent)` 分区，因此 `registered_groups` 在每个 per-agent DB 内以 `(channel_type, jid)` 为 key。迁移时必须 round-trip 的字段：

- JID 和 channel ownership（`channel_type`、channel-specific chat ID）
- display name
- folder（映射到 `<tenant>/<agent>/<group>` runtime path）
- trigger pattern
- `requiresTrigger`
- `isMain`
- `containerConfig.timeout`（变为 `limits`）
- `containerConfig.additionalMounts`（分类为 agent-wide 或 group-specific，见下文）
- 如存在 P2P metadata：`is_p2p`、`p2p_user`、`source_group`

## 额外挂载

当前 additional mounts 按 group 声明，并通过 `~/.config/nanoclaw/mount-allowlist.json` 校验。新 runtime 中，一个控制平面可以服务多个 groups，因此 mount scope 很重要。

规则：

- Agent-wide mounts 对该 agent 处理的每个 group 都可见。
- Group-specific mounts 必须挂载到 per-group path 下，并授权给该 group user；否则拒绝。
- Non-main read-only policy 必须继续强制执行。
- 即使 tenant config 请求，blocked secret patterns 仍然必须阻止。
- Host `.env` 和其他 secret files 必须保持 shadowed 或 unmounted。
- Claude `additionalDirectories` 使用的 `/workspace/extra/*` 行为必须为每个 provider 显式映射。

## 密钥和 Provider 凭据

两级凭据威胁模型（ADR-024）：

- **LLM credentials (low risk)**：指向 NanoClaw 的内部 LLM gateway。Agent 使用内部 credential 连接内部 gateway endpoint；该 credential 只在内部网络内被接受，不指向公共 LLM provider endpoint。Spawn 时通过 env 交付。不使用 proxy，也不使用 scoped tokens。出于通用卫生要求，仍保存在 `/var/lib/nanoclaw/auth/tenants/<tenant>/<agent>/llm/credentials.json` 下的 `0600` 文件中。
- **Channel credentials (high risk)**：真实外部凭据。只存在于 `/var/lib/nanoclaw/auth/tenants/<tenant>/<agent>/<channel>/credentials.json` 下由 `nanoclaw-svc` 拥有的 `0600` 文件和控制平面进程内存中。Run processes 通过 tool IPC 访问 channels。
- **Runtime data (high risk)**：聊天历史、continuation、生成的 skills、下载的 files。由 Linux user ownership、`0700` runtime directories，以及只授予 `nanoclaw-svc` 访问权的 per-runtime POSIX ACL 保护。

Secret references 有类型：

- `llm:<name>` 可以解析到 run environment。
- `channel:<name>` 只能在控制平面内解析。
- 未知或无类型 refs 会导致 config validation 失败。

不要把真实 channel credentials 放入：

- tenant repositories
- runtime DB rows
- control plane shared environment（只允许进入具体 run envs）
- logs
- group-readable snapshots

## 可变 Runner 和 Group-local Customization

NanoClaw V1.x 会把 `container/agent-runner/src` 复制到 per-group writable path，并挂载为 `/app/src`。这种 self-modification 机制**不会**进入 V2.0。

最终规则：

- Platform runner、helper 和 provider code 由 image/distribution 拥有，并保持只读。
- Group-created behaviour 位于 group-generated skills 或 group memory files。
- Tenant/agent skills 是 reviewed inputs，并通过 run runtime directory 下的 per-run resolved skill bundle 以只读方式暴露。不得复制到 world-readable `/opt` paths。
- Runtime-generated skills 在显式提升前留在 group runtime directory 下。

Reporter/local API 中编辑 skills 或 memory 的方法必须写入新的 generated skill root 或 group memory path，绝不能写入 tenant skill repositories。

## 必须迁移的 Tool 清单

Tool IPC migration 将 tools 从 legacy file IPC 迁移到 `tools.db`。清单：

- `send_message`
- `schedule_task`, `list_tasks`, `pause_task`, `resume_task`, `cancel_task`, `update_task`
- `new_session`
- `register_group`, `refresh_groups`
- Feishu docs：fetch、create、update、delete、search
- Feishu bitable：app/table/field/record CRUD 和 listing
- Feishu permissions/collaboration/ownership/public settings
- Feishu card/rich text sends
- Feishu resource download 和 file send
- Feishu P2P/user tools：send to user、user department/name lookup
- Feishu task 和 tasklist operations
- Approval query/get/approve/reject/transfer/comment

对于 downloads/uploads，DB rows 应通过受控 runtime file IDs 或 run 的 `files/` / `downloads/` 目录下路径引用 files。避免向 run processes 返回 host paths。

## 等价检查：Provider

Claude provider 当前依赖的行为必须被保留，或显式记录为其他 providers 不支持：

- session resume 和 session-not-found recovery
- `resumeSessionAt` / last assistant UUID behaviour
- PreCompact transcript archive to `conversations/`
- additional directories 和 `CLAUDE.md` loading
- MCP server inheritance by subagents
- agent teams tools
- allowed tool set 和 permission mode
- streaming result markers 和 null-result session updates
- active query 期间推送的 follow-up messages

OpenCode 可以用不同方式实现这些行为，但 provider adapter 必须向 poll loop 暴露稳定的 result、continuation、error 和 follow-up contract。

## 仅 Host 侧操作

这些操作保留在 run processes 外：

- `/remote-control` 和 `/remote-control-end`
- setup/status/verify/service management
- channel connection 和 credential refresh
- reporter websocket/local API server
- orphan cleanup（legacy Docker containers；new runtime 中没有相同意义的 orphans）
- migration commands

最终 runtime status command 应包含 host process state、active runs、queue depth、helper health 和（post-migration）verify output。

## 部署包装要求

参考生产部署是单个 NanoClaw Docker image。每个 NanoClaw 实例运行一个长期容器，不是每个 tenant、agent 或 group 一个容器。

Docker image configuration 必须提供：

- 允许 helper 完成其窄职责的 privilege profile。默认运维形态是 `--privileged`；hardened profile 仍必须允许 SUID execution、`setuid/setgid`、signal mapped `ncg-*` processes、POSIX ACL changes 和 cgroup v2 writes。
- 禁用 `no_new_privileges`，让 `/usr/lib/nanoclaw/nc-setuid-helper` 能以 SUID root 执行。
- 持久 `/var/lib/nanoclaw` storage，底层 filesystem 必须支持 POSIX ACL。
- `/sys/fs/cgroup/nanoclaw/` 下可写或已委派的 cgroup v2 访问。
- Container-local user/group management，或等价的 helper-owned local user database，让 `prepare` 可以创建 mapped `ncg-*` users。
- 当 operators 需要独立 backup 和 rotation policies 时，为 tenant repositories 和 auth storage 提供独立 mounts。

在 Kubernetes 中，每个 NanoClaw 实例使用一个等价 pod，带匹配的 `securityContext`、persistent volume、ACL support 和 cgroup v2 delegation。

默认 Docker profile 是 `--privileged --security-opt no-new-privileges:false --cgroupns=host`，并带持久 `/var/lib/nanoclaw` 和可写 `/sys/fs/cgroup` mounts。只有在该 profile 下 helper operations 和 isolation test suite 通过后，才可以使用更收紧的 runtime profile。

## 清理和迁移数据

一次性 migration command 必须分类并转换或备份：

- `groups/<group>/CLAUDE.md` 和其他 group memory files → `tenants/<t>/agents/<name>/instructions.md` + `agent.json`
- `groups/<group>/logs` → `logs/<t>/<a>/<group>/`
- `store/auth/feishu/credentials.json` → `/var/lib/nanoclaw/auth/tenants/<t>/<name>/feishu/credentials.json`
- `data/ipc/<group>/**` → 仅备份；不迁移（V2.0 不支持 file IPC）
- `data/sessions/<group>/.claude` → 重新打包到 runtime `state.db`
- `data/sessions/<group>/agent-runner-src` → 丢弃（platform code 在新模型中共享）
- `data/sessions/<group>/isolated-ipc-*` → 丢弃
- `data/nanoclaw.db`、`data/messages.db`、`store/messages.db` → 拆分到 `/var/lib/nanoclaw/data/tenants/<tenant>/<agent>/` 下的 per-`(tenant, agent)` host DBs；没有 tenant/agent identity 的 legacy sources 使用 `tenant=<configured-default>` 和 `agent=<folder>`
- `chats`、`messages`、`registered_groups`、`router_state`、`sessions`、`scheduled_tasks` → 凡是 table 存储 channel/chat/message identity 的地方，都填充 `channel_type`；legacy single-channel rows 默认使用源 channel（例如 `feishu`）
- `approval-allowlist.json` → 保留（host-side policy）
- `~/.config/nanoclaw/mount-allowlist.json` → 保留
- `~/.config/nanoclaw/sender-allowlist.json` → 保留

除非 operator 显式传入 cleanup flag，否则 migration 期间不要删除 legacy runtime data。迁移是一次性的；恢复方式是从文件级备份恢复。

## 验收 Gates

在声明 V2.0 可发布前：

- 普通消息在控制平面重启后不会重复或漏回复。
- Triggered 和 non-triggered messages 保持当前行为。
- Sender allowlist drop 和 trigger modes 仍然有效。
- Card actions 仍会立即 enqueue。
- Group 和 isolated scheduled tasks 保留 context 语义。
- `new_session` 清除正确的 provider continuation。
- 对 message、task、group 和 approval tools 强制执行 main/self authorization。
- File download/send 不暴露 host paths 或 secrets。
- Reporter skill/memory edits 指向新的 runtime paths。
- Remote control 仍然 only main-group 且 host-side。
- Run process 不能读取另一个 run 的 runtime DBs。
- Run process 不能读取 channel credentials。
- Run process 不能读取另一个 tenant/agent 的 resolved skill bundle。
- `nanoclaw-svc` 和 mapped run user 都可以 read/write 由任一进程创建的 runtime DB files 和 SQLite sidecars。
- Run process env 只包含 typed `llm:` credentials 和 run config，不包含 `channel:` secrets。
- Webhook routing 能正确区分 `(tenant, agent)` tuples。
- Helper 能幂等 prepare users/runtime dirs，拒绝 tuple-to-username collisions，拒绝 PID start-time/cgroup/runtime-dir mismatches，并拒绝 out-of-scope spawn/kill/cgroup requests。
- Restart reconciliation 把 PID reuse 视为 stale，除非 PID、start time、expected UID、runtime dir 和 cgroup 全部匹配 active-run record，否则绝不 kill 进程。
- Migration dry-run 生成准确转换报告；迁移后 verify 通过。
