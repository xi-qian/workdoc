# Version 1.8: Tool IPC Migration

## Goal

Migrate business tool calls from ad hoc file request/result directories to DB-backed request/response or supervisor RPC.

## Non-goals

- Do not migrate every tool in one commit.
- Do not remove compatibility until all active skills are updated.
- Do not expose host secrets directly to agent users.

## Problem

Current tool bridge writes files like:

```text
ipc/<group>/feishu/requests/<id>.json
ipc/<group>/feishu/results/<id>.json
```

This makes permissions, timeouts, cleanup, auditing, and isolated task concurrency harder. With per-agent users, world-writable files must be removed.

## Tool Request DB

Add to `state.db` or separate `tools.db`. Prefer `tools.db` for clarity:

```sql
CREATE TABLE IF NOT EXISTS tool_requests (
  id TEXT PRIMARY KEY,
  tool TEXT NOT NULL,
  payload TEXT NOT NULL,
  requester_run_id TEXT,
  created_at TEXT NOT NULL,
  claimed_at TEXT,
  completed_at TEXT,
  status TEXT NOT NULL DEFAULT 'pending',
  result TEXT,
  error TEXT
);

CREATE INDEX IF NOT EXISTS idx_tool_requests_status_created
  ON tool_requests(status, created_at);
```

Status:

```text
pending
claimed
completed
error
timeout
cancelled
```

## Flow

```text
agent MCP tool
  -> insert tool_requests pending
  -> wait for result with timeout

host tool worker or supervisor
  -> claim pending request
  -> execute host-side API call
  -> write result/error
```

## Tools to Migrate

Priority order:

1. `send_message`
2. `schedule_task`, `update_task`, `cancel_task`, `list_tasks`
3. Feishu read-only tools
4. Feishu write tools
5. downloads/uploads
6. approvals
7. remaining business-specific skills

Start with tools that cross the host/agent boundary and handle secrets.

## Compatibility Layer

During migration:

- New tool implementation writes DB request.
- Legacy file watcher still runs.
- Host worker can process both DB and file requests.
- Skill authors get a migration guide.

Do not break existing tenant skills until the tenant repo has been migrated.

## Timeouts

Every tool request must include:

```ts
timeoutMs: number
```

Default:

```text
read-only API tools: 30s
write API tools: 60s
downloads/uploads: configurable
approval waits: explicit long timeout only
```

Tool wait loop should return a clear error on timeout.

## Auditing

Store enough metadata to answer:

- which agent requested the tool
- which run requested it
- which tenant it belonged to
- when it started and completed
- whether it succeeded
- sanitized request/response summaries

Do not store raw secrets.

## Tests

- agent inserts request and host worker completes it.
- timeout marks request timeout.
- failed host operation returns tool error.
- two isolated tasks do not consume each other's tool result.
- legacy file request still works during compatibility phase.
- permissions prevent another agent user from reading tool DB.

## Acceptance Criteria

- Core message delivery and task scheduling tools no longer require file IPC.
- Feishu tools can run without exposing Feishu secrets to agent users.
- File IPC can be disabled per tenant/agent for migrated skills.
- Tool requests are auditable and retry-safe where appropriate.

