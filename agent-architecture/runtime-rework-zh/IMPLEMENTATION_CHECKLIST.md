# 运行时重构实施检查清单

本检查清单将设计文档转化为具体编码任务。面向后续不具备原始规划对话的智能体。

逐版本推进。当前版本的默认行为、兼容性检查和测试通过之前，不要开始下一个版本。

## 全局不变量

迁移过程中必须始终满足以下条件：

- 现有部署继续使用默认的 `docker-per-group` 运行时，直到 Version 2.0。
- 现有 `groups/` 配置持续受支持，直到明确的迁移操作将其移除。
- 现有飞书/业务工具在兼容阶段内继续正常工作。
- 现有定时任务语义保持不变：
  - `context_mode: "group"` 使用群聊上下文。
  - `context_mode: "isolated"` 使用无聊天历史的全新一次性运行。
- 隔离任务是上下文/生命周期隔离，而非独立的群组安全边界。
- 密钥不得写入租户 Git 仓库。
- Version 1.6 之后，任何群组运行时目录不得依赖 `0777` 或 `0666` 权限。
- Claude 提供方持续可用，直到 OpenCode 在部署中达到生产级功能对等。
- 每个阶段都必须存在回滚路径。
- 中央宿主 DB 在路由、任务、已注册分组和游标方面持续为权威来源，直到明确迁移。
- 运行时 DB 行必须保留当前消息元数据：通道、聊天 JID、发送者 ID/名称、附件、卡片操作、定时任务 ID 和源消息 ID。
- 宿主侧授权门控仍由宿主/监督进程执行：发送者白名单、触发规则、main/self 权限、审批白名单和挂载白名单。
- 不要将按群组可写的 `agent-runner-src` 静默带入最终的共享智能体容器。
- 额外的挂载在用于智能体容器运行时之前，必须被归类为智能体范围或群组特定。
- reporter/本地 API 技能编辑不得修改租户管理的技能。

## Version 1.1：租户、智能体与技能边界

设计文档：[01-tenant-agent-skill-boundary.md](./01-tenant-agent-skill-boundary.md)

### 代码任务

- 添加租户类型：
  - `src/tenants/types.ts`
  - `RegisteredTenant`
  - `RegisteredAgent`
  - `ResolvedSkill`
  - `AgentLimits`
  - `AgentMount`
- 添加 schema：
  - `schemas/tenant.schema.json`
  - `schemas/agent.schema.json`
  - `schemas/skill-manifest.schema.json`
- 添加加载器：
  - `src/tenants/loader.ts`
  - 读取 `NANOCLAW_TENANTS_DIR`
  - 加载 `tenants/*/tenant.json`
  - 加载 `tenants/*/agents/*/agent.json`
  - 解析 `builtin:`、`tenant:` 和 `agent:` 技能引用
  - 拒绝重复 ID
  - 对缺失引用产生诊断信息
- 添加遗留适配器：
  - 将现有 `groups/<folder>/CLAUDE.md` 映射为合成租户 `legacy`
  - 不改变现有群组行为
  - 保留 JID、触发器、`requiresTrigger`、`isMain`、超时、额外挂载以及 P2P/源元数据
- 添加配置/设置支持：
  - 记录 `NANOCLAW_TENANTS_DIR` 文档
  - 启动时打印已加载的租户/智能体
  - 记录 `approval-allowlist.json`、`sender-allowlist.json` 和 `mount-allowlist.json` 仍为宿主侧策略的说明
- 添加迁移脚本：
  - 默认 dry-run
  - 源：现有 `groups/`
  - 目标：外部租户仓库

### 测试

- 合法的租户仓库能正常加载。
- 重复的租户 ID 被拒绝。
- 租户内重复的智能体 ID 被拒绝。
- 缺失的技能引用产生诊断信息。
- JSON 中疑似密钥的值被拒绝或警告。
- 遗留群组仍能正常加载。
- 遗留已注册群组字段在迁移 dry-run 中完整往返。

### 手动验证

```bash
npm test
NANOCLAW_TENANTS_DIR=/path/to/tenants npm run dev
```

确认启动日志同时显示遗留群组和租户智能体。

### 不得变更

- `src/container-runner.ts` 行为
- `src/group-queue.ts` 行为
- `src/task-scheduler.ts` 任务执行行为

## Version 1.2：运行时驱动抽象

设计文档：[02-runtime-driver.md](./02-runtime-driver.md)

### 代码任务

- 添加运行时类型：
  - `src/runtime/types.ts`
  - `RuntimeDriver`
  - `RuntimeRunInput`
  - `RuntimeRunHandle`
  - `RuntimeRunStatus`
- 添加运行时注册：
  - `src/runtime/index.ts`
  - `createRuntimeDriver(name)`
- 将当前 Docker 行为迁移到：
  - `src/runtime/docker-per-group.ts`
- 保留兼容导出：
  - `runContainerAgent(...)` 初期可作为包装器保留
- 替换直接运行时调用：
  - `src/group-queue.ts` 使用 `runtime.getRunStatus()`
  - `src/task-scheduler.ts` 使用 `runtime.startRun()`
  - `src/container-runtime.ts` 变为 Docker 驱动内部实现或仅作兼容
- 添加配置：
  - `NANOCLAW_RUNTIME_DRIVER=docker-per-group`
  - 默认为 `docker-per-group`

### 测试

- 等效输入下 Docker 参数不变。
- `GroupQueue` 不再直接 shell 调用 `docker inspect`。
- 隔离任务仍传递 `singleShot: true` 和 `isolated: true`。
- 伪造运行时驱动能驱动队列测试。
- 孤儿清理仍按实例 ID 过滤。

### 手动验证

```bash
npm test
NANOCLAW_RUNTIME_DRIVER=docker-per-group npm run dev
```

发送一条普通消息和一条隔离定时任务。行为应与重构前一致。

### 不得变更

- Docker 镜像
- IPC 格式
- agent-runner 入口
- 提供方行为

## Version 1.3：DB-backed IPC

设计文档：[03-db-backed-ipc.md](./03-db-backed-ipc.md)

### 代码任务

- 添加宿主 IPC 模块：
  - `src/runtime-ipc/schema.ts`
  - `src/runtime-ipc/inbound.ts`
  - `src/runtime-ipc/outbound.ts`
  - `src/runtime-ipc/state.ts`
  - `src/runtime-ipc/paths.ts`
- 添加 agent-runner IPC 模块：
  - `container/agent-runner/src/runtime-ipc/schema.ts`
  - `container/agent-runner/src/runtime-ipc/inbound.ts`
  - `container/agent-runner/src/runtime-ipc/outbound.ts`
  - `container/agent-runner/src/runtime-ipc/state.ts`
- 创建数据库：
  - `inbound.db`
  - `outbound.db`
  - `state.db`
- 启用 WAL 和 busy timeout。
- 宿主为普通消息写入 inbound 记录。
- 智能体为提供方结果写入 outbound 记录。
- 宿主轮询 outbound 并投递。
- 兼容阶段保留 stdout 哨兵。
- 遗留文件 IPC 目录保留在 `legacy-ipc` 路径下。
- 保留中央 DB 游标所有权的明确性：
  - `last_timestamp`
  - `last_agent_timestamp`
  - 输出前运行失败时回滚
- 在 inbound/outbound 元数据中包含附件、卡片操作、发送者 ID、定时任务 ID、聊天/通道和源消息 ID。
- 为 inbound 和 outbound 行添加幂等/去重键。

### 测试

- Schema 初始化幂等。
- 并发宿主/智能体访问在正常负载下不失败。
- 智能体能且仅能认领一条消息一次。
- 宿主能标记 outbound 为已投递。
- 隔离任务的 DB 路径与实时 DB 路径分离。
- 遗留文件 IPC 路径仍被创建。
- 游标回滚和去重投递均已覆盖。
- 宿主重启后可恢复 outbound 投递，不会重复发送。

### 手动验证

```bash
npm test
```

运行一次普通对话，确认：

- inbound 行变为 completed
- outbound 行变为 delivered
- 兼容阶段 stdout 响应仍出现在日志中

### 不得变更

- 工具文件 IPC 行为（暂不）
- 一次性回退
- 运行时驱动默认值

## Version 1.4：Agent-runner 轮询循环

设计文档：[04-agent-runner-poll-loop.md](./04-agent-runner-poll-loop.md)

### 代码任务

- 添加 agent-runner 模式解析：
  - `legacy-single-shot`
  - `live`
  - `isolated-task`
- 添加 CLI/环境变量支持：
  - `--runtime-dir`
  - `--tenant`
  - `--agent`
  - `--mode`
  - `--cwd`
- 实现实时轮询循环。
- 实现隔离任务单批次循环。
- 在 `state.db` 中存储提供方续接。
- 通过 `state.db` 键 `control:stop` 添加关闭控制。
- 兼容阶段将旧的 `_close` 文件桥接到 `control:stop`。
- 若运行时目录已配置，优先使用 `outbound.db` 返回结果。
- 为旧运行时保留遗留 stdin/stdout 模式。

### 测试

- 实时模式在启动后处理一条 DB inbound 消息。
- 隔离任务模式在首个结果后退出。
- 续接按提供方键保存。
- 关闭控制停止实时进程。
- 活跃查询期间后续消息被安全处理或排队。
- 遗留单次测试仍通过。

### 手动验证

使用临时运行时目录以实时模式启动 agent-runner。手动或通过辅助工具插入 inbound 行。确认 outbound 行出现。

### 不得变更

- 运行时容器拓扑
- Linux 用户
- 提供方选择语义

## Version 1.5：智能体服务容器监督进程

设计文档：[05-single-container-supervisor.md](./05-single-container-supervisor.md)

### 代码任务

- 添加运行时容器镜像目标或更新 Dockerfile：
  - 包含监督进程
  - 包含 agent-runner
  - 包含运行时依赖
- 添加监督进程：
  - `container/supervisor/src/index.ts` 或等价文件
  - 优先使用 Unix socket JSON-RPC
- 实现监督进程方法：
  - `runs.start`
  - `runs.stop`
  - `runs.close`
  - `runs.status`
  - `runs.list`
  - `events.subscribe`
- 添加宿主运行时驱动：
  - `src/runtime/agent-container.ts`
  - 为每个智能体服务启动一个长驻 Docker 容器
  - 连接到监督进程
  - 映射 `RuntimeDriver` 方法
- 添加运行日志：
  - 每次运行的 stdout/stderr
- 添加监督进程过期运行恢复。

### 测试

- 监督进程启动群组进程。
- 监督进程报告运行中/已退出状态。
- 运行时驱动在目标智能体服务容器不存在时启动它。
- `runs.stop` 终止进程。
- 隔离任务退出并记录状态。
- 旧的 `docker-per-group` 驱动仍通过测试。

### 手动验证

```bash
NANOCLAW_RUNTIME_DRIVER=agent-container npm run dev
```

验证目标智能体服务存在一个 Docker 容器，且消息不会创建额外的按群组容器。

### 不得变更

- 进程用户在 1.6 之前仍为默认容器用户
- DB IPC 保持与旧运行时兼容

## Version 1.6：容器用户隔离

设计文档：[06-container-user-isolation.md](./06-container-user-isolation.md)

### 代码任务

- 添加监督进程 `users.ensure`。
- 添加用户名清理与冲突处理。
- 添加目录准备器：
  - 创建运行时目录
  - chown 到群组用户
  - 设置群组为监督进程群组
  - 设置权限 `0770` 或更严格
- 通过以下方式启动 agent-runner：
  - `setpriv`、`gosu` 或 `su-exec`
  - `--no-new-privs`
- 移除全局可写运行时路径。
- 确保隔离任务使用与实时群组相同的用户。
- 添加可选的按智能体限制配置，即使初始执行不完整。
- 确保密钥不在共享文件或全局环境变量中。
- 拒绝不安全的群组特定额外挂载，而非将其提升为智能体范围挂载。
- 审计遗留提供方环境变量注入，并将其替换为此运行时的凭据代理/作用域令牌。

### 测试

- 群组用户可读写自己的运行时 DB。
- 群组用户无法读取其他群组运行时 DB。
- 群组用户无法写入租户共享技能路径。
- 监督进程可停止和检查所有运行。
- 隔离任务使用与实时智能体相同的 UID。
- 无运行时目录为 `0777` 或文件为 `0666`。
- 其他群组用户无法读取群组特定的额外挂载。
- 提供方凭据不存在于运行时 DB 行、日志和群组可读文件中。

### 手动验证

在运行时容器内：

```bash
id ncg_feishu_main
sudo -u ncg_feishu_main test -r /runtime/groups/feishu-main/live/state.db
sudo -u ncg_feishu_main test ! -r /runtime/groups/ops-group/live/state.db
```

### 不得变更

- 提供方默认值
- 租户配置格式（除添加可选限制外）

## Version 1.7：提供方抽象与 OpenCode

设计文档：[07-opencode-provider.md](./07-opencode-provider.md)

### 代码任务

- 在 agent-runner 中添加提供方接口。
- 将 Claude 特定代码移至 `ClaudeProvider` 之后。
- 添加提供方注册。
- 添加 `MockProvider` 用于测试。
- 添加 `OpenCodeProvider`。
- 添加 OpenCode 依赖和 Docker 镜像安装步骤。
- 添加 MCP 配置转换。
- 在 `state.db` 中按提供方存储续接。
- 在 `agent.json` 中添加提供方配置字段。

### 测试

- 注册可加载 Claude、OpenCode、mock。
- Claude 行为保持兼容。
- OpenCode 在简单提示下发出 init/activity/result。
- OpenCode MCP 转换正确。
- 提供方切换不复用不兼容的续接。
- 隔离任务使用 OpenCode 运行。

### 手动验证

运行两个智能体：

- 一个使用 `provider: claude`
- 一个使用 `provider: opencode`

向两者发送简单消息并确认续接独立。

### 不得变更

- OpenCode 在经运维验证前不应成为默认。
- 不要移除 Claude。

## Version 1.8：工具 IPC 迁移

设计文档：[08-tool-ipc-migration.md](./08-tool-ipc-migration.md)

### 代码任务

- 添加 `tools.db` schema。
- 添加智能体侧工具请求客户端。
- 添加宿主侧工具工作器。
- 按优先级顺序迁移工具：
  - `send_message`
  - 任务调度工具
  - `new_session`
  - `register_group`、`refresh_groups`
  - 飞书只读
  - 飞书写入
  - 下载/上传
  - 审批
  - 飞书 P2P/用户、任务、任务列表、多维表格、协作者、卡片和富文本工具
- 为每个工具请求添加超时。
- 添加清理后的审计字段。
- 在租户技能迁移之前保留遗留文件监听。
- 添加配置标志以按智能体禁用遗留文件 IPC。
- 从源运行时身份而非请求载荷强制执行授权。
- 下载/上传使用运行时文件 ID 或受控运行时路径；不返回宿主路径。
- 对被拒绝的请求以明确的工具错误完成。

### 测试

- DB 工具请求成功完成。
- 超时被记录。
- 宿主侧错误返回给智能体。
- 两个隔离任务不能消费彼此的结果。
- 遗留文件请求仍能工作。
- 已迁移的智能体可禁用文件 IPC。
- main/self 授权拒绝返回工具错误。
- 审批白名单拒绝返回工具错误。
- 下载/上传使宿主路径和密钥不出现在工具结果中。
- P2P 自动注册保留源群组元数据。

### 手动验证

运行已迁移的 `send_message` 和飞书只读工具。确认：

- 请求行变为 completed
- 请求/结果中未存储密钥
- 出站消息已投递

## Version 1.9：技能运行时加载

设计文档：[09-skill-runtime-loading.md](./09-skill-runtime-loading.md)

### 代码任务

- 添加已解析的技能清单类型。
- 扩展租户配置加载器以解析：
  - `builtin:<skill>`
  - `tenant:<skill>`
  - `agent:<skill>`
- 为每个智能体服务生成或挂载 `skills.manifest.json`。
- 添加运行时驱动对只读技能挂载的支持。
- 添加监督进程清单修订检查。
- 添加群组生成技能目录：
  - `/runtime/groups/<group>/skills/generated`
- 添加提供方技能加载器适配器：
  - Claude 适配器
  - OpenCode 适配器
  - 测试用 mock 适配器
- 添加重载行为：
  - 初始实现可在清单变更时重启智能体服务
- 迁移或有意遗留旧的按群组 `.claude/skills` 编辑。
- 在最终运行时中停止挂载可写的 `agent-runner-src`；若暂时保留，用兼容标志和按群组权限守卫。
- 将 reporter/本地 API 技能编辑重映射为群组生成技能。
- 保持群组记忆和对话归档在群组本地。

### 测试

- 缺失的技能引用阻止部署或发出配置的诊断信息。
- 群组用户可读取租户技能。
- 群组用户无法写入租户技能。
- 群组用户可在其运行时目录下写入生成技能。
- 生成技能默认对其他群组不可见。
- 提供方适配器接收相同的标准化清单。
- 修订不匹配被检测到。
- 平台运行器源码在最终运行时中为只读。
- reporter 技能编辑无法修改租户管理的技能。

### 手动验证

在智能体容器内：

```bash
sudo -u ncg_feishu_main test -r /workspace/skills/tenant/acme/acme-approval/SKILL.md
sudo -u ncg_feishu_main test ! -w /workspace/skills/tenant/acme/acme-approval/SKILL.md
sudo -u ncg_feishu_main mkdir -p /runtime/groups/feishu-main/skills/generated/demo
```

### 不得变更

- 不要让运行时写入租户仓库。
- 不要在调度器/路由器中硬编码提供方特定的技能路径。

## Version 2.0：最终架构

设计文档：[20-final-architecture.md](./20-final-architecture.md)

### 代码任务

- 将默认运行时驱动设为 `agent-container-users`。
- 保留 `docker-per-group` 回退。
- 更新部署和设置文档。
- 添加租户和运行时数据的迁移命令。
- 添加运维状态命令：
  - 运行时容器状态
  - 监督进程状态
  - 活跃运行
  - 按群组用户
  - DB 队列深度
  - 中央宿主 DB 游标状态
  - 仍存在的遗留 IPC/会话目录
- 添加清理命令：
  - 过期运行
  - 旧隔离任务目录
  - 旧日志
  - 遗留 `data/ipc` 目录
  - 遗留按群组 `agent-runner-src`
- 默认禁用已迁移租户的遗留文件 IPC。

### 测试

- 完整 e2e 普通消息
- 完整 e2e 隔离任务
- 跨智能体权限拒绝
- 运行时重启恢复
- 提供方切换安全性
- 回滚到 `docker-per-group`
- 触发器/发送者白名单行为
- 卡片操作入队行为
- P2P 发送给用户的自动注册
- reporter 技能/记忆编辑行为
- 远程控制仍为宿主侧且仅限 main 分组

### 手动验证

使用默认设置部署全新环境。确认：

- 每个智能体服务启动一个 Docker 服务/容器
- 智能体容器内的群组以不同 Linux 用户运行
- 普通对话正常
- 隔离任务正常
- 旧运行时可通过环境变量选择以支持回滚
