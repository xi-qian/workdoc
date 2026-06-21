# Version 1.6: In-container Group User Isolation

## Goal

Inside each agent service container, run each group under its own Linux user. Enforce directory ownership and permissions so groups handled by the same agent service cannot read or write each other's runtime state.

## Non-goals

- This is not equivalent to one container per group.
- This does not provide separate network namespaces per group.
- This does not protect against all kernel/container escapes.
- This does not isolate tasks from their parent group identity.
- This does not merge multiple agent services into one container.

## User Model

User name inside an agent service container:

```text
ncg_<group>
```

Sanitization:

- lowercase
- `[a-z0-9_]` only
- max length chosen to fit Linux username limits
- collision-resistant suffix if needed

Example:

```text
group feishu-main -> ncg_feishu_main
```

Supervisor group:

```text
nc-supervisor
```

Each group user is added to a private group and, where needed, shared ACLs grant supervisor access.

## Directory Permissions

Runtime root for one agent container:

```text
/runtime/
```

Group runtime:

```text
/runtime/groups/feishu-main
  owner: ncg_feishu_main
  group: nc-supervisor
  mode: 0770
```

Live run:

```text
/runtime/groups/feishu-main/live
  owner: ncg_feishu_main
  group: nc-supervisor
  mode: 0770
```

Task run:

```text
/runtime/groups/feishu-main/runs/task-123
  owner: ncg_feishu_main
  group: nc-supervisor
  mode: 0770
```

Tenant shared skills are configuration assets mounted into the agent container:

```text
/workspace/tenants/acme/skills
  owner: root
  group: nc-agent-readers
  mode: 0750
```

Group-local writable files:

```text
/workspace/tenants/acme/agents/finance/groups/feishu-main
  owner: ncg_feishu_main
  group: nc-supervisor
  mode: 0770
```

## Process Start

Supervisor prepares directories and starts:

```bash
setpriv \
  --reuid ncg_feishu_main \
  --regid ncg_feishu_main \
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

- validate agent and group IDs against loaded config and routing state
- create group if missing
- create user if missing
- create home directory under `/home/ncg_feishu_main`
- set shell to `/usr/sbin/nologin` if interactive shell is not needed
- ensure directory ownership and modes

## Secrets

Do not put real shared secrets in files readable by group users.

Preferred:

- host/supervisor holds secrets
- group process calls provider through credential proxy
- group process sees provider-specific placeholder or scoped runtime token

If direct env injection is required for compatibility, inject only into the specific group process, never into supervisor global env or shared files.

Before making this runtime the default, audit the legacy Docker path that passes
provider credentials into containers. The final `agent-container-users` runtime
should prefer a host/supervisor credential proxy or scoped per-run token. Direct
provider env injection is a compatibility fallback and must be visible in config
or logs as such.

## Additional Mounts

NanoClaw 1.0 validates `containerConfig.additionalMounts` per group because each
group gets its own Docker container. In this runtime, an agent service container
can host multiple groups, so mount scope must be explicit:

- agent-wide mounts are visible to all groups handled by that agent service
- group-specific mounts must be mounted under a group-owned path or prepared
  with permissions that prevent other group users from reading it
- non-main read-only policy from the external mount allowlist remains enforced
- blocked secret patterns remain blocked
- host `.env` and credential files remain shadowed or unmounted

If the runtime cannot safely apply a group-specific mount inside a shared agent
container, reject the run with an actionable error rather than widening the mount
to the whole agent service.

## Isolated Tasks

Isolated tasks use the same group user:

```text
live chat:      ncg_feishu_main
isolated task:  ncg_feishu_main
```

They differ by:

- runtime directory
- DB files
- continuation state
- lifecycle

They are context-isolated, not permission-isolated.

## Resource Limits

Add optional per-group or per-run limits:

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
- Docker container-level limits for total agent service cap

## Tests

Add permission tests inside an agent runtime container:

- `feishu-main` user cannot read another group `state.db`.
- `feishu-main` user cannot write tenant shared skills.
- `feishu-main` user can write its own `outbound.db`.
- isolated task can write its own run DB.
- supervisor can read all run statuses.
- world-writable `0777/0666` paths are absent from runtime directories.
- group-specific additional mounts are not readable by other group users.
- provider credentials are absent from group-readable files, DB rows, and logs.

## Acceptance Criteria

- Group runs execute as non-root per-group users inside their agent container.
- Cross-group runtime reads fail with permission denied.
- Current file IPC compatibility paths are not world-writable.
- Host/supervisor can still deliver messages and stop runs.
- The security model is documented as weaker than one-container-per-group but stronger than same-user group processes.
