# Version 1.5: Agent Service Container Supervisor

## Goal

Introduce one long-lived Docker container per agent service. Each agent container hosts a supervisor process. The host asks the relevant agent supervisor to start, stop, inspect, and stream group runs.

This replaces "one Docker container per group" with "one Docker container per agent service, many group runs inside it".

## Non-goals

- Do not enforce per-group Linux users yet.
- Do not make this runtime the default yet.
- Do not remove the per-group Docker runtime.
- Do not collapse multiple agent services into one Docker container.

## Architecture

```text
Host NanoClaw
  RuntimeClient
    -> Docker service/container: nanoclaw-agent-finance
         Supervisor
           -> group live process: feishu-main
           -> group isolated task process: feishu-main/task-123
           -> group live process: another-group
    -> Docker service/container: nanoclaw-agent-ops
         Supervisor
           -> group runs for the ops agent service
```

The host starts one runtime container for each configured agent service:

```bash
docker run -d \
  --name nanoclaw-agent-finance \
  -v <data>/runtime/agents/finance:/runtime \
  -v <tenant-repo>:/tenants:ro \
  nanoclaw-agent-runtime:latest
```

Tenant remains a configuration and skill-management layer. It can own multiple agent services, but runtime Docker ownership is per agent service.

## Supervisor API

Use Unix socket if possible:

```text
/runtime/supervisor/supervisor.sock
```

JSON-RPC messages:

```json
{
  "id": "req-1",
  "method": "runs.start",
  "params": {
    "tenantId": "acme",
    "agentId": "finance",
    "groupId": "feishu-main",
    "mode": "live",
    "runtimeDir": "/runtime/groups/feishu-main/live",
    "cwd": "/workspace/tenants/acme/agents/finance"
  }
}
```

Methods:

```text
runs.start
runs.stop
runs.close
runs.status
runs.input
runs.list
events.subscribe
users.ensure     # added in 1.6, stubbed here
```

## Run Registry

Supervisor keeps an in-memory registry:

```ts
interface RunRecord {
  runId: string;
  tenantId?: string;
  agentId: string;
  groupId: string;
  mode: "live" | "isolated-task";
  runtimeDir: string;
  cwd: string;
  pid: number;
  state: "starting" | "running" | "idle" | "stopping" | "exited";
  startedAt: string;
  exitedAt?: string;
  exitCode?: number;
}
```

State should also be persisted to:

```text
/runtime/supervisor/runs.json
```

This lets the supervisor recover enough metadata after restart to mark stale runs exited.

## Host Runtime Driver

Add:

```text
src/runtime/agent-container.ts
```

In 1.6 this becomes or is extended by:

```text
src/runtime/agent-container-users.ts
```

Responsibilities:

- ensure the Docker service/container for the target agent is running
- connect to that agent container's supervisor socket
- map `RuntimeDriver` calls to supervisor methods
- translate supervisor events to monitor events
- implement health checks through `runs.status`

## Process Start

Supervisor starts agent-runner:

```bash
node /app/agent-runner/dist/index.js \
  --tenant acme \
  --agent finance \
  --group feishu-main \
  --mode live \
  --runtime-dir /runtime/groups/feishu-main/live \
  --cwd /workspace/tenants/acme/agents/finance
```

In 1.6 this command is wrapped with `setpriv` for the group user.

## Initial Input

Initial prompt should be written by the host into the group's `inbound.db` before `runs.start`, or passed in `runs.start` and written by supervisor. Prefer host writes DB directly to keep supervisor small.

## Logs

Per run:

```text
/runtime/logs/<group>/<runId>.stdout.log
/runtime/logs/<group>/<runId>.stderr.log
```

Supervisor streams summary events, not raw logs, unless explicitly requested.

## Tests

- runtime driver starts the agent service container if absent.
- supervisor starts a group live run and returns status.
- supervisor stops a run.
- isolated task run exits and status becomes exited.
- stale runs are marked exited after supervisor restart.
- host no longer uses `docker inspect` for per-run health in this mode.
- two agent services use two Docker containers/services.

## Acceptance Criteria

- One Docker container can service multiple group runs for one agent service.
- Multiple agent services run as multiple Docker services/containers.
- Per-group `docker run` is not used in this runtime mode.
- Existing scheduler and queue call the same `RuntimeDriver` interface.
- The old `docker-per-group` runtime remains available.

