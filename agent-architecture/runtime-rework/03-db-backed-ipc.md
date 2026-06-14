# Version 1.3: DB-backed IPC

## Goal

Introduce durable SQLite-based IPC for host-to-agent and agent-to-host messages while preserving compatibility with the current file IPC for business tools.

## Non-goals

- Do not remove file IPC yet.
- Do not require the agent-container runtime.
- Do not rewrite every MCP tool.
- Do not change provider behavior yet.

## Problem

Current IPC relies on:

- Docker stdin for initial input
- stdout sentinel markers for output
- files under `data/ipc/<group>/input`
- files under `data/ipc/<group>/messages`
- files under `data/ipc/<group>/tasks`
- files under `data/ipc/<group>/feishu/requests` and `results`

This works for a single container per group, but it becomes fragile with warm processes, isolated tasks, user permissions, and crash recovery.

## New IPC Layout

For each live group session inside an agent service container:

```text
data/runtime/agents/<agent>/groups/<group>/live/
  inbound.db
  outbound.db
  state.db
  files/
  downloads/
  legacy-ipc/
```

For each isolated task for a group:

```text
data/runtime/agents/<agent>/groups/<group>/runs/<runId>/
  inbound.db
  outbound.db
  state.db
  files/
  downloads/
  legacy-ipc/
```

Legacy deployments can map to:

```text
agent = default
group = <groupFolder>
```

## Schemas

`inbound.db`:

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

`outbound.db`:

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

`state.db`:

```sql
CREATE TABLE IF NOT EXISTS kv (
  key TEXT PRIMARY KEY,
  value TEXT NOT NULL,
  updated_at TEXT NOT NULL
);
```

## Message Flow

Host to agent:

```text
channel adapter -> router -> inbound.db(messages_in pending)
agent-runner poll loop -> claim -> provider -> completed/error
```

Agent to host:

```text
provider result/tool bridge -> outbound.db(messages_out pending)
host outbound poller -> delivery adapter -> delivered/error
```

## Host DB Integration

The runtime DBs do not replace the central host DB in this version. The host DB
is still authoritative for:

- `chats`
- `messages`
- `scheduled_tasks`
- `task_run_logs`
- `router_state`
- `sessions`
- `registered_groups`

The host writes runtime inbound rows from central DB messages or scheduled task
records. Runtime inbound rows should carry `source_message_id`,
`scheduled_task_id`, `chat_jid`, `channel`, sender metadata, attachment JSON,
and card action JSON so prompt formatting and retry behavior match the current
router.

Cursor ownership stays host-side:

- advance `last_timestamp` when channel messages have been observed
- advance `last_agent_timestamp[chat_jid]` only when messages are handed to a
  run or active run
- rollback `last_agent_timestamp[chat_jid]` if a run fails before user-visible
  output is delivered
- do not rollback after output was delivered, to avoid duplicate replies

The host outbound poller is responsible for channel delivery, typing indicator
cleanup, delivery status, and any central audit rows. Do not let both the runtime
DB and central DB independently decide whether a message has been delivered.

## Compatibility Adapter

For this version, the agent-runner can still use legacy file IPC for tools. The runtime directory includes:

```text
legacy-ipc/
  input/
  messages/
  tasks/
  downloads/
  feishu/requests/
  feishu/results/
```

The old tool code reads and writes there. Host-side adapters bridge the most important operations into existing handlers.

## DB Access Rules

- Host owns DB creation and schema migration.
- Agent-runner can write `messages_out` and update claimed `messages_in`.
- Host can write `messages_in` and update `messages_out`.
- In the future user-isolated runtime, ownership is `group-user:nc-supervisor` with mode `0770` for the run directory.

SQLite should use WAL mode:

```sql
PRAGMA journal_mode=WAL;
PRAGMA busy_timeout=5000;
```

## New Modules

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

Duplication is acceptable initially if build systems differ. Later this can become a shared package.

## Migration Path

1. Create runtime DB paths for each run.
2. For live chat, write the same prompt currently sent over stdin into `inbound.db`.
3. Agent-runner still receives stdin for compatibility, but if `NANOCLAW_RUNTIME_DIR` exists it reads `inbound.db`.
4. Provider result is written to `outbound.db` and also stdout sentinel for compatibility.
5. Host delivery reads from `outbound.db`.
6. Once stable, stdout sentinel becomes debug-only.

## Tests

- DB schema initializes idempotently.
- Host can enqueue inbound messages and agent can claim them exactly once.
- Agent can enqueue outbound messages and host can mark them delivered.
- WAL/busy timeout handles concurrent host and agent access.
- isolated task DB does not affect live DB.
- legacy file IPC path still exists for tools.
- inbound rows preserve sender, attachment, card action, scheduled task, channel, and chat metadata.
- cursor rollback happens when a run fails before output, and does not happen after delivered output.
- host restart does not duplicate outbound delivery when `channel_message_id` or `dedupe_key` is present.

## Acceptance Criteria

- Normal message path can complete using DB IPC.
- A crash between outbound write and delivery does not lose the reply.
- Message IDs and statuses make retries auditable.
- Existing file IPC tools still work in compatibility mode.
