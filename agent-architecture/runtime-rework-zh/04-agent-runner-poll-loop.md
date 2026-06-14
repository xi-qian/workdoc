# 版本 1.4：Agent-runner 轮询循环

## 目标

使 agent-runner 能够以热进程方式运行，轮询 DB 后端的入站消息，调用选定的 Provider，写出站消息，并在空闲时保持运行直到被停止。

## 非目标

- 尚不要求智能体容器运行时。
- 尚不要求 OpenCode。
- 不移除一次性模式。
- 不移除旧版文件工具 IPC。

## 模式

Agent-runner 支持三种模式：

```text
legacy-single-shot
live
isolated-task
```

`legacy-single-shot` 保持当前的 stdin/stdout 行为。

`live` 运行直到空闲超时或收到关闭信号。

`isolated-task` 使用全新的运行时目录处理一个任务提示，写入结果后退出。

## CLI 与环境变量

首选 CLI：

```bash
agent-runner \
  --tenant acme \
  --agent finance \
  --group feishu-main \
  --mode live \
  --runtime-dir /runtime/groups/feishu-main/live \
  --cwd /workspace/tenants/acme/agents/finance
```

环境变量回退：

```env
NANOCLAW_TENANT=acme
NANOCLAW_AGENT=finance
NANOCLAW_GROUP=feishu-main
NANOCLAW_RUN_MODE=live
NANOCLAW_RUNTIME_DIR=/runtime/groups/feishu-main/live
NANOCLAW_CWD=/workspace/tenants/acme/agents/finance
```

## 轮询循环

伪代码：

```ts
while (!stopping) {
  // 获取待处理的入站消息批次
  const batch = claimPendingInbound(runtimeDir, maxMessages);
  if (batch.length === 0) {
    await sleep(pollInterval);
    continue;
  }

  // 格式化消息并发起 Provider 查询
  const prompt = formatMessages(batch);
  const continuation = state.get(`continuation:${providerName}`);

  const query = provider.query({
    prompt,
    continuation,
    cwd,
    systemContext
  });

  // 逐个处理 Provider 事件
  for await (const event of query.events) {
    if (event.type === "init") state.set(`continuation:${providerName}`, event.continuation);
    if (event.type === "result" && event.text) writeOutbound(event.text);
    if (event.type === "error") writeOutboundError(event);
  }

  // 标记入站消息已处理
  markInboundCompleted(batch);
}
```

## 后续消息

当 Provider 查询处于活跃状态时，循环应持续检查新到达的待处理入站消息。

初始行为：

- Claude Provider：调用 `query.push(message)`。
- 不支持 push 语义的 Provider：将后续消息标记为待处理，等当前查询完成后由下一轮循环处理。

后续行为：

- OpenCode Provider 可能在内部中止并重启。

## 空闲与停止

Runner 在以下条件退出：

- 模式为 `isolated-task` 且第一批次处理完成
- 模式为 `live` 且空闲超时到期
- 监督进程发送停止信号
- 通过运行时 DB 状态接收到关闭标记

关闭应在 DB 中表示：

```sql
INSERT OR REPLACE INTO kv(key, value, updated_at)
VALUES ('control:stop', 'true', datetime('now'));
```

旧版 `_close` 文件仍可桥接到此状态键。

## 隔离任务语义

隔离任务：

- 使用相同的租户管理配置、智能体服务和组身份
- 以同一个未来的 Linux 用户运行
- 使用唯一的运行目录
- 不读取实时聊天历史
- 不复用实时续接
- 结果返回或出错后退出

目录结构：

```text
runs/<taskId>-<timestamp>/
  inbound.db
  outbound.db
  state.db
```

## Provider 状态

续接按 Provider 分别存储：

```text
continuation:claude
continuation:opencode
```

这可防止 Provider 切换时读取不兼容的会话 ID。

## 输出格式化

在兼容阶段，agent-runner 可同时写入：

- `outbound.db`
- stdout 哨兵 `OUTPUT_START/OUTPUT_END`

当 `NANOCLAW_RUNTIME_DIR` 存在时，宿主应优先使用 `outbound.db`。

## 测试

- live 模式处理入站消息并在空闲时保持运行。
- isolated-task 模式精确处理一次运行后退出。
- Provider 续接按 Provider 分别保存。
- 活跃查询期间后续消息不会丢失。
- 关闭控制可停止 live 运行中的 Runner。
- 旧版一次性模式仍通过已有测试。

## 验收标准

- Agent-runner 可在无 stdin 提示输入的情况下运行。
- 宿主可在进程启动后入队一条消息并收到出站响应。
- 隔离任务不修改实时续接。
- 现有 Docker 按组运行时可启动旧版模式或轮询循环模式。
