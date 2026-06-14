# 跨版本迁移契约

本文档记录跨阶段版本文档的迁移细节。在实施任何阶段之前必须先阅读本文档。版本文档定义工作的时序；本文档定义从自定义 NanoClaw 1.0 运行时迁移到新智能体服务运行时过程中不可丢失的行为。

## 当前 1.0 源码清单

宿主机控制面：

- `src/index.ts`：通道回调、发送者过滤、自动注册、消息游标、打字指示器、调度器/IPC/Reporter 接线。
- `src/db.ts`：中央 SQLite 存储，包括聊天、消息、定时任务、任务运行日志、路由器游标、会话和已注册分组。
- `src/router.ts`：通道归属查找和 XML 提示词格式化，包括发送者 ID、附件、卡片操作和时区上下文。
- `src/group-queue.ts`：按分组的并发控制、后续消息复用、重试退避、空闲关闭和活跃进程跟踪。
- `src/task-scheduler.ts`：`context_mode` 语义和隔离的任务运行。

运行时启动与文件系统：

- `src/container-runner.ts`：Docker 参数、挂载、按分组的 `.claude`、任务/分组快照、可写的 agent-runner 源码副本、输出哨兵解析、日志和空闲超时处理。
- `src/container-runtime.ts`：Docker 运行时选择、宿主网关、只读挂载辅助函数、孤儿清理。
- `src/group-folder.ts`：文件夹验证和运行时路径解析。
- `src/mount-security.ts`：外部挂载白名单和非主分组只读策略。

IPC、工具和宿主机独占能力：

- `src/ipc.ts`：用于消息、调度、分组注册、会话重置、飞书工具、审批、上传/下载和 P2P 聊天自动注册的旧版文件 IPC 监听器。
- `container/agent-runner/src/ipc-mcp-stdio.ts`：写入旧版 IPC 文件并等待结果文件的 MCP 工具定义。
- `src/credential-proxy.ts`：宿主机侧提供方凭证代理。
- `src/approval-allowlist.ts` 和 `src/sender-allowlist.ts`：宿主机侧授权门控。
- `src/remote-control.ts`：主分组宿主机进程用于 Claude 远程控制。
- `src/reporter/*`：监控/本地 API，包括分组记忆和技能编辑。

设置与运维：

- `setup/*`：设置、状态、服务、验证、环境、容器和分组注册流程。
- `launchd/com.nanoclaw.plist`：服务管理。
- `.env`、`approval-allowlist.json` 和 `~/.config/nanoclaw/*.json` 下的根配置文件。

## 宿主机 DB 与运行时 DB

中央宿主机 DB 在后续显式迁移替换之前，始终是控制面的唯一事实来源。按运行实例的运行时 DB 是单次活跃运行或隔离任务的持久队列和状态；它们不是宿主机路由状态的替代品。

中央 DB 归属：

- `chats`：通道/聊天发现和名称。
- `messages`：权威的入站历史、机器人消息过滤、附件、卡片操作和定时任务关联。
- `scheduled_tasks` 和 `task_run_logs`：任务事实来源和审计。
- `router_state`：`last_timestamp` 和 `last_agent_timestamp`。
- `sessions`：旧版 Claude 会话 ID，直到提供方状态完全限定在运行时范围内。
- `registered_groups`：旧版 JID 到文件夹的路由和触发配置。

运行时 DB 归属：

- `inbound.db`：单次运行认领的工作。
- `outbound.db`：单次运行发出的投递请求。
- `state.db`：运行实例本地的控制键和提供方续接。
- `tools.db`：单次运行生成的工具请求。

不要让两层 DB 各自独立决定路由进度。宿主机必须在同一个地方推进和回滚游标。

游标规则：

- `last_timestamp` 表示"宿主机消息循环已看到的通道消息"。
- `last_agent_timestamp[chat_jid]` 表示"已成功交给智能体运行或活跃运行的最新用户消息"。
- 如果智能体运行在任何对用户可见的输出投递之前失败，回滚 `last_agent_timestamp[chat_jid]`，以便下次运行可以重试相同消息。
- 如果输出已投递但后续发生提供方错误，不要回滚游标以避免重复回复。
- 运行时入站记录应携带源消息 ID，使重试具有幂等性。

## 消息元数据契约

DB 支撑的 IPC 必须保留当前到达提示词或投递层的所有元数据。

入站记录至少需要：

- 租户 ID、智能体 ID、分组文件夹、聊天 JID、通道名称
- 源消息 ID 或定时任务 ID
- 发送者 ID、发送者名称、`is_from_me` 和触发原因
- 内容、时间戳、消息类型、附件 JSON、卡片操作 JSON
- 去重键和尝试次数

出站记录至少需要：

- 租户 ID、智能体 ID、分组文件夹、聊天 JID、通道名称
- 源入站 ID 或工具请求 ID
- 文本内容、可选发送者/角色、消息类型、附件路径/键
- 投递状态、通道消息 ID、错误、重试次数、幂等键

宿主机出站轮询器负责通道投递、打字指示器清理以及审计所需的中央 DB 更新。

## 路由与授权

这些门控保留在宿主机侧。分组进程可以请求工作或工具，但宿主机/监督进程必须在执行前验证身份和策略。

入站消息门控：

- 通过 `findChannel` 的通道归属
- 已注册分组查找
- 启用时的自动注册
- 存储内容前的发送者白名单丢弃模式
- 非主分组的触发白名单
- 卡片操作绕过正常的触发延迟，立即入队

分组权限：

- 主分组可以注册分组、刷新分组元数据并跨分组操作。
- 非主分组只能发送/调度/更新/取消自身操作，除非租户策略显式授予更多权限。
- 由 `send_to_user` 创建的 P2P 聊天保留 `source_group` 用于授权和审计。

工具门控：

- 调度工具强制执行主分组/自身授权。
- 审批工具强制执行 `approval-allowlist.json`。
- 飞书和其他通道工具在宿主机或监督进程侧执行，因为通道凭证位于那里。
- 额外挂载使用外部挂载白名单和阻止模式策略。

## 租户、智能体、分组和旧版字段

迁移必须映射这些现有的 `RegisteredGroup` 字段，而不仅仅是 `folder` 和 `CLAUDE.md`：

- JID 和通道归属
- 显示名称
- 文件夹
- 触发模式
- `requiresTrigger`
- `isMain`
- `containerConfig.timeout`
- `containerConfig.additionalMounts`
- P2P 元数据（如存在）：`is_p2p`、`p2p_user`、`source_group`

租户和智能体配置可以拥有默认值，但分组路由策略仍需要分组级别的表示。旧版分组应保留在 `registered_groups` 中，或迁移到具有等价字段的显式租户/智能体/分组配置文件。

## 额外挂载

当前额外挂载按分组声明并根据 `~/.config/nanoclaw/mount-allowlist.json` 验证。在新运行时中，一个智能体容器可以服务多个分组，因此挂载范围很重要。

规则：

- 智能体范围的挂载对该智能体服务中的每个分组可见。
- 分组特定挂载必须挂载到按分组的路径下并赋予该分组用户权限，或在智能体容器运行时中被拒绝。
- 非主分组只读策略必须继续执行。
- 被阻止的密钥模式即使租户配置请求也保持被阻止。
- 宿主机 `.env` 和其他密钥文件必须保持被遮蔽或未挂载。
- Claude `additionalDirectories` 使用的 `/workspace/extra/*` 行为必须为每个提供方有意映射。

## 密钥与提供方凭证

分组可读的运行时文件不得包含共享密钥。用户隔离实施后，提供方凭证应通过以下方式之一提供：

- 宿主机/监督进程凭证代理
- 短生命周期的限定范围运行时令牌
- 限于单个进程的按运行实例环境变量注入，仅作为兼容性回退

不要将真实的提供方或飞书凭证放入：

- 租户仓库
- 运行时 DB 行
- 共享的监督进程环境
- 日志
- 分组可读的快照

在 2.0 版本之前，审计将提供方环境变量传入容器的旧版路径，并替换它或将其限制在显式兼容性标志之后。

## 可变 Runner 与分组本地定制

NanoClaw 1.0 将 `container/agent-runner/src` 复制到按分组的可写路径中并挂载为 `/app/src`。这是一种强大的自修改机制，不得在最终运行时中静默延续。

最终规则：

- 平台 runner/监督进程/提供方代码归镜像所有且为只读。
- 分组创建的行为存在于分组生成的技能或分组记忆文件中。
- 租户/智能体技能是经过审查的输入，以只读方式挂载。
- 运行时生成的技能保留在分组运行时目录下，直到显式提升流程将其发布。

兼容选项：

- 仅为 `docker-per-group` 回滚驱动保留可写的 runner 源码。
- 提供临时的智能体容器兼容性标志，将修改后的 runner 源码存储在仅限分组的运行时目录中，默认禁用。
- 将现有的分组 `.claude/skills` 编辑迁移到生成式技能。

编辑技能或记忆的 Reporter/本地 API 方法必须更新为写入新的生成式技能根目录或分组记忆路径，永远不写入租户技能仓库。

## 必须迁移的工具清单

旧版文件 IPC 包含的不仅仅是通用消息和任务工具。迁移必须考虑：

- `send_message`
- `schedule_task`、`list_tasks`、`pause_task`、`resume_task`、`cancel_task`、`update_task`
- `new_session`
- `register_group`、`refresh_groups`
- 飞书文档：获取、创建、更新、删除、搜索
- 飞书多维表格：应用/表格/字段/记录的 CRUD 和列表查询
- 飞书权限/协作/归属/公开设置
- 飞书卡片/富文本发送
- 飞书资源下载和文件发送
- 飞书 P2P/用户工具：发送给用户、用户部门/名称查找
- 飞书任务和任务列表操作
- 审批查询/获取/批准/拒绝/转交/评论

对于下载/上传，DB 行应通过受控的运行时文件 ID 或运行实例的 `files/` 或 `downloads/` 目录下的路径引用文件。避免将宿主机路径返回给分组进程。

## 提供方功能对等检查

Claude 提供方路径目前依赖的行为必须被保留，或被显式文档化为其他提供方不支持：

- 会话恢复和会话未找到时的恢复
- `resumeSessionAt`/最后助手 UUID 的行为
- PreCompact 会话记录归档到 `conversations/`
- 附加目录和 `CLAUDE.md` 加载
- MCP 服务器被子智能体继承
- 智能体团队工具
- 允许的工具集和权限模式
- 流式结果标记和空结果会话更新
- 活跃查询期间推送的后续消息

OpenCode 可能以不同方式实现这些功能，但提供方适配器必须向轮询循环暴露稳定的结果、续接、错误和后续消息契约。

## 宿主机独占操作

除非后续设计显式更改，否则将这些操作保持在分组用户之外：

- `/remote-control` 和 `/remote-control-end`
- 设置/状态/验证/服务管理
- 通道连接和凭证刷新
- Reporter WebSocket/本地 API 服务器
- 孤儿清理
- 迁移和回滚命令

最终运行时状态命令应包含宿主机进程状态、智能体容器状态、监督进程状态、活跃运行、队列深度和过期的旧版 IPC 目录。

## 清理与迁移数据

迁移命令必须分类并可选地迁移：

- `groups/<group>/CLAUDE.md` 和其他分组记忆文件
- `groups/<group>/logs`
- `data/ipc/<group>/**`
- `data/sessions/<group>/.claude`
- `data/sessions/<group>/agent-runner-src`
- `data/sessions/<group>/isolated-ipc-*`
- `store/messages.db`
- `approval-allowlist.json`
- `~/.config/nanoclaw/mount-allowlist.json`
- `~/.config/nanoclaw/sender-allowlist.json`

迁移期间不要删除旧版运行时数据，除非操作者传递显式清理标志。回滚驱动可能仍需要这些数据。

## 最终兼容性门控

在将 `agent-container-users` 设为默认值之前：

- 正常消息在重启后不重复或跳过回复。
- 触发和非触发消息保留当前行为。
- 发送者白名单丢弃和触发模式仍然有效。
- 卡片操作仍然立即入队。
- 分组和隔离的定时任务保留 context 语义。
- `new_session` 清除正确的提供方续接。
- 主分组/自身授权对消息、任务、分组和审批工具强制执行。
- 文件下载/发送在不暴露宿主机路径或密钥的情况下工作。
- Reporter 技能/记忆编辑指向新的运行时路径。
- 远程控制仍然仅限主分组且位于宿主机侧。
- 回滚到 `docker-per-group` 不需要数据丢失。
