# Version 1.8: Tool IPC Migration

## Goal

Migrate business tool calls from ad hoc file request/result directories to DB-backed request/response or supervisor RPC.

## Non-goals

- Do not migrate every tool in one commit.
- Do not remove compatibility until all active skills are updated.
- Do not expose host secrets directly to group users.

## Problem

Current tool bridge writes files like:

```text
ipc/<group>/feishu/requests/<id>.json
ipc/<group>/feishu/results/<id>.json
```

This makes permissions, timeouts, cleanup, auditing, and isolated task concurrency harder. With per-group users, world-writable files must be removed.

## Tool Request DB

Add to `state.db` or separate `tools.db`. Prefer `tools.db` for clarity:

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

The legacy bridge currently includes these concrete tool families and should be
tracked explicitly during migration:

- message delivery: `send_message`
- scheduled tasks: `schedule_task`, `list_tasks`, `pause_task`, `resume_task`, `cancel_task`, `update_task`
- session control: `new_session`
- group management: `register_group`, `refresh_groups`
- Feishu docs: fetch/create/update/delete/search
- Feishu bitable: app, table, field, and record operations
- Feishu permissions: collaborators, ownership transfer, public settings
- Feishu cards/rich text
- Feishu file operations: resource download and file send
- Feishu P2P/user operations: send to user, department lookup, name lookup
- Feishu task and tasklist operations
- approval operations: get/query/approve/reject/transfer/comment

Start with tools that cross the host/agent boundary and handle secrets.

## Compatibility Layer

During migration:

- New tool implementation writes DB request.
- Legacy file watcher still runs.
- Host worker can process both DB and file requests.
- Skill authors get a migration guide.

Do not break existing tenant skills until the tenant repo has been migrated.

## Authorization Matrix

The host or supervisor must verify every request using the authenticated source
group, not a group ID supplied by the agent process.

- `send_message`: non-main groups may send only to their own chat unless tenant policy grants more.
- scheduling tools: non-main groups may manage only their own tasks.
- `register_group` and `refresh_groups`: main group only.
- `new_session`: clears only the source group's provider continuation/session.
- approval tools: enforce `approval-allowlist.json` action and approval-code policy.
- Feishu and channel tools: execute only host-side with host-held credentials.
- P2P auto-registration from `send_to_user` records `source_group` for audit and future authorization.

Rejected requests should complete with a tool error row; they should not hang
until timeout.

## File and Attachment Handling

Downloads and uploads need a controlled file contract. Do not return host paths
to group users.

Recommended layout:

```text
<runtime-dir>/
  files/
  downloads/
```

Tool results should return a runtime file ID or container path under the run's
file directory. The host worker translates that to a host path only while
executing the channel API call.

Large files may store metadata in `tools.db` and content on disk. The DB record
should include file ID, original filename, size, MIME type, and sanitized error
metadata.

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
- permissions prevent another group user from reading tool DB.
- main/self authorization is enforced for message, task, group, and session tools.
- approval allowlist denial completes as a tool error.
- file download/send does not expose host paths or secrets.
- P2P auto-registration records source group metadata.

## Acceptance Criteria

- Core message delivery and task scheduling tools no longer require file IPC.
- Feishu tools can run without exposing Feishu secrets to group users.
- File IPC can be disabled per agent service or per group for migrated skills.
- Tool requests are auditable and retry-safe where appropriate.
