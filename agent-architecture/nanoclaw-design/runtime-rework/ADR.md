# 运行时重构架构决策记录

这是 NanoClaw 2.0 的权威决策记录。此前位于 [`../runtime-rework-archived/ADR.md`](../runtime-rework-archived/ADR.md) 的记录仅作为历史参考保留，不再具有权威性。

标记为 **Reversed** 或 **Deleted** 的决策曾适用于 V1.x 计划，但已经被取代。标记为 **Revised** 的决策保留原始意图，但机制已更新。2.0 设计新增的决策从 ADR-021 开始编号。

## ADR-001: 先建立 Tenant 和 Skill 边界，再做 Runtime 隔离

决策：在实现 runtime 之前，先拆分平台代码与 tenant / agent-service / 业务 skills。

理由：

- Tenant 是部署、配置和 skill 管理层。
- Runtime mount 和权限策略依赖路径类型：平台代码、tenant 管理的配置、agent-service 配置、group runtime state，或 secret state。
- 先做 runtime 会让权限规则变成猜测，并导致返工。

Status: Accepted. 与 V1.x 保持不变。

## ADR-002: ~~保持 Per-group Docker Runtime，直到新 Runtime 达到功能等价~~

原决策：引入 `RuntimeDriver`，并在 Version 2.0 前保持 `docker-per-group` 为默认 runtime。

反转理由：2.0 设计是干净替换。同时维护 `docker-per-group` 和 `host-direct` 两套 runtime 会让测试面翻倍，并让每个跨领域改动都复杂化。Operator 通过迁移 dry-run 验证，而不是通过并行 runtime 验证。

Status: **Reversed 2026-06-16.** 不提供 `docker-per-group` fallback。迁移是一次性的。

## ADR-003: 使用 DB-backed IPC 作为新的核心 IPC

决策：将核心 host-agent/group 消息交换迁移到 SQLite `inbound.db`、`outbound.db`、`state.db` 和 `tools.db`。

理由：

- 文件队列和 sentinel 文件在 warm process 和 isolated task 下很脆弱。
- SQLite 提供持久状态、重试、排序和审计记录。

Status: Accepted. 与 V1.x 保持不变。实现路径更新为 `/var/lib/nanoclaw/runtime/<tenant>/<agent>/<group>/`。

## ADR-004: ~~临时保留 Legacy File IPC~~

原决策：迁移业务 tools 时保留旧文件 IPC 作为兼容层。

删除理由：干净替换。迁移脚本会迁移可迁移内容；legacy file IPC 目录会被备份，但新 runtime 不支持它们。

Status: **Deleted 2026-06-16.**

## ADR-005': 单 Docker Image 是部署单元；Linux User 是隔离单元

取代 ADR-005（每个 agent 服务一个 Docker container，container 内每个 group 一个 Linux user）。

决策：生产环境每个 NanoClaw 实例部署一个 NanoClaw Docker image，或一个等价 Kubernetes pod。该 image 包含控制平面、helper 和运行进程。每个运行进程在该 image 内以其规范 `(tenant, agent, group)` 元组对应的独立映射 `ncg-*` Linux user 运行。Docker/Kubernetes 是打包和运维边界，不是每个 tenant、agent 或 group 的安全边界。

默认 Docker 运维形态：一个长期运行的 `nanoclaw` container，启用 `--privileged`、SUID、持久化 `/var/lib/nanoclaw`，并提供可写或已委派的 cgroup v2 访问。Operator 只有在 helper 操作和隔离测试套件通过后，才可以加固该配置。

理由：

- 在 Linux user 分离之上，Docker 不再额外提供凭据隔离。不同 UID 已经通过进程内存隔离、`/proc/<pid>/environ` 限制和文件权限形成不同安全域。
- 一个 NanoClaw image 服务多个 tenants 和 agents 的部署便利性优于 N 个 Docker services。
- 真正需要保护的是渠道凭据和 tenant runtime data，而 Linux user 隔离可以处理这两者。

权衡：

- 弱于每个 group 一个 container：默认没有 per-group kernel namespace，也没有 per-group network namespace。
- 与 Docker 具有相同的 kernel exploit 威胁面（两者都共享宿主 kernel）。

Status: Accepted 2026-06-16.

## ADR-006: Isolated Task 使用同一个 Group User

决策：isolated task 与其父 group 使用相同 Linux user，但拥有自己的 runtime directory、DB、continuation 和生命周期。

理由：

- 当前语义表示 fresh context，而不是更强的安全边界。
- task 应该访问与 group 相同的文件和 tools。
- 每个 task 创建一个独立 user 会让 ownership 复杂化，并且不符合产品语义。

Status: Accepted. 保持不变。

## ADR-007: Isolated Task 不得复用 Live Continuation

决策：`context_mode: "isolated"` 不得读取或写入 live chat continuation。

理由：

- 当前行为是 fresh session / no chat history。
- Task prompt 必须自包含。
- 防止 scheduled/background work 污染 live conversation context。

Status: Accepted. 保持不变。

## ADR-008': 控制平面拥有生命周期策略；特权机制封装在 SUID Helper 中

修订 ADR-008（supervisor 拥有 group process lifecycle）。

决策：控制平面进程拥有 run lifecycle **policy**，包括何时 prepare runtime、spawn、monitor、reap、kill 和 reconcile。特权 **mechanism**（user/group 创建、runtime ACL 设置、`setuid`、signal、cgroup）封装在 `nc-setuid-helper` 中，这是一个只能由 `nanoclaw-svc` 调用的 SUID root binary。控制平面不持有 Linux capabilities。

理由：

- 两条 Linux 规则决定了这个拆分：`waitpid` 忽略 UID（控制平面可以 reap 由 helper 改过 UID 的子进程），但 `kill` 要求同 UID 或 privilege（控制平面不能 signal `ncg-*` 进程）。
- 把特权面限制在一个小型、可审计的 binary 中，比让整个控制平面带 elevated capabilities 更清晰。
- supervisor-process 方案（原 ADR-008 加 V1.x 计划）是未来扩展，不是初始实现。

Status: Accepted 2026-06-16. 原 ADR-008 设计在 spec 的 “Future extensions” 中列出。

## ADR-009: Provider 差异留在 AgentProvider 后面

决策：Claude、OpenCode 和未来 providers 必须实现 `AgentProvider`；scheduler/router 不得包含 provider-specific 分支。

理由：

- Claude Agent SDK 和 OpenCode 的 session、hooks、tool、event 模型不同。
- 将 provider 差异隔离起来，可以防止 runtime 代码变成 provider-specific。

Status: Accepted. 保持不变。

## ADR-010: OpenCode 先作为可选项，再考虑默认

决策：OpenCode provider 先作为选项加入。Claude 仍保持可用。

理由：

- OpenCode 可以降低简单 workload 的开销，但不是 Claude Agent SDK 的直接等价替代。
- Hooks、resume 和 subagent 行为不同。
- 生产部署需要 fallback。

Status: Accepted. 保持不变。

## ADR-011: Secrets 保留在 Host 侧；两级凭据威胁模型

细化原 ADR-011（secrets 留在 host/supervisor 侧）。

决策：Tenant repositories 可以引用 secrets，但不得存储 secret values。Secret refs 有类型：`llm:<name>` 可以注入 run environment，而 `channel:<name>` 只能在控制平面内解析。渠道凭据（`app_secret`、bot tokens）只存在于 `/var/lib/nanoclaw/auth/` 下由 `nanoclaw-svc` 拥有的 `0600` 文件和控制平面进程内存中；run processes 通过 tool IPC 访问渠道。LLM 凭据可以直接通过 env 注入，因为它们指向 NanoClaw 的内部 LLM gateway，而不是公共 provider endpoint。

理由：

- Business skill repositories 可能被共享或版本化。
- 渠道凭据是真实外部凭据；泄露后会导致 tenant impersonation 和 data exfiltration。
- LLM 凭据指向内部 gateway；agent 使用内部 gateway endpoint 和只在内部网络中被接受的内部凭据。即使 `ncg-*` 进程泄露这些凭据，也不能用它们访问外部 LLM provider，也不能从内部网络外使用。

Status: Accepted. 2026-06-16 由 ADR-024 细化。2026-06-17 进一步要求 typed secret refs 和规范 `/var/lib/nanoclaw/auth/` storage。

## ADR-012: Runtime IPC 不得 World-writable

决策：Runtime directories 不得依赖 `0777` 目录或 `0666` 文件。

理由：

- World-writable IPC 会破坏 Linux user 隔离。
- Per-group ownership 加上给 `nanoclaw-svc` 的 per-runtime POSIX ACL 是预期安全边界。避免 shared runtime group，因为它要么会在错误 owner 创建文件时失败，要么可能把所有 run users 的所有 runtime directories 访问权授给彼此。

Status: Accepted. 2026-06-17 细化为使用 per-runtime ACLs，而不是 shared runtime group。

## ADR-013: Registered Groups 不等于 Active Runs

决策：容量规划和 runtime status 必须区分 configured groups 与 active runs。

理由：

- 停止的 idle groups 不应消耗 RAM。
- 新架构优化 active light workloads。
- Scheduling 和 monitoring 需要 queue/run state，而不只是 group registration state。

Status: Accepted. 保持不变。

## ADR-015: Tenant Skills 作为只读 Agent Inputs 进入 Runtime

决策：Tenant-managed 和 agent-managed skills 在部署/配置加载时解析，并通过每次运行的 resolved skill bundle 作为只读输入暴露给 run processes；该 bundle 位于该 run 的 runtime directory 下。Run processes 只能读取为自己的 `(tenant, agent, group, run)` 选择的 skills，且不能修改它们。

理由：

- Tenant 是管理层，不是 runtime isolation unit。
- 一个控制平面可以服务多个共享 agent 配置 skill set 的 groups。
- 允许 group 写 tenant skill repositories 会绕过 review，并使 configuration drift 难以审计。

Status: Accepted. 2026-06-17 细化为要求 per-run skill bundles，而不是 world-readable tenant/agent skill paths。

## ADR-016: Generated Skills 在提升前保持 Group-local

决策：Group 在 runtime 中创建或修改的 skills 存储在该 group 的 runtime directory 下，不会自动复制到 tenant repositories。

理由：

- Runtime-generated content 不应修改部署事实源。
- 提升为 tenant skill 应该显式且可审查。
- Group-local generated skills 保持隔离预期。

Status: Accepted. 保持不变。

## ADR-017: 分区 Host DBs 仍是控制平面事实源

决策：Per-run runtime DBs 是队列和 run-local state。控制平面 host DBs 按 `(tenant, agent)` 物理分区，并且仍然是 chats、message history、scheduled tasks、task run logs、registered groups、router cursors 和 legacy session IDs 的权威来源。每个 per-agent DB 内部，channel-derived state 以 `channel_type` 为 key，因为一个 agent 可能绑定多个 channels。

理由：

- Host message loop 使用 `last_timestamp[channel_type]` 和 `last_agent_timestamp[channel_type, chat_jid]` 避免重复回复并重试 failed runs。
- Scheduled tasks、registered groups、sender policy、card actions 和 channel metadata 是 host-level control-plane data，不是单个 run 的 runtime data。
- 把 runtime DBs 当成第二事实源会造成 cursor drift 和 duplicate delivery 风险。

Status: Accepted. 保持不变。

## ADR-018: Host-side Authorization Gates 留在 Host-side

决策：Sender allowlists、trigger rules、main-group privileges、approval allowlists、mount allowlists、group registration rights 和 channel credential checks 继续由控制平面在执行工作前强制执行。

理由：

- Run processes 不可信，不能让它们自报 authorization。
- Feishu 和 channel credentials 是控制平面 capabilities。
- Main/self authorization 语义必须在 tool IPC 迁移后继续成立。

Status: Accepted. 保持不变。

## ADR-020: Additional Mount Scope 必须显式

决策：Additional host mounts 必须声明自己是 agent-wide 还是 group-specific。Group-specific mounts 不得被静默提升为 agent-wide mounts。

理由：

- 一个控制平面可以服务多个 groups；agent-wide mount 会对该 agent 下的所有 group users 可见，除非权限阻止。
- 现有 external mount allowlist 和 non-main read-only policy 是安全模型的一部分。

Status: Accepted. 保持不变。

## ADR-021: 每个 `(tenant, agent)` 有独立外部渠道身份

决策：Channel registry 使用复合 key `(tenant_id, agent_id, channel_type)`。对于每个声明 channel 的 `agent.json`，loader 使用该 agent 的外部身份构造 channel instance，并用复合 key 注册。

理由：

- 同一 tenant 下的不同 agents 可能使用不同 Feishu apps、Slack workspaces 或 Telegram bots。
- Webhook URL pattern `/<tenant>/<agent>/<channel>/event` 需要 per-(tenant, agent) 粒度才有意义。
- Singleton accessors（`getFeishuChannel()`）收敛为按复合 key lookup。

Status: Accepted 2026-06-16.

## ADR-022: 共享 Webhook HTTP Server，按 Path 路由

决策：控制平面运行单个 HTTP server。每个 channel instance 注册自己的 URL prefix。按 path 路由：`/<tenant>/<agent>/<channel>/event`。

理由：

- Per-instance HTTP servers 会要求为 N 个 tenants 暴露 N 个 ports，运维上不可行。
- WebSocket-mode channels（Feishu WS、Slack Socket Mode）不需要 HTTP path；它们的 events 按 connection identity 路由。
- Webhook-mode channels 把外部配置指向单个共享 server 上的对应 path。

Status: Accepted 2026-06-16.

## ADR-023: 特权操作只能通过 nc-setuid-helper

决策：所有特权操作（用于 user/runtime setup 的 `prepare`、用于 run spawning 的 `setuid`/`setgid`、process signal、cgroup setup）都通过 `nc-setuid-helper`，这是一个安装在 `/usr/lib/nanoclaw/nc-setuid-helper` 的 SUID root binary（mode 4750，owner=root，group=nc-priv）。只有 `nanoclaw-svc` user 在 `nc-priv` 中。Helper 根据 allowlist 校验所有参数：UID 必须匹配 `users.db` 中记录的 `(tenant, agent, group)` mapping；username 包含稳定 tuple hash 以防 sanitisation collisions；runtime dirs 必须匹配记录的 mapping 和预期 ACL；kill/status requests 必须匹配 PID、`/proc/<pid>/stat` start time、expected UID、runtime dir 和 cgroup，以防 PID reuse 错误。

理由：

- 把特权面限制在一个小型、可审计的 binary 中，比让控制平面带 elevated capabilities 更清晰。
- Helper 拒绝所有越界调用，因此控制平面被攻破也不能升级到 arbitrary root。
- 生产 Docker image 会配置 helper 需要的 SUID、ACL、user/NSS 和 cgroup primitives。

Status: Accepted 2026-06-16. 2026-06-17 细化为增加 `prepare`、tuple-hashed usernames、runtime ACL validation 和 PID start-time/cgroup checks。

## ADR-024: 两级凭据威胁模型

决策：凭据按级别分类和处理：

- **LLM credentials (low risk)**：指向 NanoClaw 的内部 LLM gateway。Agent 使用内部 credential 调用内部 gateway endpoint；该 credential 对公共 provider endpoint 无效，在内部网络之外也不可用。Spawn 时通过 env 交付。不使用 proxy，也不使用 scoped tokens。
- **Channel credentials (high risk)**：真实外部凭据（Feishu `app_secret`、Slack bot tokens、Telegram bot tokens、Discord bot tokens）。只存放在 `/var/lib/nanoclaw/auth/` 下由 `nanoclaw-svc` 拥有的 `0600` 文件和控制平面进程内存中。Run processes 通过 tool IPC 访问 channels，永远看不到原始凭据。
- **Runtime data (high risk)**：聊天历史、provider continuation、生成的 skills、下载的 files。由 Linux user ownership、`0700` runtime directories 和给 `nanoclaw-svc` 的 per-runtime POSIX ACL 保护。

理由：

- 把所有凭据都视为同等敏感，会迫使 LLM access 使用 credential proxy，在没有安全收益的情况下增加复杂度（内部 gateway 才是真正边界）。
- 把所有凭据都视为同等低风险，会把 channel secrets 暴露给 run processes，形成真实攻击面。
- 两级模型把保护工作集中在真正存在威胁的位置。

Status: Accepted 2026-06-16. 2026-06-17 细化为明确 LLM credentials 是 internal-gateway credentials，并要求 typed refs。

## ADR-025: Channel Registry 使用复合 Key；移除 Singleton Accessors

决策：移除所有 `getXxxChannel()` singleton accessors。Channel lookup 统一走 `getChannel(tenantId, agentId, channelType)`。Registry 是一个 `Map<compositeKey, ChannelInstance>`。

理由：

- Singleton accessors 编码了每种 channel 只有一个实例的假设，这在 multi-tenant 下会失效。
- 复合 key lookup 让 tenant routing 在每个 call site 都显式化，使错误在 compile time 暴露，而不是 runtime 暴露。

Status: Accepted 2026-06-16.

## 未来 ADR

推迟到实现阶段的决策：

- Per-`(tenant, agent)` host DB 是继续使用 SQLite，还是为了更大的 multi-tenant query patterns 迁移到 embedded Postgres。
- Cgroup v2 delegation：控制平面是否获得自己的 delegated cgroup subtree，或由 helper 以 root 管理完整 `/sys/fs/cgroup/nanoclaw/` tree。
- Per-(tenant, agent) resource limit defaults。
- 随着代码库增长，是否引入独立 supervisor process（原 ADR-008）。
