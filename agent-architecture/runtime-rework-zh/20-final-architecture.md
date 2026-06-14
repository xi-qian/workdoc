# 2.0 版本：最终架构

## 目标

将 `agent-container-users` 设为默认运行时，旧的按组 Docker 模型作为回滚备选。

扩展后的最终拓扑、消息流、租户部署流和技能加载流，参见 [Target Architecture Details](./TARGET_ARCHITECTURE_DETAILS.md)。

## 最终运行时

```text
Host NanoClaw
  channel adapters
  router
  scheduler
  tenant config loader
  runtime client
  tool workers
  credential proxy
  reporter/status APIs
  setup/service control

Docker service/container per agent service
  supervisor
  group users
  group agent-runner processes
  read-only mounted builtin/tenant/agent skills
  provider implementations
  DB-backed IPC files
  logs
```

租户是部署配置与技能的管理层。一个租户可以包含多个智能体服务，但它不是运行时隔离单位。

## 运行时配置

全局：

```json
{
  "runtime": {
    "driver": "agent-container-users",
    "idleTimeoutSeconds": 180,
    "maxActiveRunsPerAgent": 20,
    "defaultMemoryMb": 768,
    "defaultPids": 256
  }
}
```

智能体服务：

```json
{
  "tenant": "acme",
  "agent": "finance",
  "provider": "opencode",
  "model": "anthropic/claude-sonnet-4",
  "idleTimeoutSeconds": 300,
  "limits": {
    "memoryMb": 1024,
    "pids": 256,
    "concurrentTasksPerGroup": 1
  }
}
```

## 运行时目录

在 `finance` 智能体服务容器内：

```text
/runtime/
  supervisor/
    supervisor.sock
    runs.json
  groups/
    feishu-main/
      live/
        inbound.db
        outbound.db
        state.db
        tools.db
      runs/
        task-123/
          inbound.db
          outbound.db
          state.db
          tools.db
      skills/
        generated/
  logs/
    feishu-main/
```

宿主机上，运行时数据可存储于：

```text
data/runtime/agents/finance/groups/feishu-main/...
```

## 安全模型

已提供：

- Docker 将每个智能体服务运行时与宿主机隔离。
- Linux 用户隔离智能体容器内各组的运行时目录。
- DB IPC 避免使用全局可写文件队列。
- 租户管理的技能以只读方式挂载到智能体服务容器中。
- host/supervisor 持有密钥。
- 组用户以非 root 身份运行，并启用 `no_new_privs`。

未提供：

- 按组的内核命名空间。
- 默认不提供按组的网络命名空间。
- 与每组一个 Docker 容器同等强度的隔离。
- 防御恶意内核/容器逃逸。

这必须在面向用户的部署文档中明确说明。

## 宿主机控制面职责

以下职责保持 在组用户和 Provider 进程之外：

- 通道连接与凭证刷新
- 发送方白名单、触发策略及主组授权
- 中心宿主机 DB 游标与任务状态
- 需要通道或 Feishu 凭证的工具工作进程
- 凭证代理或限定范围令牌签发
- reporter/本地 API 服务器
- setup/status/verify/service 命令
- `/remote-control` 宿主机进程
- 孤儿清理与回滚驱动选择

## 隔离任务模型

隔离任务意味着：

- 无实时聊天历史。
- 无实时续接。
- 独立的任务 DB。
- 独立的生命周期。
- 使用同一组的 Linux 用户。
- 与父组相同的权限边界。

它并不意味着更强的租户、智能体或组隔离。

## 提供方模型

已支持：

```text
claude
opencode
mock
```

提供方状态按提供方和按组独立存储：

```text
continuation:claude
continuation:opencode
```

提供方差异由 `AgentProvider` 抽象屏蔽，但在行为差异处予以文档说明。

## 运维默认值

推荐用于常规使用负载：

```text
idle stopped group: 0 MB RAM
idle live group: target <150 MB
active light group: target <500 MB
warm start: target <2s
idle reap timeout: 2-5 minutes
```

容量规划应以每个智能体容器的活跃组数为依据，并以宿主机上的智能体容器数量为依据。

## 迁移检查清单

- 租户仓库已加载并验证部署/技能配置。
- 旧版组已迁移或明确保留。
- 旧版已注册组的字段被保留：JID、channel、trigger、`requiresTrigger`、`isMain`、超时、额外挂载以及 P2P 源元数据。
- 中心 DB 游标语义被保留或已明确迁移。
- RuntimeDriver 默认值已改为 `agent-container-users`。
- 智能体运行时镜像包含 supervisor、agent-runner、providers、`setpriv` 或等效工具。
- 每个智能体服务创建一个 Docker 服务/容器。
- DB IPC 已启用用于常规组消息。
- Tool DB/RPC 已启用用于核心工具。
- 租户和智能体技能通过只读技能清单加载。
- 组生成的技能和组记忆替代可写的 runner-source 定制。
- 额外挂载被归类为智能体范围或组专属，并对照外部白名单进行验证。
- 凭证交付不再通过组可读的 env/文件/日志暴露共享密钥。
- 文件 IPC 兼容性已对已迁移组禁用。
- 智能体容器内的跨组权限测试通过。
- OpenCode 提供方可选并已测试。
- Claude 提供方回滚已测试。
- 部署文档已更新。

## 数据迁移与清理

迁移命令应盘点、复制或保留以下内容：

- `store/messages.db`
- `groups/<group>/**`
- `data/ipc/<group>/**`
- `data/sessions/<group>/.claude`
- `data/sessions/<group>/agent-runner-src`
- `data/sessions/<group>/isolated-ipc-*`
- `approval-allowlist.json`
- `~/.config/nanoclaw/mount-allowlist.json`
- `~/.config/nanoclaw/sender-allowlist.json`

旧版 IPC/会话数据的清理必须为可选择加入。回滚至
`docker-per-group` 可能需要这些文件。

## 最终验收标准

- 每个智能体服务作为独立的 Docker 服务/容器运行。
- 一个智能体容器服务多个组。
- 智能体容器内的每个组以独立的 Linux 用户身份运行。
- 组用户无法读取其他组的运行时 DB 或会话文件。
- 实时聊天使用热轮询循环。
- 隔离任务使用全新运行目录并在完成后退出。
- 宿主机可通过相关智能体 supervisor 停止、检查和重启组运行。
- 常规消息和核心工具无需文件 IPC 即可工作。
- 组运行可在无需对租户仓库的组写入权限的情况下使用租户技能。
- trigger/sender 白名单、卡片操作、P2P 及 main/self 授权行为与旧运行时一致。
- reporter 的技能/记忆编辑定位于组生成或组本地路径。
- 文件下载/上传在不暴露宿主机路径的情况下工作。
- 远程控制保持在宿主机侧且仅限主组。
- 旧的按组 Docker 运行时仍可选择用于回滚。
