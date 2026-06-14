# Version 1.6: Container User Isolation

## Goal

Run each tenant/agent under its own Linux user inside the single runtime container. Enforce directory ownership and permissions so agents cannot read or write other agents' runtime state.

## Non-goals

- This is not equivalent to one container per group.
- This does not provide separate network namespaces.
- This does not protect against all kernel/container escapes.
- This does not isolate tasks from their parent agent identity.

## User Model

User name:

```text
nc_<tenant>_<agent>
```

Sanitization:

- lowercase
- `[a-z0-9_]` only
- max length chosen to fit Linux username limits
- collision-resistant suffix if needed

Example:

```text
tenant acme, agent finance -> nc_acme_finance
```

Supervisor group:

```text
nc-supervisor
```

Each agent user is added to a private group and, where needed, shared ACLs grant supervisor access.

## Directory Permissions

Runtime:

```text
/runtime/tenants/acme/agents/finance
  owner: nc_acme_finance
  group: nc-supervisor
  mode: 0770
```

Live run:

```text
/runtime/tenants/acme/agents/finance/live
  owner: nc_acme_finance
  group: nc-supervisor
  mode: 0770
```

Task run:

```text
/runtime/tenants/acme/agents/finance/runs/task-123
  owner: nc_acme_finance
  group: nc-supervisor
  mode: 0770
```

Tenant shared skills:

```text
/workspace/tenants/acme/skills
  owner: root
  group: nc_acme_readers
  mode: 0750
```

Agent-local writable files:

```text
/workspace/tenants/acme/agents/finance/work
  owner: nc_acme_finance
  group: nc-supervisor
  mode: 0770
```

## Process Start

Supervisor prepares directories and starts:

```bash
setpriv \
  --reuid nc_acme_finance \
  --regid nc_acme_finance \
  --init-groups \
  --no-new-privs \
  node /app/agent-runner/dist/index.js ...
```

If `setpriv` is unavailable, use `gosu` or `su-exec`.

## User Creation

Supervisor method:

```text
users.ensure
```

It must:

- validate tenant/agent IDs against loaded config
- create group if missing
- create user if missing
- create home directory under `/home/nc_acme_finance`
- set shell to `/usr/sbin/nologin` if interactive shell is not needed
- ensure directory ownership and modes

## Secrets

Do not put real shared secrets in files readable by agent users.

Preferred:

- host/supervisor holds secrets
- agent calls provider through credential proxy
- agent sees provider-specific placeholder or scoped runtime token

If direct env injection is required for compatibility, inject only into the specific agent process, never into supervisor global env or shared files.

## Isolated Tasks

Isolated tasks use the same user:

```text
live chat:      nc_acme_finance
isolated task:  nc_acme_finance
```

They differ by:

- runtime directory
- DB files
- continuation state
- lifecycle

They are context-isolated, not permission-isolated.

## Resource Limits

Add optional per-run limits:

```json
{
  "memoryMb": 1024,
  "pids": 256,
  "cpuShares": 512,
  "concurrentTasks": 1
}
```

Implementation choices:

- cgroup v2 from supervisor if container is privileged enough
- process-level monitoring and kill as fallback
- Docker container-level limits for total runtime cap

## Tests

Add permission tests inside runtime container:

- finance user cannot read ops `state.db`.
- finance user cannot write tenant shared skills.
- finance user can write its own `outbound.db`.
- isolated task can write its own run DB.
- supervisor can read all run statuses.
- world-writable `0777/0666` paths are absent from runtime directories.

## Acceptance Criteria

- Agents run as non-root per-agent users.
- Cross-agent runtime reads fail with permission denied.
- Current file IPC compatibility paths are not world-writable.
- Host/supervisor can still deliver messages and stop runs.
- The security model is documented as weaker than one-container-per-group but stronger than same-user processes.

