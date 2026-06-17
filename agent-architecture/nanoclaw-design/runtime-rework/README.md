# NanoClaw 2.0 运行时重构

本目录记录 NanoClaw 2.0 的运行时架构：一个 host-direct、多租户、基于 Linux user 隔离的运行时，用来替代此前每个 agent 服务一个 Docker 的模型。

## 当前方向

设计已在 [`docs/superpowers/specs/2026-06-16-multi-tenant-host-direct-isolation-design.md`](../superpowers/specs/2026-06-16-multi-tenant-host-direct-isolation-design.md) 中定稿。本目录把该设计同步为项目权威文档（决策记录、目标架构、跨领域契约）。如果两者不一致，以 spec 为事实源，并修正本目录文档。

此前的 V1.x → V2.0 计划（每个 agent 服务一个 Docker）已归档到 [`docs/runtime-rework-archived/`](../runtime-rework-archived/)。那里记录的决策不再具有权威性。

## 目标架构概览

```text
NanoClaw deployment (single Docker image OR equivalent pod)
├── Control plane process (uid=nanoclaw-svc, no capabilities)
│   ├── HTTP webhook server: POST /<tenant>/<agent>/<channel>/event
│   ├── channels: per-(tenant, agent) Feishu / Slack / Telegram / ... clients
│   ├── router, scheduler, sender/trigger policy
│   ├── tool workers (host-side: Feishu, approval, file, task APIs)
│   ├── tenant config loader
│   ├── run lifecycle manager (prepare, spawn, monitor, reap, kill, idle-reap)
│   └── run spawner → invokes nc-setuid-helper
├── nc-setuid-helper (SUID root binary, ~500 lines C)
└── Run processes (uid=mapped ncg-* user for tenant/agent/group)
    └── agent-runner (Claude / OpenCode / mock provider)
```

相对 V1.x 计划的关键变化：

- **隔离单元**：在单个 NanoClaw Docker image 或等价 pod 内，每个 `(tenant, agent, group)` 映射一个 `ncg-*` Linux user。Docker/Kubernetes 是打包边界，不是每个 tenant、agent 或 group 的安全边界。
- **Docker 部署**：默认运维形态是一个长期运行的容器，启用 `--privileged`、SUID、持久化 `/var/lib/nanoclaw`，并提供可写或已委派的 cgroup v2 访问；更收紧的配置必须通过 helper 和隔离测试。
- **进程拓扑**：单个控制平面进程通过 SUID helper 直接启动 run。初始实现不引入 supervisor 进程。
- **每个 host 支持多租户、多 agent**：一个 NanoClaw 实例支持多个租户和多个 agent。每个 `(tenant, agent)` 元组可以拥有自己的外部渠道身份。
- **Webhook 路由**：URL path `/<tenant>/<agent>/<channel>/event` 是单个共享 HTTP server 的路由键。
- **两级凭据模型**：LLM 凭据是只被 NanoClaw 内部 LLM gateway 接受的内部 gateway 凭据，因此可以通过 env 注入。渠道凭据是真实外部凭据，保留在 host 侧，只能通过 tool IPC 访问。
- **干净替换**：没有 `docker-per-group` fallback。一次性迁移脚本负责切换。

## 本目录文档

| 文档 | 用途 |
|----------|---------|
| [ADR.md](./ADR.md) | 架构决策记录。权威文档；取代 `runtime-rework-archived/ADR.md`。 |
| [TARGET_ARCHITECTURE_DETAILS.md](./TARGET_ARCHITECTURE_DETAILS.md) | 组件职责、数据流、消息/任务/tool/skill 序列。 |
| [CROSS_CUTTING_CONTRACTS.md](./CROSS_CUTTING_CONTRACTS.md) | 跨实现阶段的行为：消息元数据、cursor 规则、授权 gate、tool 清单。 |

实现阶段文档（版本顺序、具体编码任务）由实施计划生成，写出后会与本目录并列保存。

## 指导规则

- 不要把租户业务代码与平台代码混在一起。
- 渠道凭据（Feishu `app_secret`、Slack token 等）绝不离开控制平面进程内存或 `0600` 文件。运行进程只能通过 tool IPC 访问渠道。
- LLM 凭据可以在 spawn 时通过 env 注入，因为 agent 调用的是内部 LLM gateway，使用的内部凭据不能用于外部 provider，也不能从内部网络外使用。
- Secret ref 有类型：只有 `llm:` ref 可以进入 run env；`channel:` ref 仅限控制平面。
- Linux user 隔离加每个 runtime 的 ACL 保护运行时数据：聊天历史、continuation、生成的 skills。
- 租户/agent skills 只通过每次运行的只读 resolved bundle 暴露给 run，绝不使用全局可读的租户路径。
- 特权操作（`prepare`、`setuid`、signal、cgroup）只能通过 `nc-setuid-helper` 执行。控制平面不持有 Linux capabilities。
- IPC 迁移到 SQLite 队列（`inbound.db`、`outbound.db`、`state.db`、`tools.db`）。
- Provider 差异留在 `AgentProvider` 后面。
- 在认为运行时可发布之前，实现测试必须覆盖每个隔离边界。

## 现有代码热点

重构会触及或替换的代码：

| 文件 | 当前角色 |
|------|--------------|
| `src/index.ts` | Orchestrator：状态、消息循环、agent 调用 |
| `src/channels/registry.ts` | 以 singleton key 建立的渠道注册表（必须迁移到复合 `(tenant, agent, type)` key） |
| `src/channels/feishu.ts` | Singleton Feishu channel（必须改为每个 `(tenant, agent)` 多实例） |
| `src/feishu/auth.ts` | 硬编码凭据文件路径（必须在 `/var/lib/nanoclaw/auth/tenants/<tenant>/<agent>/` 下解析 typed refs） |
| `src/feishu/client.ts` | 每实例 `FeishuClient`（已经具备多实例状态安全性） |
| `src/container-runner.ts` | 启动 Docker containers，将被 `nc-setuid-helper` 调用替代 |
| `src/group-queue.ts` | 活跃 group 进程跟踪 |
| `src/task-scheduler.ts` | 带 `context_mode` 语义的 cron tasks |
| `src/ipc.ts` | 旧文件 IPC watcher，将被 DB IPC + tool workers 替代 |
| `src/db.ts` | 控制平面 SQLite；目标形态是 per-`(tenant, agent)` DB，带 channel-scoped 路由、tasks 和 cursors |
| `src/router.ts` | 消息格式化和 outbound 路由 |
| `src/credential-proxy.ts` | Anthropic credential proxy（两级模型下不再需要，将删除或改作他用） |
| `container/agent-runner/src/index.ts` | Container entrypoint（变成 run process entrypoint，由 SUID helper 调用） |

## 迁移

一次性迁移脚本负责切换。没有 fallback 模式。转换表和验证步骤见 spec 的 “Migration” 章节。

生产迁移前：

1. 端到端验证 dry-run 报告。
2. 备份所有源数据。
3. 确认 `verify:migration` 通过。
4. 准备按文件级备份恢复的 rollback 方案；没有进程内 rollback。
