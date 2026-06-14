# NanoClaw 1.x Runtime Rework Plan

This directory is the implementation plan for moving the current NanoClaw 1.0 extended deployment from per-group Docker containers and file IPC to a tenant-aware runtime with one long-lived Docker container, per-agent Linux users, DB-backed IPC, and pluggable providers.

The documents are written as coding references. Each version has goals, non-goals, design decisions, module changes, migration notes, and acceptance checks.

If the original planning context is unavailable, start with:

- [Implementation Checklist](./IMPLEMENTATION_CHECKLIST.md) for concrete coding tasks and validation steps.
- [Architecture Decision Record](./ADR.md) for decisions that should not be reversed accidentally.

## Target Architecture

```text
Host NanoClaw process
  channels, router, scheduler, tenant config loader
  runtime client
    -> one long-lived Docker runtime container
         supervisor
           -> per tenant/agent Linux user
                live agent process
                isolated task processes
                DB-backed inbound/outbound/state files
                provider: claude, opencode, mock, later others
```

The final architecture keeps Docker as the host isolation boundary, but avoids one Docker container per group. Group isolation inside the runtime container is provided by Linux users, directory ownership, ACLs, and process lifecycle controls.

## Guiding Rules

- Do not mix tenant business code with platform code.
- Keep secrets out of Git and out of agent-readable files.
- Keep the old per-group Docker runtime available until the new runtime has parity.
- Move IPC toward durable SQLite queues before changing process topology.
- Treat isolated tasks as fresh runs under the same agent user, not as a stronger security boundary.
- Make provider differences explicit behind an `AgentProvider` interface.
- Add implementation tests at every boundary change before making it the default.

## Version Sequence

1. [Version 1.1: Tenant, Agent, and Skill Boundary](./01-tenant-agent-skill-boundary.md)
2. [Version 1.2: Runtime Driver Abstraction](./02-runtime-driver.md)
3. [Version 1.3: DB-backed IPC](./03-db-backed-ipc.md)
4. [Version 1.4: Agent-runner Poll Loop](./04-agent-runner-poll-loop.md)
5. [Version 1.5: Single Docker Runtime Supervisor](./05-single-container-supervisor.md)
6. [Version 1.6: Container User Isolation](./06-container-user-isolation.md)
7. [Version 1.7: Provider Abstraction and OpenCode](./07-opencode-provider.md)
8. [Version 1.8: Tool IPC Migration](./08-tool-ipc-migration.md)
9. [Version 2.0: Final Architecture](./20-final-architecture.md)

Supporting references:

- [Implementation Checklist](./IMPLEMENTATION_CHECKLIST.md)
- [Architecture Decision Record](./ADR.md)

## Existing Code Hotspots

Current implementation paths in this fork:

- `src/container-runner.ts`: mount construction, Docker args, process spawn, stdout sentinel parsing.
- `src/container-runtime.ts`: Docker runtime constants, orphan cleanup, host gateway handling.
- `src/group-queue.ts`: active group process tracking, Docker health checks, file-based close signaling.
- `src/task-scheduler.ts`: group-context tasks and isolated single-shot container tasks.
- `container/agent-runner/src/index.ts`: current agent entrypoint and provider interaction.
- `container/agent-runner/src/ipc-mcp-stdio.ts`: file IPC based MCP tool bridge.
- `src/db.ts`: legacy central state and scheduled task storage.

## Deployment Strategy

Every version must be shippable. The intended order is:

```text
1.1: clean config/skill boundaries
1.2: introduce runtime interface while preserving old Docker behavior
1.3: introduce DB IPC in compatibility mode
1.4: make agent-runner capable of warm poll-loop operation
1.5: introduce supervisor-backed single runtime container
1.6: enforce per-agent Linux users and permissions
1.7: add OpenCode as provider
1.8: migrate business tool IPC from files to DB/RPC
2.0: make new runtime default
```

Do not skip 1.1 or 1.2. The user isolation design depends on clear ownership of tenant, agent, skill, runtime, and secret paths.
