# 版本 1.8：工具 IPC 迁移

## 目标

将业务工具调用从临时文件请求/结果目录迁移到 DB 支持的请求/响应或监督进程 RPC。

## 非目标

- 不在一次提交中迁移所有工具。
- 在所有活跃技能更新完成前不移除兼容层。
- 不将宿主机密钥直接暴露给组用户。

## 问题

当前工具桥接写入如下文件：

```text
ipc/<group>/feishu/requests/<id>.json
ipc/<group>/feishu/results/<id>.json
```

这使得权限管理、超时处理、清理、审计以及隔离任务的并发控制更加困难。在按组分用户的情况下，必须移除全局可写文件。

## 工具请求 DB

添加到 `state.db` 或独立的 `tools.db`。为清晰起见，优先使用 `tools.db`：

```sql
CREATE TABLE IF NOT EXISTS tool_requests (
  id TEXT PRIMARY KEY,
  tenant_id TEXT,
  agent_id TEXT,
  group_folder TEXT NOT NULL,
  chat_jid TEXT,
  requester_run_id TEXT,
  requester_user TEXT,
  tool TEXT NOT NULL,
  tool_family TEXT,
  payload TEXT NOT NULL,
  auth_context TEXT,
  timeout_ms INTEGER NOT NULL,
  idempotency_key TEXT,
  created_at TEXT NOT NULL,
  updated_at TEXT,
  expires_at TEXT,
  claimed_at TEXT,
  completed_at TEXT,
  status TEXT NOT NULL DEFAULT 'pending',
  result TEXT,
  result_file_id TEXT,
  error TEXT
);

CREATE INDEX IF NOT EXISTS idx_tool_requests_status_created
  ON tool_requests(status, created_at);
```

状态：

```text
pending
claimed
completed
error
timeout
cancelled
```

## 流程

```text
智能体 MCP 工具
  -> 插入 tool_requests 为 pending
  -> 带超时等待结果

宿主机工具工作进程或监督进程
  -> 认领 pending 请求
  -> 执行宿主机侧 API 调用
  -> 写入结果/错误
```

## 待迁移工具

优先级排序：

1. `send_message`
2. `schedule_task`、`update_task`、`cancel_task`、`list_tasks`
3. 飞书只读工具
4. 飞书写入工具
5. 下载/上传
6. 审批
7. 其余业务特定技能

旧版桥接当前包含以下具体工具族，迁移期间应显式跟踪：

- 消息发送：`send_message`
- 定时任务：`schedule_task`、`list_tasks`、`pause_task`、`resume_task`、`cancel_task`、`update_task`
- 会话控制：`new_session`
- 分组管理：`register_group`、`refresh_groups`
- 飞书文档：获取/创建/更新/删除/搜索
- 飞书多维表格：应用、表、字段和记录操作
- 飞书权限：协作者、所有权转移、公开设置
- 飞书卡片/富文本
- 飞书文件操作：资源下载和文件发送
- 飞书 P2P/用户操作：发送给用户、部门查询、姓名查询
- 飞书任务和任务清单操作
- 审批操作：获取/查询/同意/拒绝/转交/评论

优先迁移跨越宿主机/智能体边界且涉及密钥处理的工具。

## 兼容层

迁移期间：

- 新工具实现写入 DB 请求。
- 旧版文件监听器仍继续运行。
- 宿主机工作进程可同时处理 DB 和文件请求。
- 技能作者获得迁移指南。

在租户仓库完成迁移之前，不中断现有租户技能。

## 授权矩阵

宿主机或监督进程必须使用经认证的来源分组来验证每个请求，而不是智能体进程提供的分组 ID。

- `send_message`：非主分组只能向自己的聊天发送消息，除非租户策略授予更多权限。
- 调度工具：非主分组只能管理自己的任务。
- `register_group` 和 `refresh_groups`：仅限主分组。
- `new_session`：仅清除来源分组的 Provider 续接/会话。
- 审批工具：执行 `approval-allowlist.json` 的操作和审批码策略。
- 飞书和通道工具：仅使用宿主机持有的凭据在宿主机侧执行。
- P2P 自动注册：`send_to_user` 记录 `source_group` 以供审计和后续授权使用。

被拒绝的请求应以工具错误行完成，不应挂起直到超时。

## 文件与附件处理

下载和上传需要受控的文件契约。不要将宿主机路径返回给组用户。

推荐布局：

```text
<runtime-dir>/
  files/
  downloads/
```

工具结果应返回运行时文件 ID 或运行文件目录下的容器路径。宿主机工作进程仅在执行通道 API 调用时才将其转换为宿主机路径。

大文件可在 `tools.db` 中存储元数据，在磁盘上存储内容。DB 记录应包含文件 ID、原始文件名、大小、MIME 类型和脱敏错误元数据。

## 超时

每个工具请求必须包含：

```ts
timeoutMs: number
```

默认值：

```text
只读 API 工具：30s
写入 API 工具：60s
下载/上传：可配置
审批等待：仅使用显式长超时
```

工具等待循环在超时时应返回明确的错误信息。

## 审计

存储足够的元数据以回答以下问题：

- 哪个智能体请求了该工具
- 哪次运行发起了该请求
- 属于哪个租户
- 开始和完成时间
- 是否成功
- 脱敏后的请求/响应摘要

不存储原始密钥。

## 测试

- 智能体插入请求，宿主机工作进程完成该请求。
- 超时将请求标记为 timeout。
- 宿主机操作失败返回工具错误。
- 两个隔离任务不会消费彼此的工具结果。
- 兼容阶段旧版文件请求仍然可用。
- 权限阻止其他组用户读取工具 DB。
- 消息、任务、分组和会话工具执行主/自身授权。
- 审批白名单拒绝以工具错误完成。
- 文件下载/发送不暴露宿主机路径或密钥。
- P2P 自动注册记录来源分组元数据。

## 验收标准

- 核心消息发送和任务调度工具不再需要文件 IPC。
- 飞书工具可以在不向组用户暴露飞书密钥的情况下运行。
- 对于已迁移的技能，可以按智能体服务或按组禁用文件 IPC。
- 工具请求可审计，并在适当场景下支持安全重试。
