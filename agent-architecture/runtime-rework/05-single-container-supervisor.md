# Version 1.5: Single Docker Runtime Supervisor

## Goal

Introduce one long-lived Docker runtime container that hosts a supervisor process. The host asks the supervisor to start, stop, inspect, and stream agent runs.

## Non-goals

- Do not enforce per-agent Linux users yet.
- Do not make this runtime the default yet.
- Do not remove the per-group Docker runtime.

## Architecture

```text
Host NanoClaw
  RuntimeClient
    -> Docker container: nanoclaw-runtime
         Supervisor
           -> agent-runner live process
           -> isolated task process
```

The host starts the runtime container once:

```bash
docker run -d \
  --name nanoclaw-runtime \
  -v <data>/runtime:/runtime \
  -v <tenant-repo>:/tenants:ro \
  nanoclaw-runtime:latest
```

Then all agent lifecycle operations go through the supervisor.

## Supervisor API

Use Unix socket if possible:

```text
/runtime/supervisor.sock
```

JSON-RPC messages:

```json
{
  "id": "req-1",
  "method": "runs.start",
  "params": {
    "tenantId": "acme",
    "agentId": "finance",
    "mode": "live",
    "runtimeDir": "/runtime/tenants/acme/agents/finance/live",
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
  tenantId: string;
  agentId: string;
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
src/runtime/single-container-users.ts
```

In 1.5 it may be named `single-container` because user isolation is not active yet.

Responsibilities:

- ensure the runtime Docker container is running
- connect to supervisor socket
- map `RuntimeDriver` calls to supervisor methods
- translate supervisor events to monitor events
- implement health checks through `runs.status`

## Process Start

Supervisor starts agent-runner:

```bash
node /app/agent-runner/dist/index.js \
  --tenant acme \
  --agent finance \
  --mode live \
  --runtime-dir /runtime/tenants/acme/agents/finance/live \
  --cwd /workspace/tenants/acme/agents/finance
```

In 1.6 this command is wrapped with `setpriv`.

## Initial Input

Initial prompt should be written by the host into `inbound.db` before `runs.start`, or passed in `runs.start` and written by supervisor. Prefer host writes DB directly to keep supervisor small.

## Logs

Per run:

```text
/runtime/logs/<tenant>/<agent>/<runId>.stdout.log
/runtime/logs/<tenant>/<agent>/<runId>.stderr.log
```

Supervisor streams summary events, not raw logs, unless explicitly requested.

## Tests

- runtime driver starts the runtime container if absent.
- supervisor starts a live run and returns status.
- supervisor stops a run.
- isolated task run exits and status becomes exited.
- stale runs are marked exited after supervisor restart.
- host no longer uses `docker inspect` for per-run health in this mode.

## Acceptance Criteria

- One Docker container can service multiple agent runs.
- Per-group `docker run` is not used in this runtime mode.
- Existing scheduler and queue call the same `RuntimeDriver` interface.
- The old `docker-per-group` runtime remains available.

