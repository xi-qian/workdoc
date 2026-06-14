# 版本 1.3：基于 DB 的 IPC

## 目标

引入基于 SQLite 的持久化 IPC，用于宿主到智能体及智能体到宿主的消息传递，同时保持与现有文件 IPC 的业务工具兼容性。

## 非目标

- 暂不移除文件 IPC。
- 暂不要求 agent-container 运行时。
- 暂不重写所有 MCP 工具。
- 暂不改变 Provider 行为。

## 问题

当前 IPC 依赖：

- Docker stdin 作为初始输入
- stdout 哨兵标记作为输出
- `data/ipc/<group>/input` 下的文件
- `data/ipc/<group>/messages` 下的文件
- `data/ipc/<group>/tasks` 下的文件
- `data/ipc/<group>/feishu/requests` 和 `results` 下的文件

这在每个分组使用单个容器时没有问题，但在热进程、隔离任务、用户权限和崩溃恢复场景下变得脆弱。

## 新 IPC 布局

对于智能体服务容器内的每个活跃分组会话：

```text
data/runtime/agents/<agent>/groups/<group>/live/
  inbound.db
  outbound.db
  state.db
  files/
  downloads/
  legacy-ipc/
```

对于每个分组的隔离任务：

```text
data/runtime/agents/<agent>/groups/<group>/runs/<runId>/
  inbound.db
  outbound.db
  state.db
  files/
  downloads/
  legacy-ipc/
```

旧版部署可映射为：

```text
agent = default
group = <groupFolder>
```

## Schema

`inbound.db`：

```sql
CREATE TABLE IF NOT EXISTS messages_in (
  id TEXT PRIMARY KEY,
  tenant_id TEXT,
  agent_id TEXT,
  group_folder TEXT NOT NULL,
  chat_jid TEXT NOT NULL,
  channel TEXT,
  source_message_id TEXT,
  kind TEXT NOT NULL,
  content TEXT NOT NULL,
  sender TEXT,
  sender_name TEXT,
  is_from_me INTEGER DEFAULT 0,
  source_chat_jid TEXT,
  scheduled_task_id TEXT,
  message_type TEXT,
  attachment_json TEXT,
  card_action_json TEXT,
  trigger_reason TEXT,
  dedupe_key TEXT,
  attempt_count INTEGER NOT NULL DEFAULT 0,
  created_at TEXT NOT NULL,
  status TEXT NOT NULL DEFAULT 'pending',
  claimed_at TEXT,
  completed_at TEXT,
  error TEXT
);

CREATE INDEX IF NOT EXISTS idx_messages_in_status_created
  ON messages_in(status, created_at);
```

`outbound.db`：

```sql
CREATE TABLE IF NOT EXISTS messages_out (
  id TEXT PRIMARY KEY,
  tenant_id TEXT,
  agent_id TEXT,
  group_folder TEXT NOT NULL,
  chat_jid TEXT NOT NULL,
  channel TEXT,
  kind TEXT NOT NULL,
  content TEXT NOT NULL,
  destination TEXT,
  in_reply_to TEXT,
  source_inbound_ids TEXT,
  scheduled_task_id TEXT,
  tool_request_id TEXT,
  sender TEXT,
  message_type TEXT,
  attachment_json TEXT,
  channel_message_id TEXT,
  dedupe_key TEXT,
  attempt_count INTEGER NOT NULL DEFAULT 0,
  created_at TEXT NOT NULL,
  status TEXT NOT NULL DEFAULT 'pending',
  delivered_at TEXT,
  error TEXT
);

CREATE INDEX IF NOT EXISTS idx_messages_out_status_created
  ON messages_out(status, created_at);
```

`state.db`：

```sql
CREATE TABLE IF NOT EXISTS kv (
  key TEXT PRIMARY KEY,
  value TEXT NOT NULL,
  updated_at TEXT NOT NULL
);
```

## 消息流

宿主到智能体：

```text
channel adapter -> router -> inbound.db(messages_in pending)
agent-runner poll loop -> claim -> provider -> completed/error
```

智能体到宿主：

```text
provider result/tool bridge -> outbound.db(messages_out pending)
host outbound poller -> delivery adapter -> delivered/error
```

## 宿主 DB 集成

运行时 DB 在此版本中不替代中央宿主 DB。宿主 DB 仍对以下内容具有权威性：

- `chats`
- `messages`
- `scheduled_tasks`
- `task_run_logs`
- `router_state`
- `sessions`
- `registered_groups`

宿主从中央 DB 消息或定时任务记录写入运行时入站行。运行时入站行应携带 `source_message_id`、`scheduled_task_id`、`chat_jid`、`channel`、发送者元数据、附件 JSON 和卡片动作 JSON，以便提示格式化和重试行为与当前路由器保持一致。

游标归属权保留在宿主侧：

- 当通道消息已被观察到时，推进 `last_timestamp`
- 仅当消息已移交给运行实例或活跃运行时，推进 `last_agent_timestamp[chat_jid]`
- 如果运行实例在用户可见输出投递之前失败，回滚 `last_agent_timestamp[chat_jid]`
- 输出已投递后不回滚，以避免重复回复

宿主出站轮询器负责通道投递、输入指示器清理、投递状态及任何中央审计行。不要让运行时 DB 和中央 DB 独立决定消息是否已投递。

## 兼容适配器

在此版本中，agent-runner 仍可对工具使用旧版文件 IPC。运行时目录包含：

```text
legacy-ipc/
  input/
  messages/
  tasks/
  downloads/
  feishu/requests/
  feishu/results/
```

旧工具代码在此读写。宿主侧适配器将最重要的操作桥接到现有处理器。

## DB 访问规则

- 宿主拥有 DB 创建和 schema 迁移权限。
- agent-runner 可写入 `messages_out`，并更新已认领的 `messages_in`。
- 宿主可写入 `messages_in`，并更新 `messages_out`。
- 在未来的用户隔离运行时中，运行目录归属为 `group-user:nc-supervisor`，权限模式为 `0770`。

SQLite 应使用 WAL 模式：

```sql
PRAGMA journal_mode=WAL;
PRAGMA busy_timeout=5000;
```

## 新模块

```text
src/runtime-ipc/
  inbound.ts
  outbound.ts
  state.ts
  schema.ts
  paths.ts

container/agent-runner/src/runtime-ipc/
  inbound.ts
  outbound.ts
  state.ts
  schema.ts
```

如果构建系统不同，初始阶段允许重复代码。后续可抽取为共享包。

## 迁移路径

1. 为每次运行创建运行时 DB 路径。
2. 对于活跃聊天，将当前通过 stdin 发送的相同 prompt 写入 `inbound.db`。
3. agent-runner 仍接收 stdin 以保持兼容，但如果 `NANOCLAW_RUNTIME_DIR` 存在则读取 `inbound.db`。
4. Provider 结果写入 `outbound.db`，同时输出 stdout 哨兵以保持兼容。
5. 宿主投递从 `outbound.db` 读取。
6. 稳定后，stdout 哨兵仅用于调试。

## 测试

- DB schema 幂等初始化。
- 宿主可入队入站消息，智能体可恰好认领一次。
- 智能体可入队出站消息，宿主可标记为已投递。
- WAL/busy timeout 能处理宿主和智能体的并发访问。
- 隔离任务 DB 不影响活跃 DB。
- 旧版文件 IPC 路径仍可用于工具。
- 入站行保留发送者、附件、卡片动作、定时任务、通道和聊天元数据。
- 当运行实例在输出前失败时执行游标回滚，输出已投递后不执行回滚。
- 宿主重启时，若存在 `channel_message_id` 或 `dedupe_key`，不会重复出站投递。

## 验收标准

- 正常消息路径可通过 DB IPC 完成。
- 出站写入与投递之间的崩溃不会丢失回复。
- 消息 ID 和状态使重试可审计。
- 现有文件 IPC 工具在兼容模式下仍可工作。
