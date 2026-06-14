# NanoClaw 1.x 运行时重构方案

本目录是实现方案，用于将当前 NanoClaw 1.0 扩展部署从每组一个 Docker 容器和文件 IPC 迁移到智能体服务运行时——每个智能体服务拥有一个 Docker 容器，容器内按组分设 Linux 用户，采用 DB 支撑的 IPC，并支持可插拔的 Provider。

文档以编码参考的形式编写。每个版本包含目标、非目标、设计决策、模块变更、迁移说明和验收检查。

如果原始规划上下文不可用，请从以下文档开始：

- [Implementation Checklist](./IMPLEMENTATION_CHECKLIST.md) — 具体编码任务和验证步骤。
- [Architecture Decision Record](./ADR.md) — 不应被意外撤销的决策。

## 目标架构

```text
Host NanoClaw process
  channels, router, scheduler, tenant config loader
  runtime client
    -> Docker service: agent "assistant-a"
         supervisor
           -> group user: ncg_group_a
                live group process
                isolated task processes
                DB-backed inbound/outbound/state files
           -> group user: ncg_group_b
                live group process
                isolated task processes
    -> Docker service: agent "assistant-b"
         supervisor
           -> group users for that agent's groups
```

最终架构保留 Docker 作为每个智能体服务的主机隔离边界，但避免每组一个 Docker 容器。每个智能体容器内的组隔离由 Linux 用户、目录所有权、ACL 和进程生命周期控制提供。租户是部署和技能管理概念，而非运行时隔离单元。

## 指导原则

- 不要将租户业务代码与平台代码混合。
- 将密钥排除在 Git 和组可读的运行时文件之外。
- 在新运行时实现功能对等之前，保留旧的每组 Docker 运行时。
- 在更改进程拓扑之前，将 IPC 迁移到持久的 SQLite 队列。
- 将隔离任务视为同一组用户下的全新运行，而非更强的安全边界。
- 在 `AgentProvider` 接口后明确区分 Provider 差异。
- 在每次边界变更成为默认行为之前，添加实现测试。

## 版本序列

1. [Version 1.1: Tenant, Agent, and Skill Boundary](./01-tenant-agent-skill-boundary.md)
2. [Version 1.2: Runtime Driver Abstraction](./02-runtime-driver.md)
3. [Version 1.3: DB-backed IPC](./03-db-backed-ipc.md)
4. [Version 1.4: Agent-runner Poll Loop](./04-agent-runner-poll-loop.md)
5. [Version 1.5: Agent Service Container Supervisor](./05-single-container-supervisor.md)
6. [Version 1.6: In-container Group User Isolation](./06-container-user-isolation.md)
7. [Version 1.7: Provider Abstraction and OpenCode](./07-opencode-provider.md)
8. [Version 1.8: Tool IPC Migration](./08-tool-ipc-migration.md)
9. [Version 1.9: Skill Runtime Loading](./09-skill-runtime-loading.md)
10. [Version 2.0: Final Architecture](./20-final-architecture.md)

参考资料：

- [Implementation Checklist](./IMPLEMENTATION_CHECKLIST.md)
- [Architecture Decision Record](./ADR.md)
- [Cross-cutting Migration Contracts](./CROSS_CUTTING_CONTRACTS.md)
- [Target Architecture Details](./TARGET_ARCHITECTURE_DETAILS.md)

## 现有代码热点

当前 fork 中的实现路径：

- `src/index.ts`：通道回调、发送者/触发过滤、自动注册、消息游标、输入指示器、调度器/IPC/报告器连接。
- `src/container-runner.ts`：挂载构建、Docker 参数、进程生成、stdout 哨兵解析。
- `src/container-runtime.ts`：Docker 运行时常量、孤儿清理、主机网关处理。
- `src/group-queue.ts`：活跃组进程跟踪、Docker 健康检查、基于文件的关闭信号。
- `src/task-scheduler.ts`：组上下文任务和隔离的一次性容器任务。
- `container/agent-runner/src/index.ts`：当前智能体入口点和 Provider 交互。
- `container/agent-runner/src/ipc-mcp-stdio.ts`：基于文件 IPC 的 MCP 工具桥接。
- `src/ipc.ts`：遗留主机 IPC 监听器，处理消息、任务、组注册、Feishu 工具、审批、文件和会话重置。
- `src/db.ts`：中央主机 DB，存储聊天、消息、路由游标、会话、已注册组、任务和任务运行日志。
- `src/router.ts`：提示词格式化、通道所有权、附件、发送者 ID 和卡片操作格式化。
- `src/credential-proxy.ts`：主机侧 Provider 凭证代理和鉴权模式检测。
- `src/mount-security.ts`：附加主机挂载的外部白名单。
- `src/sender-allowlist.ts` 和 `src/approval-allowlist.ts`：主机授权策略。
- `src/reporter/*`：监控/本地 API，包含组记忆和技能编辑路径。
- `src/remote-control.ts`：主组主机远程控制进程。
- `setup/*` 和 `launchd/com.nanoclaw.plist`：安装、服务、状态检查、验证和启动集成。

在实施每个阶段之前，请对照 [Cross-cutting Migration Contracts](./CROSS_CUTTING_CONTRACTS.md) 进行检查。该文件记录了跨多个版本文档的行为，在运行时重写过程中容易被遗漏。

## 部署策略

每个版本都必须可发布。预期顺序为：

```text
1.1: clean config/skill boundaries
1.2: introduce runtime interface while preserving old Docker behavior
1.3: introduce DB IPC in compatibility mode
1.4: make agent-runner capable of warm poll-loop operation
1.5: introduce one supervisor-backed Docker service per agent
1.6: enforce per-group Linux users and permissions inside each agent container
1.7: add OpenCode as provider
1.8: migrate business tool IPC from files to DB/RPC
1.9: load tenant/agent skills into agent service runtime as read-only inputs
2.0: make new runtime default
```

不要跳过 1.1 或 1.2。用户隔离设计依赖于对租户管理的配置、智能体服务、组运行时、技能和密钥路径的明确所有权。
