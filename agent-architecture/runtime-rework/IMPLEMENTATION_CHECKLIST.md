# Runtime Rework Implementation Checklist

This checklist turns the design documents into concrete coding tasks. It is intended for future agents that do not have the original planning conversation.

Work one version at a time. Do not start the next version until the current version's default behavior, compatibility checks, and tests pass.

## Global Invariants

These must remain true throughout the migration:

- Existing deployments continue to run with the default `docker-per-group` runtime until Version 2.0.
- Existing `groups/` configuration remains supported until an explicit migration removes it.
- Existing Feishu/business tools continue to work during compatibility phases.
- Existing scheduled task semantics remain:
  - `context_mode: "group"` uses group chat context.
  - `context_mode: "isolated"` uses a fresh single-shot run with no chat history.
- isolated task is context/lifecycle isolation, not a separate tenant security boundary.
- Secrets must not be written to tenant Git repositories.
- No runtime directory should rely on `0777` or `0666` after Version 1.6.
- Claude provider remains available until OpenCode has production parity for the deployment.
- Rollback path must exist at every phase.

## Version 1.1: Tenant, Agent, and Skill Boundary

Design doc: [01-tenant-agent-skill-boundary.md](./01-tenant-agent-skill-boundary.md)

### Code Tasks

- Add tenant types:
  - `src/tenants/types.ts`
  - `RegisteredTenant`
  - `RegisteredAgent`
  - `ResolvedSkill`
  - `AgentLimits`
  - `AgentMount`
- Add schemas:
  - `schemas/tenant.schema.json`
  - `schemas/agent.schema.json`
  - `schemas/skill-manifest.schema.json`
- Add loader:
  - `src/tenants/loader.ts`
  - Reads `NANOCLAW_TENANTS_DIR`
  - Loads `tenants/*/tenant.json`
  - Loads `tenants/*/agents/*/agent.json`
  - Resolves `builtin:`, `tenant:`, and `agent:` skill references
  - Rejects duplicate IDs
  - Produces diagnostics for missing references
- Add legacy adapter:
  - maps existing `groups/<folder>/CLAUDE.md` to synthetic tenant `legacy`
  - does not change existing group behavior
- Add setup/config support:
  - document `NANOCLAW_TENANTS_DIR`
  - print loaded tenants/agents at startup
- Add migration script:
  - dry-run by default
  - source: existing `groups/`
  - target: external tenant repo

### Tests

- Valid tenant repo loads.
- Duplicate tenant ID is rejected.
- Duplicate agent ID inside tenant is rejected.
- Missing skill reference yields diagnostic.
- Secret-looking values in JSON are rejected or warned.
- Legacy groups still load.

### Manual Verification

```bash
npm test
NANOCLAW_TENANTS_DIR=/path/to/tenants npm run dev
```

Confirm startup logs show both legacy groups and tenant agents.

### Do Not Change

- `src/container-runner.ts` behavior
- `src/group-queue.ts` behavior
- `src/task-scheduler.ts` task execution behavior

## Version 1.2: Runtime Driver Abstraction

Design doc: [02-runtime-driver.md](./02-runtime-driver.md)

### Code Tasks

- Add runtime types:
  - `src/runtime/types.ts`
  - `RuntimeDriver`
  - `RuntimeRunInput`
  - `RuntimeRunHandle`
  - `RuntimeRunStatus`
- Add runtime registry:
  - `src/runtime/index.ts`
  - `createRuntimeDriver(name)`
- Move current Docker behavior into:
  - `src/runtime/docker-per-group.ts`
- Keep compatibility export:
  - `runContainerAgent(...)` may remain as wrapper initially
- Replace direct runtime usage:
  - `src/group-queue.ts` uses `runtime.getRunStatus()`
  - `src/task-scheduler.ts` uses `runtime.startRun()`
  - `src/container-runtime.ts` becomes Docker driver internals or compatibility only
- Add config:
  - `NANOCLAW_RUNTIME_DRIVER=docker-per-group`
  - default is `docker-per-group`

### Tests

- Docker args are unchanged for equivalent input.
- `GroupQueue` no longer shells out to `docker inspect` directly.
- isolated task still passes `singleShot: true` and `isolated: true`.
- fake runtime driver can drive queue tests.
- orphan cleanup still filters by instance ID.

### Manual Verification

```bash
npm test
NANOCLAW_RUNTIME_DRIVER=docker-per-group npm run dev
```

Send one normal message and one isolated scheduled task. Behavior should match pre-refactor.

### Do Not Change

- Docker image
- IPC format
- agent-runner entrypoint
- provider behavior

## Version 1.3: DB-backed IPC

Design doc: [03-db-backed-ipc.md](./03-db-backed-ipc.md)

### Code Tasks

- Add host IPC modules:
  - `src/runtime-ipc/schema.ts`
  - `src/runtime-ipc/inbound.ts`
  - `src/runtime-ipc/outbound.ts`
  - `src/runtime-ipc/state.ts`
  - `src/runtime-ipc/paths.ts`
- Add agent-runner IPC modules:
  - `container/agent-runner/src/runtime-ipc/schema.ts`
  - `container/agent-runner/src/runtime-ipc/inbound.ts`
  - `container/agent-runner/src/runtime-ipc/outbound.ts`
  - `container/agent-runner/src/runtime-ipc/state.ts`
- Create DBs:
  - `inbound.db`
  - `outbound.db`
  - `state.db`
- Enable WAL and busy timeout.
- Host writes inbound records for normal messages.
- Agent writes outbound records for provider results.
- Host polls outbound and delivers.
- Keep stdout sentinel during compatibility.
- Keep legacy file IPC directories under a `legacy-ipc` path.

### Tests

- Schema initialization is idempotent.
- Concurrent host/agent access does not fail under normal load.
- Agent can claim a message exactly once.
- Host can mark outbound delivered.
- isolated task DB path is separate from live DB path.
- legacy file IPC path is still created.

### Manual Verification

```bash
npm test
```

Run a normal chat and confirm:

- inbound row becomes completed
- outbound row becomes delivered
- existing stdout response still appears in logs during compatibility

### Do Not Change

- Tool file IPC behavior yet
- single-shot fallback
- runtime driver default

## Version 1.4: Agent-runner Poll Loop

Design doc: [04-agent-runner-poll-loop.md](./04-agent-runner-poll-loop.md)

### Code Tasks

- Add agent-runner mode parsing:
  - `legacy-single-shot`
  - `live`
  - `isolated-task`
- Add CLI/env support:
  - `--runtime-dir`
  - `--tenant`
  - `--agent`
  - `--mode`
  - `--cwd`
- Implement live poll loop.
- Implement isolated-task single-batch loop.
- Store provider continuation in `state.db`.
- Add close control via `state.db` key `control:stop`.
- Bridge old `_close` file to `control:stop` during compatibility.
- Prefer `outbound.db` for results if runtime dir is configured.
- Keep legacy stdin/stdout mode for old runtime.

### Tests

- live mode processes a DB inbound message after startup.
- isolated-task mode exits after first result.
- continuation is saved per provider key.
- close control stops live process.
- follow-up message is processed or queued safely during active query.
- legacy single-shot tests still pass.

### Manual Verification

Start agent-runner in live mode with a temp runtime dir. Insert inbound row manually or via helper. Confirm outbound row appears.

### Do Not Change

- runtime container topology
- Linux users
- provider selection semantics

## Version 1.5: Single Docker Runtime Supervisor

Design doc: [05-single-container-supervisor.md](./05-single-container-supervisor.md)

### Code Tasks

- Add runtime container image target or update Dockerfile:
  - includes supervisor
  - includes agent-runner
  - includes runtime dependencies
- Add supervisor:
  - `container/supervisor/src/index.ts` or equivalent
  - Unix socket JSON-RPC preferred
- Implement supervisor methods:
  - `runs.start`
  - `runs.stop`
  - `runs.close`
  - `runs.status`
  - `runs.list`
  - `events.subscribe`
- Add host runtime driver:
  - `src/runtime/single-container.ts`
  - starts long-lived Docker runtime container
  - connects to supervisor
  - maps `RuntimeDriver` methods
- Add run logs:
  - stdout/stderr per run
- Add supervisor stale-run recovery.

### Tests

- supervisor starts an agent process.
- supervisor reports running/exited status.
- runtime driver starts runtime container if absent.
- `runs.stop` terminates process.
- isolated task exits and status is recorded.
- old `docker-per-group` driver still passes tests.

### Manual Verification

```bash
NANOCLAW_RUNTIME_DRIVER=single-container npm run dev
```

Verify one runtime Docker container exists and no per-group container is created for a message.

### Do Not Change

- process user remains default container user until 1.6
- DB IPC remains compatible with old runtime

## Version 1.6: Container User Isolation

Design doc: [06-container-user-isolation.md](./06-container-user-isolation.md)

### Code Tasks

- Add supervisor `users.ensure`.
- Add username sanitizer and collision handling.
- Add directory preparer:
  - creates runtime dirs
  - chowns to agent user
  - sets group to supervisor group
  - sets mode `0770` or stricter
- Start agent-runner via:
  - `setpriv`, `gosu`, or `su-exec`
  - `--no-new-privs`
- Remove world-writable runtime paths.
- Ensure isolated task uses same user as live agent.
- Add optional per-agent limits config, even if enforcement is initially partial.
- Ensure secrets are not in shared files or global env.

### Tests

- agent user can read/write own runtime DB.
- agent user cannot read another agent runtime DB.
- agent user cannot write tenant shared skill path.
- supervisor can stop and inspect all runs.
- isolated task uses same UID as live agent.
- no runtime directory is `0777` or file `0666`.

### Manual Verification

Inside runtime container:

```bash
id nc_acme_finance
sudo -u nc_acme_finance test -r /runtime/tenants/acme/agents/finance/live/state.db
sudo -u nc_acme_finance test ! -r /runtime/tenants/acme/agents/ops/live/state.db
```

### Do Not Change

- provider default
- tenant config format except adding optional limits

## Version 1.7: Provider Abstraction and OpenCode

Design doc: [07-opencode-provider.md](./07-opencode-provider.md)

### Code Tasks

- Add provider interface in agent-runner.
- Move Claude-specific code behind `ClaudeProvider`.
- Add provider registry.
- Add `MockProvider` for tests.
- Add `OpenCodeProvider`.
- Add OpenCode dependency and Docker image install step.
- Add MCP config translation.
- Store continuation per provider in `state.db`.
- Add provider config fields to `agent.json`.

### Tests

- registry loads Claude, OpenCode, mock.
- Claude behavior remains compatible.
- OpenCode emits init/activity/result on simple prompt.
- OpenCode MCP translation is correct.
- provider switch does not reuse incompatible continuation.
- isolated task runs with OpenCode.

### Manual Verification

Run two agents:

- one with `provider: claude`
- one with `provider: opencode`

Send simple messages to both and confirm separate continuations.

### Do Not Change

- OpenCode should not become default until operationally validated.
- Do not remove Claude.

## Version 1.8: Tool IPC Migration

Design doc: [08-tool-ipc-migration.md](./08-tool-ipc-migration.md)

### Code Tasks

- Add `tools.db` schema.
- Add agent-side tool request client.
- Add host-side tool worker.
- Migrate tools in priority order:
  - `send_message`
  - task scheduling tools
  - Feishu read-only
  - Feishu write
  - downloads/uploads
  - approvals
- Add timeouts for every tool request.
- Add sanitized audit fields.
- Keep legacy file watcher until tenant skills are migrated.
- Add config flag to disable legacy file IPC per agent.

### Tests

- DB tool request completes successfully.
- timeout is recorded.
- host-side error is returned to agent.
- two isolated tasks cannot consume each other's results.
- legacy file request still works.
- migrated agent can disable file IPC.

### Manual Verification

Run a migrated `send_message` and a Feishu read-only tool. Confirm:

- request row is completed
- no secret is stored in request/result
- outbound message is delivered

## Version 2.0: Final Architecture

Design doc: [20-final-architecture.md](./20-final-architecture.md)

### Code Tasks

- Set default runtime driver to `single-container-users`.
- Keep `docker-per-group` fallback.
- Update setup and deployment docs.
- Add migration command for tenants and runtime data.
- Add operational status command:
  - runtime container status
  - supervisor status
  - active runs
  - per-agent users
  - DB queue depth
- Add cleanup commands:
  - stale runs
  - old isolated task dirs
  - old logs
- Disable legacy file IPC for migrated tenants by default.

### Tests

- full e2e normal message
- full e2e isolated task
- cross-agent permission denial
- runtime restart recovery
- provider switch safety
- rollback to `docker-per-group`

### Manual Verification

Deploy fresh environment with default settings. Confirm:

- only one runtime Docker container is started
- agents run as different Linux users
- normal chat works
- isolated task works
- old runtime can be selected via env var for rollback

