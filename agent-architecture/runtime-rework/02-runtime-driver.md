# Version 1.2: Runtime Driver Abstraction

## Goal

Introduce a runtime boundary so the current per-group Docker implementation can coexist with future single-container/user-isolated runtime. Behavior must remain unchanged by default.

## Non-goals

- Do not replace Docker per group yet.
- Do not change IPC format yet.
- Do not introduce supervisor.
- Do not change task semantics.

## Problem

Runtime behavior is currently embedded across:

- `src/container-runner.ts`
- `src/container-runtime.ts`
- `src/group-queue.ts`
- `src/task-scheduler.ts`

The code constructs Docker args, spawns Docker, parses stdout sentinels, checks `docker inspect`, and writes close sentinels directly. Future runtime modes need the same logical operations without Docker-specific calls in scheduler and queue code.

## Runtime Interface

Add:

```ts
export interface RuntimeDriver {
  readonly name: string;

  ensureReady(): Promise<void>;
  cleanupOrphans(): Promise<void>;

  startRun(input: RuntimeRunInput): Promise<RuntimeRunHandle>;
  sendInput(runId: string, message: string): Promise<boolean>;
  closeRun(runId: string): Promise<void>;
  stopRun(runId: string): Promise<void>;
  getRunStatus(runId: string): Promise<RuntimeRunStatus>;
}
```

Supporting types:

```ts
export type RuntimeRunMode = "live" | "isolated-task" | "single-shot";

export interface RuntimeRunInput {
  tenantId?: string;
  agentId?: string;
  groupFolder: string;
  chatJid: string;
  prompt?: string;
  isMain: boolean;
  assistantName: string;
  mode: RuntimeRunMode;
  legacyContainerInput: ContainerInput;
  isolated?: boolean;
}

export interface RuntimeRunHandle {
  runId: string;
  process: ChildProcess;
  displayName: string;
  output: Promise<ContainerOutput>;
}

export interface RuntimeRunStatus {
  runId: string;
  state: "starting" | "running" | "idle" | "stopping" | "exited" | "unknown";
  pid?: number;
  startedAt?: string;
  exitedAt?: string;
}
```

In this version, the driver can still expose a `ChildProcess` because the existing queue tracks processes. Later supervisor-backed runtime can use a lightweight process-like handle or adapter.

## Driver Registry

Add:

```ts
export function createRuntimeDriver(name: string): RuntimeDriver;
```

Config:

```env
NANOCLAW_RUNTIME_DRIVER=docker-per-group
```

Default remains `docker-per-group`.

## DockerPerGroupRuntimeDriver

Move current behavior into:

```text
src/runtime/docker-per-group.ts
```

Responsibilities:

- build volume mounts
- build Docker args
- spawn `docker run`
- send initial input over stdin
- parse stdout sentinel events
- implement `docker inspect` health checks
- implement `docker stop`
- cleanup labeled orphan containers

Existing `container-runner.ts` becomes either:

- a compatibility wrapper around `RuntimeDriver`, or
- the implementation file for `DockerPerGroupRuntimeDriver` during initial refactor.

## GroupQueue Changes

Replace direct Docker checks:

```ts
private async isContainerAlive(containerName: string): Promise<boolean>
```

with:

```ts
private async isRunAlive(runId: string): Promise<boolean> {
  const status = await this.runtime.getRunStatus(runId);
  return status.state === "running" || status.state === "idle";
}
```

State fields should start transitioning:

```ts
containerName -> runId/displayName
```

Keep `containerName` as a display alias for monitor compatibility if needed.

## TaskScheduler Changes

Replace direct `runContainerAgent(...)` dependency with:

```ts
runtime.startRun({
  mode: "isolated-task",
  isolated: true,
  legacyContainerInput: { singleShot: true, ... }
})
```

The scheduler should not know whether the runtime is Docker, supervisor, or local.

## Monitoring Compatibility

Existing monitor events can remain named `containerStarted/containerStopped` for this version, but payloads should include:

```ts
{
  runtimeDriver: "docker-per-group",
  runId,
  displayName
}
```

Later UI copy can change from container to run.

## Tests

- `DockerPerGroupRuntimeDriver` builds the same Docker args as before.
- `GroupQueue` uses runtime status instead of `docker inspect`.
- isolated task still passes `{ isolated: true }` through the runtime input.
- orphan cleanup still only targets this instance.
- default env config selects `docker-per-group`.

## Acceptance Criteria

- Existing deployment behavior is unchanged.
- No scheduler/router code imports Docker-specific helpers.
- A fake runtime driver can be used in tests.
- All current tests pass with the default driver.

