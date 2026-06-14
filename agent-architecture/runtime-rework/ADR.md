# Runtime Rework Architecture Decision Record

This file records decisions that must survive context loss. Future implementation should not reverse these without an explicit new ADR entry.

## ADR-001: Do Tenant and Skill Boundary Before Runtime Isolation

Decision: Split platform code from tenant/agent/business skills before implementing the new runtime.

Reason:

- User isolation needs clear file ownership.
- Runtime mount and permission policy depends on whether a path is platform, tenant, agent, session, or secret state.
- Doing runtime first would make permission rules guesswork and cause rework.

Status: Accepted.

## ADR-002: Keep Per-group Docker Runtime Until New Runtime Has Parity

Decision: Introduce `RuntimeDriver` and keep `docker-per-group` as default until Version 2.0.

Reason:

- Provides rollback.
- Allows each version to ship independently.
- Prevents runtime topology changes from masking config/IPC bugs.

Status: Accepted.

## ADR-003: Use DB-backed IPC as the New Core IPC

Decision: Move core host-agent message exchange to SQLite `inbound.db`, `outbound.db`, and `state.db`.

Reason:

- File queues and sentinel files are fragile with warm processes and isolated tasks.
- SQLite gives durable status, retry, ordering, and audit trails.
- NanoClaw 2.0 already validates this direction.

Status: Accepted.

## ADR-004: Preserve Legacy File IPC Temporarily

Decision: Keep old file IPC as a compatibility layer while migrating business tools.

Reason:

- Existing Feishu/business skills rely on file request/result directories.
- Migrating all tools at once is high risk.
- The DB core can ship first while tool bridge migration proceeds gradually.

Status: Accepted.

## ADR-005: One Long-lived Docker Container, Per-agent Linux Users

Decision: Final runtime uses a single Docker container with a supervisor and per-tenant/agent Linux users.

Reason:

- Docker remains the host isolation boundary.
- Per-agent users are lighter than per-agent Docker containers.
- Simple user workloads benefit from lower warm-start latency and lower idle overhead.

Tradeoff:

- Weaker than one container per group.
- No per-agent network or mount namespace by default.

Status: Accepted.

## ADR-006: Isolated Task Uses Same Agent User

Decision: An isolated task runs as the same Linux user as its parent agent, but with its own runtime directory, DBs, continuation, and lifecycle.

Reason:

- Current semantics mean fresh context, not stronger security.
- The task should access the same files and tools the agent can access.
- Creating a separate user per task would complicate ownership and not match product semantics.

Status: Accepted.

## ADR-007: Isolated Task Must Not Reuse Live Continuation

Decision: `context_mode: "isolated"` must not read or write the live chat continuation.

Reason:

- Current behavior is fresh session/no chat history.
- Task prompts must be self-contained.
- Prevents scheduled/background work from polluting live conversation context.

Status: Accepted.

## ADR-008: Supervisor Owns Process Lifecycle

Decision: In the single-container runtime, host code does not spawn per-agent processes directly. It asks the supervisor.

Reason:

- User creation, chown, chmod, process start, stop, and status are container-internal responsibilities.
- Host should not need to know container-internal user IDs or process tree details.
- Enables one stable RuntimeDriver API.

Status: Accepted.

## ADR-009: Provider Differences Stay Behind AgentProvider

Decision: Claude, OpenCode, and future providers must implement `AgentProvider`; scheduler/router should not contain provider-specific branches.

Reason:

- Claude Agent SDK and OpenCode have different session, hooks, tool, and event models.
- Keeping provider differences isolated prevents runtime code from becoming provider-specific.

Status: Accepted.

## ADR-010: OpenCode Is Optional Before It Is Default

Decision: OpenCode provider is added as an option first. Claude remains available.

Reason:

- OpenCode reduces overhead for simple workloads but is not a drop-in equivalent for Claude Agent SDK.
- Hooks, resume, and subagent behavior differ.
- Production deployments need fallback.

Status: Accepted.

## ADR-011: Secrets Stay Host/Supervisor-side

Decision: Tenant repositories can reference secrets, but cannot store secret values. Agent-readable files should not contain shared secrets.

Reason:

- Business skill repositories may be shared or versioned.
- Agent users should not read secrets for other tenants/agents.
- Credential proxy or scoped env injection is safer.

Status: Accepted.

## ADR-012: No World-writable Runtime IPC After User Isolation

Decision: After Version 1.6, runtime directories must not rely on `0777` directories or `0666` files.

Reason:

- World-writable IPC defeats Linux user isolation.
- Per-agent ownership plus supervisor group access is the intended security boundary.

Status: Accepted.

## ADR-013: Registered Groups Are Not the Same as Active Runs

Decision: Capacity planning and runtime status must distinguish configured agents/groups from active runs.

Reason:

- Stopped idle groups should consume no RAM.
- The new architecture optimizes active light workloads.
- Scheduling and monitoring need queue/run state, not only group registration state.

Status: Accepted.

## ADR-014: Version Documents Are Authoritative for Phase Scope

Decision: Each `0x-*.md` document defines that phase's goals and non-goals. Implementation should not pull later-phase work earlier unless it is required to complete the current phase safely.

Reason:

- Prevents broad rewrites.
- Keeps rollback easy.
- Makes progress reviewable.

Status: Accepted.

