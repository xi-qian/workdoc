# Isolated 模式 IPC 目录隔离问题

## 日期
2026-05-20

## 问题概述

Isolated 模式的容器与群聊共享容器使用同一个 IPC 目录（`data/ipc/{group_folder}/`），导致消息串扰、`_close` sentinel 互相干扰、MCP 请求/响应文件冲突。

## 问题分析

### 现象

1. **Isolated 容器不退出**：agent 执行完 query 后，容器卡在 `waitForIpcMessage()` 等待 30 分钟 idle timeout
2. **Task 永远 active**：`runContainerAgent` 等容器 close 才 resolve，容器不退出则 task 不完成
3. **`_close` sentinel 串消费**：前一个容器的 `_close` 被后一个容器消费，或反之
4. **群聊消息误入 isolated 容器**：host 通过 IPC 写入的群聊消息被 isolated 容器读到

### 根因

`buildVolumeMounts`（`container-runner.ts:185-201`）为所有容器统一挂载 group 的 IPC 目录：

```
data/ipc/{group_folder}/ → /workspace/ipc
```

不区分 isolated 还是 group 模式。Host 侧的 IPC watcher（`ipc.ts`）也向同一个目录写入 MCP 请求的响应文件。

### 影响链

```
同一个 group 的两个 isolated task 并发或串行执行
  → 共享 data/ipc/{group_folder}/input/ 和 output/
  → 容器 A 的 MCP 请求响应被容器 B 读走
  → 容器 A 等不到响应卡住
  → 或容器 A 的 _close 被 host 写入后被容器 B 消费
  → 或群聊消息通过 IPC 写入后被 isolated 容器误读
```

## 解决方案

### 方案：为 isolated 容器创建独立临时 IPC 目录

**改动范围**：`container-runner.ts`、`task-scheduler.ts`

#### 1. `buildVolumeMounts` 接受 `isolatedIpcDir` 参数

```typescript
// container-runner.ts
function buildVolumeMounts(
  group: RegisteredGroup,
  isMain: boolean,
  isolatedIpcDir?: string,  // 新增
): VolumeMount[] {
  // ...
  const ipcDir = isolatedIpcDir || groupIpcDir;
  fs.mkdirSync(ipcDir, { recursive: true });
  mounts.push({
    hostPath: ipcDir,
    containerPath: '/workspace/ipc',
    readonly: false,
  });
  // ...
}
```

#### 2. `runContainerAgent` 接受 `isolated` 标志，创建临时目录

```typescript
// container-runner.ts
export async function runContainerAgent(
  group: RegisteredGroup,
  input: ContainerInput,
  onProcess: ...,
  onOutput?: ...,
  options?: { isolated?: boolean },  // 新增
): Promise<ContainerOutput> {
  // ...
  let isolatedIpcDir: string | undefined;
  if (options?.isolated) {
    isolatedIpcDir = path.join(DATA_DIR, 'sessions', group.folder, `isolated-ipc-${Date.now()}`);
    fs.mkdirSync(path.join(isolatedIpcDir, 'input'), { recursive: true });
    fs.mkdirSync(path.join(isolatedIpcDir, 'output'), { recursive: true });
    fs.mkdirSync(path.join(isolatedIpcDir, 'results'), { recursive: true });
  }
  const mounts = buildVolumeMounts(group, input.isMain, isolatedIpcDir);
  // ...
}
```

#### 3. `task-scheduler.ts` isolated path 传入 `isolated: true`

```typescript
// task-scheduler.ts（isolated path）
const output = await runContainerAgent(
  group,
  { prompt, groupFolder, chatJid, isMain, assistantName, singleShot: true },
  onProcess,
  onOutput,
  { isolated: true },  // 新增
);
```

#### 4. 容器退出后清理临时 IPC 目录

在 `runContainerAgent` 的 `container.on('close')` 回调中删除临时目录：

```typescript
container.on('close', (code) => {
  // ...existing cleanup...
  if (isolatedIpcDir) {
    fs.rmSync(isolatedIpcDir, { recursive: true, force: true });
  }
  resolve(...);
});
```

#### 5. Host 侧 IPC watcher 不扫描 isolated 临时目录

现有的 IPC watcher（`ipc.ts`）扫描 `data/ipc/{group_folder}/input/` 目录。临时目录路径为 `data/sessions/{group_folder}/isolated-ipc-{timestamp}/`，不会被扫描到，天然隔离。

但如果 isolated 容器需要使用 MCP 工具（如 `feishu_approval_comment`），它通过 IPC 写请求文件，host 需要能读到并响应。

**解决方案**：isolated 容器不走 IPC，MCP 工具通过 stdin/stdout 直接交互。但这改动太大。

**折中方案**：isolated 容器的临时 IPC 目录仍然被 host 的 IPC watcher 监控，但 watcher 需要能发现这些临时目录。在 `ipc.ts` 中，除了扫描固定 IPC 目录外，也扫描 `data/sessions/{group_folder}/isolated-ipc-*/input/`。

### 补充：singleShot 修复

之前已做的 `singleShot` 修复仍然需要保留。即使 IPC 隔离了，isolated 容器执行完 query 后也应该立即退出，而不是等 IPC 消息。

## 验证方法

1. 同时触发同一 group 的两个 isolated webhook task
2. 确认两个容器使用不同的 IPC 目录（`docker inspect` 查看挂载）
3. 确认两个容器互不干扰，各自正常完成
4. 确认 task 状态正确标记为 `completed`
5. 确认临时 IPC 目录在容器退出后被清理
