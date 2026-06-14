# 版本 1.5：智能体服务容器监督进程

## 目标

为每个智能体服务引入一个长驻 Docker 容器。每个智能体容器承载一个监督进程。宿主机通过相应的智能体监督进程来启动、停止、检查和流式传输组运行。

这用"每个智能体服务一个 Docker 容器，内部运行多个组运行"替代"每个组一个 Docker 容器"。

## 非目标

- 尚不强制执行每组的 Linux 用户隔离。
- 尚不将此运行时设为默认。
- 尚不移除每组的 Docker 运行时。
- 尚不将多个智能体服务合并到一个 Docker 容器中。

## 架构

```text
宿主机 NanoClaw
  RuntimeClient
    -> Docker 服务/容器: nanoclaw-agent-finance
         Supervisor
           -> 组实时进程: feishu-main
           -> 组隔离任务进程: feishu-main/task-123
           -> 组实时进程: another-group
    -> Docker 服务/容器: nanoclaw-agent-ops
         Supervisor
           -> ops 智能体服务的组运行
```

宿主机为每个已配置的智能体服务启动一个运行时容器：

```bash
docker run -d \
  --name nanoclaw-agent-finance \
  -v <data>/runtime/agents/finance:/runtime \
  -v <tenant-repo>:/tenants:ro \
  nanoclaw-agent-runtime:latest
```

租户仍然是配置和技能管理层。一个租户可以拥有多个智能体服务，但运行时 Docker 的所有权归属于智能体服务。

## 监督进程 API

尽可能使用 Unix socket：

```text
/runtime/supervisor/supervisor.sock
```

JSON-RPC 消息：

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

方法列表：

```text
runs.start
runs.stop
runs.close
runs.status
runs.input
runs.list
events.subscribe
users.ensure     # 在 1.6 中添加，此处为占位
```

## 运行注册表

监督进程维护一个内存注册表：

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

状态也应持久化到：

```text
/runtime/supervisor/runs.json
```

这使监督进程在重启后能恢复足够的元数据，将过期运行标记为已退出。

## 宿主机运行时驱动

新增：

```text
src/runtime/agent-container.ts
```

在 1.6 中变为或扩展为：

```text
src/runtime/agent-container-users.ts
```

职责：

- 确保目标智能体的 Docker 服务/容器正在运行
- 连接到该智能体容器的监督进程 socket
- 将 `RuntimeDriver` 调用映射为监督进程方法
- 将监督进程事件转换为监控事件
- 通过 `runs.status` 实现健康检查

## 进程启动

监督进程启动 agent-runner：

```bash
node /app/agent-runner/dist/index.js \
  --tenant acme \
  --agent finance \
  --group feishu-main \
  --mode live \
  --runtime-dir /runtime/groups/feishu-main/live \
  --cwd /workspace/tenants/acme/agents/finance
```

在 1.6 中此命令将用 `setpriv` 包装，以设置组用户权限。

## 初始输入

初始提示应由宿主机在调用 `runs.start` 之前写入组的 `inbound.db`，或在 `runs.start` 中传入并由监督进程写入。优先让宿主机直接写 DB，以保持监督进程轻量。

## 日志

每个运行：

```text
/runtime/logs/<group>/<runId>.stdout.log
/runtime/logs/<group>/<runId>.stderr.log
```

监督进程默认发送摘要事件而非原始日志，除非显式请求。

## 测试

- 运行时驱动在智能体服务容器不存在时启动该容器。
- 监督进程启动一个组实时运行并返回状态。
- 监督进程停止一个运行。
- 隔离任务运行退出后状态变为已退出。
- 监督进程重启后，过期运行被标记为已退出。
- 宿主机在此模式下不再使用 `docker inspect` 进行单运行健康检查。
- 两个智能体服务使用两个 Docker 容器/服务。

## 验收标准

- 一个 Docker 容器可服务一个智能体服务的多个组运行。
- 多个智能体服务作为多个 Docker 服务/容器运行。
- 此运行时模式不使用每组的 `docker run`。
- 现有调度器和队列调用相同的 `RuntimeDriver` 接口。
- 旧的 `docker-per-group` 运行时仍然可用。
