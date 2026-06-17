# Unified Task-Message Processing

**Date:** 2026-05-05
**Status:** design

## Context

Scheduled tasks (`context_mode: 'group'`) currently bypass the normal message processing pipeline. They call `runContainerAgent()` directly with a raw `task.prompt`, skip history messages, use a 10-second container close delay instead of `IDLE_TIMEOUT`, and cannot reuse the group's active container. This means task execution lacks chat context and duplicates container/session management logic.

The goal is to make task execution identical to message processing: pull history, use the latest SDK session, reuse the group's running container, and share the same container lifecycle.

## Design

### Approach: Synthetic message through unified queue

When a `context_mode: 'group'` task fires, create a synthetic message flagged with `scheduled_task_id`, store it in the DB, and run it through a shared helper that both `processGroupMessages` and `runTask` call. The synthetic message bypasses the normal trigger check and is excluded from future history queries.

`context_mode: 'isolated'` tasks keep a separate direct path: they call `runAgent()` without session/history but still benefit from removed `isScheduledTask` flag and `IDLE_TIMEOUT` lifecycle.

### Data Flow

```
Scheduler poll → enqueueTask(chatJid, taskId, closure)
  └─ GroupQueue.runTask() awaits closure
       └─ closure = newRunTask(task):
            1. validate group
            2. create synthetic NewMessage {
                 scheduled_task_id: task.id,
                 content: task.prompt,
                 is_from_me: true
               }
            3. storeMessage() to DB
            4. getMessagesSince() → fetch history
            5. combined = [...history, syntheticMsg]
            6. processMessagesForGroup(group, chatJid, combined, channel)
                 └─ formatMessages() → runAgent() → runContainerAgent()
            7. logTaskRun(), computeNextRun(), updateTaskAfterRun()
```

Message path is unchanged — `processGroupMessages` calls the same `processMessagesForGroup`.

### Files Changed

#### `src/types.ts`
- Add `scheduled_task_id?: string` to `NewMessage`

#### `src/db.ts`
- Add `scheduled_task_id TEXT` column to `messages` table schema
- `storeMessage()` includes `scheduled_task_id` in INSERT
- `getMessagesSince()` and `getNewMessages()` add `AND scheduled_task_id IS NULL` to WHERE clause

#### `src/index.ts`
- Extract `processMessagesForGroup()` from `processGroupMessages`:
  - Takes: group, chatJid, messages[], channel
  - Handles: format → cursor advance → runAgent → stream output → idle timer
  - Returns: `{ status, outputSentToUser }`
- `processGroupMessages` calls `processMessagesForGroup` with fetched messages
- `startMessageLoop()` trigger detection: also treat `scheduled_task_id IS NOT NULL` messages as triggers (bypass TRIGGER_PATTERN)

#### `src/task-scheduler.ts`
- `runTask()` for `context_mode: 'group'`: create synthetic message → store → fetch history → call `processMessagesForGroup` → log/update
- `runTask()` for `context_mode: 'isolated'`: call `runAgent()` directly with no session/history, still uses IDLE_TIMEOUT lifecycle
- Remove: `TASK_CLOSE_DELAY_MS`, `scheduleClose()`, `isScheduledTask: true`, direct `runContainerAgent()` call, all streaming/process callbacks from group mode

#### `src/group-queue.ts`
- `sendMessage()`: remove `!state.isTaskContainer` restriction — task containers accept follow-up IPC

### What is Removed
- `isScheduledTask` flag in `ContainerInput` and container-runner
- `TASK_CLOSE_DELAY_MS` (10s) — replaced by `IDLE_TIMEOUT`
- Container lifecycle management from `runTask()` — `runAgent()` handles it

### Verification
1. `npm run build` compiles
2. Group task: agent sees history context + task prompt, uses existing session, container lives for IDLE_TIMEOUT, result sent to chat, synthetic message excluded from future history
3. Isolated task: fresh session, no history, still uses IDLE_TIMEOUT lifecycle
4. Concurrency: task queues as pending when message is processing, runs after via drain
5. Message processing continues to work (no regression)
