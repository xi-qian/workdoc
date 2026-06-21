# NanoClaw 1.x Runtime Rework Plan

This directory is the implementation plan for moving the current NanoClaw 1.0 extended deployment from per-group Docker containers and file IPC to an agent-service runtime where each agent service owns one Docker container, with per-group Linux users inside that container, DB-backed IPC, and pluggable providers.

The documents are written as coding references. Each version has goals, non-goals, design decisions, module changes, migration notes, and acceptance checks.

If the original planning context is unavailable, start with:

- [Implementation Checklist](./IMPLEMENTATION_CHECKLIST.md) for concrete coding tasks and validation steps.
- [Architecture Decision Record](./ADR.md) for decisions that should not be reversed accidentally.

## Target Architecture

```text
Host NanoClaw process
  channels, router, scheduler, tenant config loader
  runtime client
    -> Docker service: agent "assistant-a"
         supervisor
           -> group user: ncg_group_a
                live group process
                isolated task processes
                DB-backed inbound/outbound/state files
           -> group user: ncg_group_b
                live group process
                isolated task processes
    -> Docker service: agent "assistant-b"
         supervisor
           -> group users for that agent's groups
```

The final architecture keeps Docker as the host isolation boundary per agent service, but avoids one Docker container per group. Group isolation inside each agent container is provided by Linux users, directory ownership, ACLs, and process lifecycle controls. Tenant is a deployment and skill-management concept, not the runtime isolation unit.

## Guiding Rules

- Do not mix tenant business code with platform code.
- Keep secrets out of Git and out of group-readable runtime files.
- Keep the old per-group Docker runtime available until the new runtime has parity.
- Move IPC toward durable SQLite queues before changing process topology.
- Treat isolated tasks as fresh runs under the same group user, not as a stronger security boundary.
- Make provider differences explicit behind an `AgentProvider` interface.
- Add implementation tests at every boundary change before making it the default.

## Version Sequence

1. [Version 1.1: Tenant, Agent, and Skill Boundary](./01-tenant-agent-skill-boundary.md)
2. [Version 1.2: Runtime Driver Abstraction](./02-runtime-driver.md)
3. [Version 1.3: DB-backed IPC](./03-db-backed-ipc.md)
4. [Version 1.4: Agent-runner Poll Loop](./04-agent-runner-poll-loop.md)
5. [Version 1.5: Agent Service Container Supervisor](./05-single-container-supervisor.md)
6. [Version 1.6: In-container Group User Isolation](./06-container-user-isolation.md)
7. [Version 1.7: Provider Abstraction and OpenCode](./07-opencode-provider.md)
8. [Version 1.8: Tool IPC Migration](./08-tool-ipc-migration.md)
9. [Version 1.9: Skill Runtime Loading](./09-skill-runtime-loading.md)
10. [Version 2.0: Final Architecture](./20-final-architecture.md)

Supporting references:

- [Implementation Checklist](./IMPLEMENTATION_CHECKLIST.md)
- [Architecture Decision Record](./ADR.md)
- [Cross-cutting Migration Contracts](./CROSS_CUTTING_CONTRACTS.md)
- [Target Architecture Details](./TARGET_ARCHITECTURE_DETAILS.md)

## Existing Code Hotspots

Current implementation paths in this fork:

- `src/index.ts`: channel callbacks, sender/trigger filtering, auto-registration, message cursors, typing indicators, scheduler/IPC/reporter wiring.
- `src/container-runner.ts`: mount construction, Docker args, process spawn, stdout sentinel parsing.
- `src/container-runtime.ts`: Docker runtime constants, orphan cleanup, host gateway handling.
- `src/group-queue.ts`: active group process tracking, Docker health checks, file-based close signaling.
- `src/task-scheduler.ts`: group-context tasks and isolated single-shot container tasks.
- `container/agent-runner/src/index.ts`: current agent entrypoint and provider interaction.
- `container/agent-runner/src/ipc-mcp-stdio.ts`: file IPC based MCP tool bridge.
- `src/ipc.ts`: legacy host IPC watcher for messages, tasks, group registration, Feishu tools, approvals, files, and session reset.
- `src/db.ts`: central host DB for chats, messages, router cursors, sessions, registered groups, tasks, and task run logs.
- `src/router.ts`: prompt formatting, channel ownership, attachments, sender IDs, and card action formatting.
- `src/credential-proxy.ts`: host-side provider credential proxy and auth-mode detection.
- `src/mount-security.ts`: external allowlist for additional host mounts.
- `src/sender-allowlist.ts` and `src/approval-allowlist.ts`: host authorization policy.
- `src/reporter/*`: monitor/local API, including group memory and skill editing paths.
- `src/remote-control.ts`: main-group host remote-control process.
- `setup/*` and `launchd/com.nanoclaw.plist`: setup, service, status, verify, and startup integration.

Before implementing a phase, compare it against [Cross-cutting Migration Contracts](./CROSS_CUTTING_CONTRACTS.md). That file captures behavior that spans multiple version documents and is easy to drop during a runtime rewrite.

## Deployment Strategy

Every version must be shippable. The intended order is:

```text
1.1: clean config/skill boundaries
1.2: introduce runtime interface while preserving old Docker behavior
1.3: introduce DB IPC in compatibility mode
1.4: make agent-runner capable of warm poll-loop operation
1.5: introduce one supervisor-backed Docker service per agent
1.6: enforce per-group Linux users and permissions inside each agent container
1.7: add OpenCode as provider
1.8: migrate business tool IPC from files to DB/RPC
1.9: load tenant/agent skills into agent service runtime as read-only inputs
2.0: make new runtime default
```

Do not skip 1.1 or 1.2. The user isolation design depends on clear ownership of tenant-managed config, agent service, group runtime, skill, and secret paths.
