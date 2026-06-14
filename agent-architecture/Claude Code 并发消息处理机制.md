# Claude Code 并发消息处理机制

对比分析 Claude Code 在执行 Task 过程中收到新消息时的处理方式与 Hermes Agent 三模式（interrupt / queue / steer）系统的异同。

---

## 总览：架构理念

| 维度 | Hermes Agent | Claude Code |
|---|---|---|
| **设计** | 三种显式模式，用户可配置 | 统一行为，由工具能力隐式决定 |
| **模式选择** | 用户配置 `busy_input_mode` 或环境变量 | 隐式 — 取决于当前执行工具的 `interruptBehavior` |
| **状态守卫** | 异步 session 锁（`asyncio.Event`） | 同步 `QueryGuard` 状态机（idle → dispatching → running） |
| **取消机制** | `agent.interrupt()` + 线程信号 | `AbortController.abort('interrupt')` 经 `AbortSignal` 链传播 |
| **消息队列** | 单字符串槽 + `/queue` FIFO 溢出表 | 优先级队列（now > next > later），按 Agent 隔离，作为 turn attachment 消费 |
| **中途注入** | `steer()` 追加到 `_pending_steer`，下个工具轮次消费 | 无等价机制 — 消息始终等待 turn 边界 |

---

## 1. 状态守卫：QueryGuard vs Session 锁

### Hermes：异步 Session 锁

```python
# base.py — _active_sessions: Dict[str, asyncio.Event]
# 每个 session 一个 Event 作为互斥锁，handle_message 检查之
if session_key in self._active_sessions:
    # Session 忙碌 → 路由到 _handle_active_session_busy_message
else:
    # 空闲 → 设置 Event + 启动后台任务
```

- 异步原语 — 依赖 `asyncio.Event` 实现互斥
- 需要 `_heal_stale_session_lock()` 检测 split-brain（任务已死但锁残留）
- 使用 `_AGENT_PENDING_SENTINEL` 哨兵防止异步间隙竞态

### Claude Code：同步 QueryGuard

```typescript
// QueryGuard.ts — 三态状态机
idle → dispatching (reserve) → running (tryStart) → idle (end)

tryStart(): number | null {
  if (this._status === 'running') return null  // 并发守卫
  this._status = 'running'
  ++this._generation
  return this._generation
}
```

- **同步** — 无异步间隙，无需哨兵。`reserve()` 和 `tryStart()` 是原子的
- **代际追踪** — `end(generation)` 在新 query 启动后返回 `false`，防止过期的 finally 回调污染新 session
- **forceEnd()** — 递增 generation 使所有在途回调失效
- **React 集成** — `useSyncExternalStore` 实现响应式 UI（loading spinner、禁用输入框）

**关键优势：** 同步设计消除了 Hermes 用 `_AGENT_PENDING_SENTINEL` 解决的竞态条件。「检查是否忙碌」和「标记为忙碌」之间不存在异步间隙。

---

## 2. 忙碌时收到新输入的处理

### Hermes：三模式分发

```
busy_input_mode:
  interrupt → agent.interrupt() + 丢弃部分结果 + 启动新 turn
  queue     → 合并到 _pending_messages，当前 turn 继续
  steer     → agent.steer(text)，在下一个工具结果边界注入
```

### Claude Code：工具能力驱动的中断

Claude Code **没有用户可配置的模式**。行为由当前执行工具的 `interruptBehavior` 决定：

```typescript
// handlePromptSubmit.ts:313-351

if (queryGuard.isActive || isExternalLoading) {
  // 仅允许 prompt 和 bash 模式入队
  if (mode !== 'prompt' && mode !== 'bash') return

  // 仅当所有正在执行的工具都允许取消时才中断
  if (params.hasInterruptibleToolInProgress) {
    params.abortController?.abort('interrupt')
  }

  // 始终入队 — 新消息将在下一轮处理
  enqueue({ value: finalInput.trim(), mode, ... })
  return
}
```

`hasInterruptibleToolInProgress` 仅在**所有**当前执行工具的 `interruptBehavior` 都为 `'cancel'` 时为 `true`。默认值为 `'block'`。

```typescript
// Tool.ts:416
interruptBehavior?(): 'cancel' | 'block'
```

| 场景 | Hermes（interrupt 模式） | Claude Code |
|---|---|---|
| LLM 流式输出文本 | 立即中断 | 立即中断（无工具执行中） |
| Bash 脚本执行中 | 中断，杀进程 | **阻塞** — BashTool 返回 `'block'`，消息排队 |
| 文件写入中 | 中断，丢弃部分写入 | **阻塞** — FileWriteTool 返回 `'block'` |
| Sleep/delay 执行中 | 中断 | 中断 — SleepTool 返回 `'cancel'` |
| Bash + Sleep 混合 | 中断 | **阻塞** — Bash 的 `'block'` 阻止中断 |

**设计理由：** Claude Code 的 `'block'` 默认值防止中断破坏性文件操作导致数据损坏。Hermes 的 `interrupt()` 更激进 — 无论工具在做什么都会杀线程，依靠 `persist_session()` 保存部分状态。

---

## 3. 队列架构

### Hermes

```
_pending_messages[session_key]  ← 单个文本槽（换行拼接）
_queued_events[session_key]     ← FIFO 溢出列表（用于 /queue 命令）
```

- 文本消息合并：`existing + "\n" + new_text`
- 图片合并 `media_urls` 和 `media_types`，不触发中断
- Telegram：3 秒宽限期防止消息分片导致误中断
- `/queue` 命令产生独立 turn（FIFO 列表）

### Claude Code

```typescript
// messageQueueManager.ts

type QueuedCommand = {
  value: string | ContentBlockParam[]
  mode: 'prompt' | 'bash' | 'task-notification' | ...
  priority: 'now' | 'next' | 'later'   // now(0) > next(1) > later(2)
  agentId?: string                       // 目标 agent 作用域
  uuid: string
  pastedContents?: PastedContent[]
}
```

- **优先级队列** — `'now'` 命令插队，`'later'`（通知）等待
- **Agent 作用域** — 主线程消费 `agentId===undefined`，子 agent 消费自己的 `agentId`
- **作为 attachment 消费** — turn 之间，`getCommandsByMaxPriority()` 将命令拉入下一轮 API 请求，作为 `queued_command` attachment 块
- **Sleep 感知** — 如果 SleepTool 执行过，仅消费 `'later'` 优先级（不消费 `'next'`），保留用户 prompt 等 sleep 结束后处理
- **批处理** — 多个同 mode 的非斜杠命令一起出队

`query.ts:1570` 中的队列消费：

```typescript
const queuedCommandsSnapshot = getCommandsByMaxPriority(
  sleepRan ? 'later' : 'next',
).filter(cmd => {
  if (isSlashCommand(cmd)) return false
  if (isMainThread) return cmd.agentId === undefined
  return cmd.mode === 'task-notification' && cmd.agentId === currentAgentId
})
```

Agent 作用域过滤防止子 agent 消费用户 prompt，反之亦然。

---

## 4. 取消/中止机制

### Hermes：`interrupt()` 级联传播

```python
# run_agent.py:4796
def interrupt(self, reason=None):
    self._interrupt_requested = True
    self._set_interrupt(True, execution_thread_id)  # 通知工具线程
    for tid in self._tool_worker_threads:            # fan out 到 workers
        ...
    for child in self._active_children:              # 递归传播到子 agent
        child.interrupt(reason)
```

主循环、流式输出、工具执行和重试路径中约 20 个检查点。每个检查点要么抛出 `InterruptedError`，要么 `break` 出当前阶段。

中断后流程：
1. `persist_session()` 保存已执行上下文
2. 返回 `{interrupted: True}` 给 gateway
3. Gateway **丢弃**部分结果
4. 读取 `_pending_messages` 并启动新 turn

### Claude Code：`AbortController` 信号链

```
REPL state (AbortController)
  → query.ts (toolUseContext.abortController)
    → claude.ts API 调用 (signal 参数)
      → Streaming SDK
    → StreamingToolExecutor
      → childAbortController per tool
```

`query.ts` 中的三个中止检查点：

| 检查点 | 行号 | 行为 |
|---|---|---|
| 流式输出后，工具执行前 | 1015 | 消费剩余工具结果（为已中止工具生成合成 tool_result 块），产出中断消息，返回 `aborted_streaming` |
| 工具执行后 | 1485 | 相同清理，返回 `aborted_tools` |
| 流式输出中 | API 层 | 抛出 `APIUserAbortError` |

与 Hermes 不同，Claude Code **不丢弃**部分结果。已执行完的工具结果保留在消息历史中。当 `signal.reason === 'interrupt'` 时**跳过**中断消息（行 1046/1501），因为排队的用户消息已提供足够上下文。

### 取消（ESC 键）

`useCancelRequest.ts` 处理 ESC/Ctrl+C：

1. `queryGuard.forceEnd()` — 释放锁，使过期回调失效
2. 保留流式文本（将部分输出保存到 UI）
3. 若有工具权限弹窗激活 → 调用 onAbort
4. 若有 prompt 弹窗激活 → 拒绝待处理 prompt
5. `abortController?.abort('user-cancel')`

---

## 5.「Steer」的缺失

Hermes 的 `steer()` 模式是 Claude Code **不具备**的最显著特性：

```python
# run_agent.py:4897
def steer(self, text: str) -> bool:
    with self._pending_steer_lock:
        if self._pending_steer:
            self._pending_steer += "\n" + text
        else:
            self._pending_steer = text
    # 不设中断标志。当前操作继续执行。
```

Steer 文本在两个点被消费：
1. 工具执行后：`_apply_pending_steer_to_tool_results()` 追加 `User guidance: {text}` 到最后一条工具消息
2. 下次 API 调用前：相同的注入点

这允许用户在**不丢失工作进度的情况下重定向运行中的 agent** — 例如「聚焦认证模块，别看数据库层」。

### Claude Code 最接近的等价机制

最接近的是子 agent 的 `queued_command` attachment 系统：

```typescript
// SendMessageTool → queuePendingMessage(agentId, message)
// → task.pendingMessages[] 
// → 通过 getAgentPendingMessageAttachments() 消费
// → 作为用户消息注入到下一轮 API 调用
```

但这：
- 仅在 **turn 之间**工作，而非 Hermes steer 那样在工具执行中途
- 仅适用于**子 agent**，不适用于主对话
- 是完整的用户消息，而非工具结果上的轻量「指导」注释

主 REPL 没有 steer 等价机制 — 用户消息要么触发中断（如果工具允许），要么等待当前 turn 完成。

---

## 6. 子 Agent / 后代 Agent 处理

### Hermes

```python
# interrupt() 递归传播到 _active_children
for child in self._active_children:
    child.interrupt(reason)
```

子节点在 AIAgent 实例的 `_active_children` 中追踪。

### Claude Code

子 agent 通过 `LocalAgentTaskState` 管理：

```typescript
// LocalAgentTask.tsx
interface LocalAgentTaskState {
  pendingMessages: string[]        // 中途消息队列
  abortController?: AbortController
  isBackgrounded: boolean
  retain: boolean                  // 防止被驱逐
}
```

子 agent 通信路径：

| 机制 | 如何工作 | 何时消费 |
|---|---|---|
| `queuePendingMessage()` | 追加到 `task.pendingMessages[]` | 下次 API 调用，通过 `getAgentPendingMessageAttachments()` |
| `enqueueAgentNotification()` | 创建 `<task-notification>` XML 块 | 立即作为 attachment 入队 |
| 文件邮箱 | `.claude/teams/{team}/inboxes/` 中的 JSON 文件 | 由 `useInboxPoller` / `inProcessRunner` 等待循环轮询 |
| `SendMessageTool` | 路由到 UDS、进程内队列或邮箱 | 取决于目标 agent 类型 |

进程内 teammate 等待循环（`inProcessRunner.ts:689`）轮询三个来源：
1. `task.pendingUserMessages[]` — 直接注入
2. 文件邮箱 — 跨进程消息
3. 共享任务列表 — 自动领取待处理任务

---

## 7. 总结：优势与取舍

| 维度 | Hermes Agent | Claude Code |
|---|---|---|
| **灵活性** | 三种显式模式，用户可选 | 单一行为，由工具能力决定 |
| **安全性** | 可中断任何工具（文件操作有数据丢失风险） | 破坏性工具执行中阻止中断 |
| **中途引导** | `steer()` — 注入引导不丢失上下文 | 无等价机制 — 必须等待 turn 边界 |
| **并发守卫** | 异步锁 + 哨兵（复杂，有边界情况风险） | 同步状态机（更简单，无竞态） |
| **队列复杂度** | 简单合并 + FIFO | 优先级队列、Agent 作用域、Sleep 感知 |
| **部分结果处理** | 中断时丢弃 | 保留已执行的工具结果 |
| **子 agent 消息** | 递归 `interrupt()` | 多渠道：队列、邮箱、task-notification |
| **过期状态处理** | `_session_run_generation` token + `_heal_stale_session_lock` | `QueryGuard.generation` + `forceEnd()` 失效机制 |

### Claude Code 为什么不需要 Steer

Claude Code 省略 steer 的设计决策与其以工具为中心的架构一致：

1. **工具是工作单元** — Claude Code 的工具（Bash、FileEdit、Agent）是自包含的单轮操作。Hermes 的工具可能更长运行（人机审批循环、多步子 agent）。

2. **子 agent 委派** — Claude Code 通过生成子 agent 来执行独立任务，而非引导运行中的 agent。父 agent 可通过 `SendMessageTool` 向子 agent 发送消息，通过结构化委派而非注入实现类似的「任务中途引导」效果。

3. **Turn 边界可预测** — Claude Code 的 API 循环产生 tool_use → tool_result 对。任何时点的新用户输入要么中断（如果安全），要么等待下一个 turn 边界，这通常最多只有一次工具执行的距离。

### Hermes 为什么需要三种模式

Hermes 的三模式设计反映了其更广泛的部署场景：

1. **Telegram/聊天平台** — 消息平台用户期望即时响应；`interrupt` 模式模仿自然对话的轮流发言。

2. **长时间运行的自主 agent** — `queue` 模式防止一个偶然的「hello」中断批量处理任务。

3. **协作调试** — `steer` 模式允许人类操作员轻推 agent 而不重启其上下文，对数小时长的调试会话至关重要。
