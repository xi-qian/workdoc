# Version 2.0: Final Architecture

## Goal

Make `single-container-users` the default runtime and treat the old per-group Docker model as fallback.

## Final Runtime

```text
Host NanoClaw
  channel adapters
  router
  scheduler
  tenant config loader
  runtime client
  tool workers

Runtime Docker Container
  supervisor
  agent users
  agent-runner processes
  provider implementations
  DB-backed IPC files
  logs
```

## Runtime Configuration

Global:

```json
{
  "runtime": {
    "driver": "single-container-users",
    "idleTimeoutSeconds": 180,
    "maxActiveRuns": 20,
    "defaultMemoryMb": 768,
    "defaultPids": 256
  }
}
```

Agent:

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
    "concurrentTasks": 1
  }
}
```

## Runtime Directory

```text
/runtime/
  supervisor/
    supervisor.sock
    runs.json
  tenants/
    acme/
      agents/
        finance/
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
  logs/
```

## Security Model

Provided:

- Docker isolates runtime from host.
- Linux users isolate tenant/agent runtime directories.
- DB IPC avoids world-writable file queues.
- host/supervisor holds secrets.
- agent users run without root and with `no_new_privs`.

Not provided:

- per-agent kernel namespace
- per-agent network namespace by default
- same strength as one Docker container per group
- protection against malicious kernel/container escape

This must be documented in user-facing deployment docs.

## Isolated Task Model

Isolated task means:

- no live chat history
- no live continuation
- own run DBs
- own lifecycle
- same agent Linux user
- same permission boundary as parent agent

It does not mean stronger tenant isolation.

## Provider Model

Supported:

```text
claude
opencode
mock
```

Provider state is per provider:

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

Capacity should be planned by active groups, not registered groups.

## Migration Checklist

- Tenant repo loaded and validated.
- Legacy groups migrated or explicitly kept.
- RuntimeDriver default changed to `single-container-users`.
- Runtime container image includes supervisor, agent-runner, providers, `setpriv` or equivalent.
- DB IPC enabled for normal messages.
- Tool DB/RPC enabled for core tools.
- File IPC compatibility disabled for migrated tenants.
- Cross-agent permission tests pass.
- OpenCode provider optional and tested.
- Claude provider fallback tested.
- Deployment docs updated.

## Final Acceptance Criteria

- One runtime Docker container serves multiple agents.
- Each tenant/agent runs as a distinct Linux user.
- A user cannot read another agent's runtime DBs or session files.
- live chat uses warm poll loop.
- isolated tasks use fresh run directories and exit after completion.
- host can stop, inspect, and restart runs through supervisor.
- normal messages and core tools work without file IPC.
- old Docker per-group runtime remains selectable for rollback.

