# Runtime Rework Architecture Decision Record

This file records decisions that must survive context loss. Future implementation should not reverse these without an explicit new ADR entry.

## ADR-001: Do Tenant and Skill Boundary Before Runtime Isolation

Decision: Split platform code from tenant/agent-service/business skills before implementing the new runtime.

Reason:

- Tenant is a deployment, configuration, and skill-management layer.
- Runtime mount and permission policy depends on whether a path is platform code, tenant-managed config, agent-service config, group runtime state, or secret state.
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

Decision: Move core host-agent/group message exchange to SQLite `inbound.db`, `outbound.db`, and `state.db`.

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

## ADR-005: One Docker Container per Agent Service, Per-group Linux Users

Decision: Final runtime uses one Docker container/service per agent service. Inside each agent container, groups are isolated with per-group Linux users.

Reason:

- Docker remains the host isolation boundary for each agent service.
- Multiple agents should be multiple Docker services.
- Per-group users are lighter than per-group Docker containers.
- Simple user workloads benefit from lower warm-start latency and lower idle overhead.

Tradeoff:

- Weaker than one container per group.
- No per-group network or mount namespace by default.

Status: Accepted.

## ADR-006: Isolated Task Uses Same Group User

Decision: An isolated task runs as the same Linux user as its parent group, but with its own runtime directory, DBs, continuation, and lifecycle.

Reason:

- Current semantics mean fresh context, not stronger security.
- The task should access the same files and tools the group can access.
- Creating a separate user per task would complicate ownership and not match product semantics.

Status: Accepted.

## ADR-007: Isolated Task Must Not Reuse Live Continuation

Decision: `context_mode: "isolated"` must not read or write the live chat continuation.

Reason:

- Current behavior is fresh session/no chat history.
- Task prompts must be self-contained.
- Prevents scheduled/background work from polluting live conversation context.

Status: Accepted.

## ADR-008: Supervisor Owns Group Process Lifecycle

Decision: In the agent-container runtime, host code does not spawn group processes directly. It asks the supervisor in the relevant agent container.

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

Decision: Tenant repositories can reference secrets, but cannot store secret values. Group-readable files should not contain shared secrets.

Reason:

- Business skill repositories may be shared or versioned.
- Group users should not read secrets for other groups or agent services.
- Credential proxy or scoped env injection is safer.

Status: Accepted.

## ADR-012: No World-writable Runtime IPC After User Isolation

Decision: After Version 1.6, group runtime directories must not rely on `0777` directories or `0666` files.

Reason:

- World-writable IPC defeats Linux user isolation.
- Per-group ownership plus supervisor group access is the intended security boundary inside an agent container.

Status: Accepted.

## ADR-013: Registered Groups Are Not the Same as Active Runs

Decision: Capacity planning and runtime status must distinguish configured groups from active runs.

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

## ADR-015: Tenant Skills Enter Runtime as Read-only Agent Service Inputs

Decision: Tenant-managed and agent-managed skills are resolved during deployment/config loading and mounted read-only into the agent service container. Group users can read these skills but cannot modify them.

Reason:

- Tenant is a management layer, not the runtime isolation unit.
- One agent service container can serve multiple groups that share the agent's configured skill set.
- Group user write access to tenant skill repositories would bypass review and make configuration drift hard to audit.

Status: Accepted.

## ADR-016: Generated Skills Are Group-local Until Promoted

Decision: Skills created or modified by a group at runtime are stored under that group's runtime directory and are not copied into tenant repositories automatically.

Reason:

- Runtime-generated content should not mutate deployment source of truth.
- Promotion to tenant skill should be explicit and reviewable.
- Group-local generated skills preserve isolation expectations.

Status: Accepted.

## ADR-017: Central Host DB Remains the Control-plane Source of Truth During Migration

Decision: Per-run runtime DBs are queues and run-local state. The existing host DB remains authoritative for chats, message history, scheduled tasks, task run logs, registered groups, router cursors, and legacy session IDs until a later explicit migration replaces it.

Reason:

- The host message loop currently uses `last_timestamp` and `last_agent_timestamp` to avoid duplicate replies and to retry failed runs.
- Scheduled tasks, registered groups, sender policy, card actions, and channel metadata are host-level control-plane data, not one-run runtime data.
- Treating runtime DBs as a second source of truth would create cursor drift and duplicate delivery risks.

Status: Accepted.

## ADR-018: Host-side Authorization Gates Stay Host-side

Decision: Sender allowlists, trigger rules, main-group privileges, approval allowlists, mount allowlists, group registration rights, and channel credential checks remain enforced by the host or supervisor before work is executed.

Reason:

- Group processes are not trusted to self-report authorization.
- Feishu and channel credentials are host-side capabilities.
- Main/self authorization semantics already exist in the file IPC bridge and must survive the tool IPC migration.

Status: Accepted.

## ADR-019: Runtime Code Is Immutable by Default

Decision: In the final agent-container runtime, supervisor, provider, MCP, and agent-runner platform code is image-owned and read-only. Group-created behavior moves to group-generated skills or group memory, not writable runner source.

Reason:

- NanoClaw 1.0 copies `container/agent-runner/src` into a per-group writable directory. Carrying that into a shared agent container would let one group mutate runtime code in ways that are hard to audit.
- Skills provide a smaller and reviewable customization boundary.
- Keeping platform code immutable makes provider and supervisor upgrades predictable.

Status: Accepted.

## ADR-020: Additional Mount Scope Must Be Explicit

Decision: Additional host mounts must declare whether they are agent-wide or group-specific. Group-specific mounts must not be promoted silently to agent-wide mounts in a one-container-per-agent runtime.

Reason:

- In NanoClaw 1.0, mounts are effectively per group because each group has its own Docker container.
- In the new runtime, an agent container can serve multiple groups, so an agent-wide mount becomes visible to every group user unless additional permissions prevent it.
- The existing external mount allowlist and non-main read-only policy are part of the security model.

Status: Accepted.
