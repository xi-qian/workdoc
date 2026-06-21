# Version 2.0: Final Architecture

## Goal

Make `agent-container-users` the default runtime and treat the old per-group Docker model as fallback.

For the expanded final topology, message flow, tenant deployment flow, and skill
loading flow, see [Target Architecture Details](./TARGET_ARCHITECTURE_DETAILS.md).

## Final Runtime

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

Tenant is a management layer for deployment configuration and skills. It can group multiple agent services, but it is not the runtime isolation unit.

## Runtime Configuration

Global:

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

Agent service:

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

## Runtime Directory

Inside the `finance` agent service container:

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

On the host, runtime data may be stored under:

```text
data/runtime/agents/finance/groups/feishu-main/...
```

## Security Model

Provided:

- Docker isolates each agent service runtime from the host.
- Linux users isolate group runtime directories inside each agent container.
- DB IPC avoids world-writable file queues.
- tenant-managed skills are mounted read-only into agent service containers.
- host/supervisor holds secrets.
- group users run without root and with `no_new_privs`.

Not provided:

- per-group kernel namespace
- per-group network namespace by default
- same strength as one Docker container per group
- protection against malicious kernel/container escape

This must be documented in user-facing deployment docs.

## Host Control-plane Responsibilities

These remain outside group users and outside provider processes:

- channel connection and credential refresh
- sender allowlist, trigger policy, and main-group authorization
- central host DB cursors and task state
- tool workers that require channel or Feishu credentials
- credential proxy or scoped token minting
- reporter/local API server
- setup/status/verify/service commands
- `/remote-control` host process
- orphan cleanup and rollback driver selection

## Isolated Task Model

Isolated task means:

- no live chat history
- no live continuation
- own run DBs
- own lifecycle
- same group Linux user
- same permission boundary as parent group

It does not mean stronger tenant, agent, or group isolation.

## Provider Model

Supported:

```text
claude
opencode
mock
```

Provider state is per provider and per group:

```text
continuation:claude
continuation:opencode
```

Provider differences are hidden behind `AgentProvider`, but documented where behavior differs.

## Operational Defaults

Recommended for simple usage workloads:

```text
idle stopped group: 0 MB RAM
idle live group: target <150 MB
active light group: target <500 MB
warm start: target <2s
idle reap timeout: 2-5 minutes
```

Capacity should be planned by active groups per agent container, and by number of agent containers per host.

## Migration Checklist

- Tenant repo loaded and validated for deployment/skill configuration.
- Legacy groups migrated or explicitly kept.
- Legacy registered group fields are preserved: JID, channel, trigger, `requiresTrigger`, `isMain`, timeout, additional mounts, and P2P source metadata.
- Central DB cursor semantics are preserved or explicitly migrated.
- RuntimeDriver default changed to `agent-container-users`.
- Agent runtime image includes supervisor, agent-runner, providers, `setpriv` or equivalent.
- One Docker service/container is created per agent service.
- DB IPC enabled for normal group messages.
- Tool DB/RPC enabled for core tools.
- Tenant and agent skills loaded through read-only skill manifest.
- Group-generated skills and group memory replace writable runner-source customization.
- Additional mounts are classified as agent-wide or group-specific and validated against the external allowlist.
- Credential delivery no longer exposes shared secrets through group-readable env/files/logs.
- File IPC compatibility disabled for migrated groups.
- Cross-group permission tests pass inside an agent container.
- OpenCode provider optional and tested.
- Claude provider fallback tested.
- Deployment docs updated.

## Data Migration and Cleanup

The migration command should inventory, copy, or preserve:

- `store/messages.db`
- `groups/<group>/**`
- `data/ipc/<group>/**`
- `data/sessions/<group>/.claude`
- `data/sessions/<group>/agent-runner-src`
- `data/sessions/<group>/isolated-ipc-*`
- `approval-allowlist.json`
- `~/.config/nanoclaw/mount-allowlist.json`
- `~/.config/nanoclaw/sender-allowlist.json`

Cleanup of legacy IPC/session data must be opt-in. Rollback to
`docker-per-group` may require those files.

## Final Acceptance Criteria

- Each agent service runs as a distinct Docker service/container.
- One agent container serves multiple groups.
- Each group inside an agent container runs as a distinct Linux user.
- A group user cannot read another group's runtime DBs or session files.
- live chat uses warm poll loop.
- isolated tasks use fresh run directories and exit after completion.
- host can stop, inspect, and restart group runs through the relevant agent supervisor.
- normal messages and core tools work without file IPC.
- tenant skills can be used by group runs without group write access to tenant repos.
- trigger/sender allowlist, card action, P2P, and main/self authorization behavior matches the old runtime.
- reporter skill/memory edits target group-generated or group-local paths.
- file downloads/uploads work without exposing host paths.
- remote control remains host-side and main-group-only.
- old Docker per-group runtime remains selectable for rollback.
