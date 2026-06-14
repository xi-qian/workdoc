# 版本 1.2：运行时 Driver 抽象层

## 目标

引入运行时边界，使当前的按组 Docker 实现能与未来的智能体容器/按组用户隔离运行时共存。默认行为必须保持不变。

## 非目标

- 暂不替换按组 Docker。
- 暂不更改 IPC 格式。
- 暂不引入监督进程。
- 暂不更改任务语义。

## 问题

运行时行为目前散布在以下文件中：

- `src/container-runner.ts`
- `src/container-runtime.ts`
- `src/group-queue.ts`
- `src/task-scheduler.ts`

代码直接构造 Docker 参数、启动 Docker 进程、解析 stdout 哨兵标记、执行 `docker inspect` 检查、写入关闭哨兵。未来的运行时模式需要相同的逻辑操作，但不应在调度器和队列代码中出现 Docker 特定调用。

## 运行时接口

新增：

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

配套类型：

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

在本版本中，Driver 仍可暴露 `ChildProcess`，因为现有队列依赖进程跟踪。后续基于监督进程的运行时可使用轻量级类进程句柄或适配器。

## Driver 注册表

新增：

```ts
export function createRuntimeDriver(name: string): RuntimeDriver;
```

配置：

```env
NANOCLAW_RUNTIME_DRIVER=docker-per-group
```

默认值保持 `docker-per-group`。

## DockerPerGroupRuntimeDriver

将当前行为迁移至：

```text
src/runtime/docker-per-group.ts
```

职责：

- 构建卷挂载
- 构建 Docker 参数
- 启动 `docker run`
- 通过 stdin 发送初始输入
- 解析 stdout 哨兵事件
- 实现 `docker inspect` 健康检查
- 实现 `docker stop`
- 清理带标记的孤儿容器

现有 `container-runner.ts` 可改为：

- `RuntimeDriver` 的兼容包装层，或
- 初始重构期间 `DockerPerGroupRuntimeDriver` 的实现文件。

## GroupQueue 变更

将直接的 Docker 检查：

```ts
private async isContainerAlive(containerName: string): Promise<boolean>
```

替换为：

```ts
private async isRunAlive(runId: string): Promise<boolean> {
  const status = await this.runtime.getRunStatus(runId);
  return status.state === "running" || status.state === "idle";
}
```

状态字段应开始迁移：

```ts
containerName -> runId/displayName
```

如需监控兼容性，保留 `containerName` 作为显示别名。

## TaskScheduler 变更

将直接的 `runContainerAgent(...)` 依赖替换为：

```ts
runtime.startRun({
  mode: "isolated-task",
  isolated: true,
  legacyContainerInput: { singleShot: true, ... }
})
```

调度器不应感知运行时是 Docker、监督进程还是本地进程。

## 监控兼容性

现有监控事件在本版本中可保持 `containerStarted/containerStopped` 命名，但载荷应包含：

```ts
{
  runtimeDriver: "docker-per-group",
  runId,
  displayName
}
```

后续 UI 层面可从 container 改为 run。

## 测试

- `DockerPerGroupRuntimeDriver` 构建的 Docker 参数与之前一致。
- `GroupQueue` 使用运行时状态而非 `docker inspect`。
- 隔离任务仍通过运行时输入传递 `{ isolated: true }`。
- 孤儿清理仍仅针对当前实例。
- 默认环境配置选择 `docker-per-group`。

## 验收标准

- 现有部署行为不变。
- 调度器/路由器代码不导入 Docker 特定的辅助函数。
- 测试中可使用伪造的 runtime driver。
- 所有现有测试在默认 driver 下通过。
