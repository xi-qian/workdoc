# unified-task-message-processing

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make scheduled task execution (`context_mode: 'group'`) use the same pipeline as group message processing — history messages, latest SDK session, container reuse, IDLE_TIMEOUT lifecycle.

**Architecture:** Extract `processMessagesForGroup()` from `processGroupMessages` as a shared helper. Tasks create a synthetic `NewMessage` (flagged with `scheduled_task_id`), fetch history, and call the shared helper. Isolated tasks keep a direct `runAgent()` path but lose the 10-second close.

**Tech Stack:** TypeScript, better-sqlite3, Node.js

---

### Task 1: Add `scheduled_task_id` to types, DB schema, and queries

**Files:**
- Modify: `src/types.ts:53-75`
- Modify: `src/db.ts:17-38` (schema), `src/db.ts:279-295` (storeMessage), `src/db.ts:300-322` (storeMessageDirect), `src/db.ts:336-346` (getNewMessages), `src/db.ts:376-386` (getMessagesSince)

- [ ] **Step 1: Add `scheduled_task_id` to `NewMessage` interface**

In `src/types.ts`, add after `attachment?: MessageAttachment;` (line 74):

```typescript
  scheduled_task_id?: string;
```

- [ ] **Step 2: Add column to messages table schema**

In `src/db.ts`, in `createSchema()`, update the `messages` CREATE TABLE (lines 26-38) to include `scheduled_task_id TEXT` after `card_action TEXT`:

```typescript
    CREATE TABLE IF NOT EXISTS messages (
      id TEXT,
      chat_jid TEXT,
      sender TEXT,
      sender_name TEXT,
      content TEXT,
      timestamp TEXT,
      is_from_me INTEGER,
      is_bot_message INTEGER DEFAULT 0,
      card_action TEXT,
      scheduled_task_id TEXT,
      PRIMARY KEY (id, chat_jid),
      FOREIGN KEY (chat_jid) REFERENCES chats(jid)
    );
```

- [ ] **Step 3: Include `scheduled_task_id` in `storeMessage()` INSERT**

In `src/db.ts`, update the `storeMessage()` function (lines 279-295). Change the INSERT column list and VALUES:

```typescript
export function storeMessage(msg: NewMessage): void {
  db.prepare(
    `INSERT OR REPLACE INTO messages (id, chat_jid, sender, sender_name, content, timestamp, is_from_me, is_bot_message, card_action, message_type, attachment, scheduled_task_id) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)`,
  ).run(
    msg.id,
    msg.chat_jid,
    msg.sender,
    msg.sender_name,
    msg.content,
    msg.timestamp,
    msg.is_from_me ? 1 : 0,
    msg.is_bot_message ? 1 : 0,
    msg.card_action ? JSON.stringify(msg.card_action) : null,
    msg.message_type || null,
    msg.attachment ? JSON.stringify(msg.attachment) : null,
    msg.scheduled_task_id || null,
  );
}
```

- [ ] **Step 4: Include `scheduled_task_id` in `storeMessageDirect()`**

In `src/db.ts`, update `storeMessageDirect()` (lines 300-322) to accept and store `scheduled_task_id`:

```typescript
export function storeMessageDirect(msg: {
  id: string;
  chat_jid: string;
  sender: string;
  sender_name: string;
  content: string;
  timestamp: string;
  is_from_me: boolean;
  is_bot_message?: boolean;
  scheduled_task_id?: string;
}): void {
  db.prepare(
    `INSERT OR REPLACE INTO messages (id, chat_jid, sender, sender_name, content, timestamp, is_from_me, is_bot_message, scheduled_task_id) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?)`,
  ).run(
    msg.id,
    msg.chat_jid,
    msg.sender,
    msg.sender_name,
    msg.content,
    msg.timestamp,
    msg.is_from_me ? 1 : 0,
    msg.is_bot_message ? 1 : 0,
    msg.scheduled_task_id || null,
  );
}
```

- [ ] **Step 5: Filter synthetic messages from `getNewMessages()`**

In `src/db.ts`, update the SQL in `getNewMessages()` (lines 336-346). Add `AND scheduled_task_id IS NULL` to the WHERE clause and `scheduled_task_id` to the SELECT list:

```typescript
  const sql = `
    SELECT * FROM (
      SELECT id, chat_jid, sender, sender_name, content, timestamp, is_from_me, card_action, message_type, attachment, scheduled_task_id
      FROM messages
      WHERE timestamp > ? AND chat_jid IN (${placeholders})
        AND is_bot_message = 0 AND content NOT LIKE ?
        AND content != '' AND content IS NOT NULL
        AND scheduled_task_id IS NULL
      ORDER BY timestamp DESC
      LIMIT ?
    ) ORDER BY timestamp
  `;
```

- [ ] **Step 6: Filter synthetic messages from `getMessagesSince()`**

In `src/db.ts`, update the SQL in `getMessagesSince()` (lines 376-386). Add `AND scheduled_task_id IS NULL` and `scheduled_task_id` to SELECT:

```typescript
  const sql = `
    SELECT * FROM (
      SELECT id, chat_jid, sender, sender_name, content, timestamp, is_from_me, card_action, message_type, attachment, scheduled_task_id
      FROM messages
      WHERE chat_jid = ? AND timestamp > ?
        AND is_bot_message = 0 AND content NOT LIKE ?
        AND content != '' AND content IS NOT NULL
        AND scheduled_task_id IS NULL
      ORDER BY timestamp DESC
      LIMIT ?
    ) ORDER BY timestamp
  `;
```

- [ ] **Step 7: Also filter in `getMessagesBefore()`**

Read `src/db.ts` lines 412-419 and apply the same filter there (used for context history, same protection needed):

```typescript
  const sql = `
    SELECT * FROM (
      SELECT id, chat_jid, sender, sender_name, content, timestamp, is_from_me, card_action, message_type, attachment, scheduled_task_id
      FROM messages
      WHERE chat_jid = ? AND timestamp <= ?
        AND is_bot_message = 0 AND content NOT LIKE ?
        AND content != '' AND content IS NOT NULL
        AND scheduled_task_id IS NULL
      ORDER BY timestamp DESC
      LIMIT ?
    ) ORDER BY timestamp
  `;
```

- [ ] **Step 8: Build and verify**

```bash
npm run build
```

Expected: TypeScript compiles without errors.

- [ ] **Step 9: Commit**

```bash
git add src/types.ts src/db.ts
git commit -m "feat: add scheduled_task_id to messages schema with query filtering"
```

---

### Task 2: Extract `processMessagesForGroup` shared helper from `processGroupMessages`

**Files:**
- Modify: `src/index.ts:280-435`

- [ ] **Step 1: Extract `processMessagesForGroup` function**

In `src/index.ts`, insert this new function BEFORE `processGroupMessages` (before line 280). It encapsulates the core: format → runAgent → stream output → idle timer → error handling:

```typescript
async function processMessagesForGroup(
  group: RegisteredGroup,
  chatJid: string,
  messages: NewMessage[],
): Promise<{ status: 'success' | 'error'; outputSentToUser: boolean }> {
  const channel = findChannel(channels, chatJid);
  if (!channel) {
    logger.warn({ chatJid }, 'No channel owns JID, skipping messages');
    return { status: 'error', outputSentToUser: false };
  }

  const previousCursor = lastAgentTimestamp[chatJid] || '';
  lastAgentTimestamp[chatJid] = messages[messages.length - 1].timestamp;
  saveState();

  const prompt = formatMessages(messages, TIMEZONE);

  let idleTimer: ReturnType<typeof setTimeout> | null = null;
  const resetIdleTimer = () => {
    if (idleTimer) clearTimeout(idleTimer);
    idleTimer = setTimeout(() => {
      logger.debug({ group: group.name }, 'Idle timeout, closing container stdin');
      queue.closeStdin(chatJid);
    }, IDLE_TIMEOUT);
  };

  await channel.setTyping?.(chatJid, true);
  let hadError = false;
  let outputSentToUser = false;

  const result = await runAgent(group, prompt, chatJid, async (output) => {
    try {
      if (output.result) {
        const raw =
          typeof output.result === 'string'
            ? output.result
            : JSON.stringify(output.result);
        const text = raw.replace(/<internal>[\s\S]*?<\/internal>/g, '').trim();
        if (text) {
          await channel.sendMessage(chatJid, text);
          outputSentToUser = true;
        }
        resetIdleTimer();
      }
      if (output.status === 'success') {
        queue.notifyIdle(chatJid);
      }
      if (output.status === 'error') {
        hadError = true;
      }
    } catch (err) {
      logger.error(
        { group: group.name, chatJid, err },
        'onOutput callback failed',
      );
    }
  });

  await channel.setTyping?.(chatJid, false);
  if (idleTimer) clearTimeout(idleTimer);

  if (result === 'error' || hadError) {
    if (outputSentToUser) {
      logger.warn(
        { group: group.name },
        'Agent error after output was sent, skipping cursor rollback',
      );
      return { status: 'error', outputSentToUser: true };
    }
    lastAgentTimestamp[chatJid] = previousCursor;
    saveState();
    logger.warn({ group: group.name }, 'Agent error, rolled back cursor for retry');
    return { status: 'error', outputSentToUser: false };
  }

  return { status: 'success', outputSentToUser };
}
```

- [ ] **Step 2: Rewrite `processGroupMessages` to use the shared helper**

Replace the body of `processGroupMessages` (lines 280-435) with a thin wrapper that fetches messages, does the trigger check, then delegates:

```typescript
async function processGroupMessages(chatJid: string): Promise<boolean> {
  const group = registeredGroups[chatJid];
  if (!group) return true;

  const isMainGroup = group.isMain === true;
  const hasExistingSession = !!sessions[group.folder];

  const sinceTimestamp = lastAgentTimestamp[chatJid] || '';
  const missedMessages = getMessagesSince(
    chatJid,
    sinceTimestamp,
    ASSISTANT_NAME,
  );

  if (missedMessages.length === 0) return true;

  if (!isMainGroup && group.requiresTrigger !== false) {
    const allowlistCfg = loadSenderAllowlist();
    const hasTrigger = missedMessages.some(
      (m) =>
        TRIGGER_PATTERN.test(m.content.trim()) &&
        (m.is_from_me || isTriggerAllowed(chatJid, m.sender, allowlistCfg)),
    );
    if (!hasTrigger) return true;
  }

  logger.info(
    { group: group.name, messageCount: missedMessages.length, hasExistingSession },
    'Processing messages',
  );

  const { outputSentToUser } = await processMessagesForGroup(
    group,
    chatJid,
    missedMessages,
  );

  return !(outputSentToUser === false);
}
```

- [ ] **Step 3: Build and verify**

```bash
npm run build
```

Expected: TypeScript compiles without errors.

- [ ] **Step 4: Commit**

```bash
git add src/index.ts
git commit -m "refactor: extract processMessagesForGroup shared helper from processGroupMessages"
```

---

### Task 3: Remove `isScheduledTask` from container interfaces

**Files:**
- Modify: `src/container-runner.ts:39-47`
- Modify: `container/agent-runner/src/index.ts:80-88`, `container/agent-runner/src/index.ts:602-606`

- [ ] **Step 1: Remove from host-side `ContainerInput`**

In `src/container-runner.ts`, remove line 45 (`isScheduledTask?: boolean;`):

```typescript
export interface ContainerInput {
  prompt: string;
  sessionId?: string;
  groupFolder: string;
  chatJid: string;
  isMain: boolean;
  assistantName?: string;
}
```

- [ ] **Step 2: Remove from container-side `ContainerInput`**

In `container/agent-runner/src/index.ts`, remove line 86 (`isScheduledTask?: boolean;`):

```typescript
interface ContainerInput {
  prompt: string;
  sessionId?: string;
  groupFolder: string;
  chatJid: string;
  isMain: boolean;
  assistantName?: string;
}
```

- [ ] **Step 3: Remove the scheduled-task prefix logic**

In `container/agent-runner/src/index.ts`, replace lines 602-606. Change from:

```typescript
  let prompt = containerInput.prompt;
  if (containerInput.isScheduledTask) {
    prompt = `[SCHEDULED TASK - The following message was sent automatically and is not coming directly from the user or group.]\n\n${prompt}`;
  }
```

To:

```typescript
  const prompt = containerInput.prompt;
```

- [ ] **Step 4: Build and verify**

```bash
npm run build
```

Expected: TypeScript compiles without errors.

- [ ] **Step 5: Commit**

```bash
git add src/container-runner.ts container/agent-runner/src/index.ts
git commit -m "refactor: remove isScheduledTask flag from ContainerInput"
```

---

### Task 4: Remove `!state.isTaskContainer` restriction in `sendMessage()`

**Files:**
- Modify: `src/group-queue.ts:177-219`

- [ ] **Step 1: Remove `isTaskContainer` checks**

In `src/group-queue.ts`, in `sendMessage()`, remove `!state.isTaskContainer` from both conditionals (lines 184 and 195). Change:

```typescript
    if (processAlive && state.groupFolder && !state.isTaskContainer) {
```

To:

```typescript
    if (processAlive && state.groupFolder) {
```

And change:

```typescript
    } else if (state.active && state.groupFolder && !state.isTaskContainer) {
```

To:

```typescript
    } else if (state.active && state.groupFolder) {
```

- [ ] **Step 2: Build and verify**

```bash
npm run build
```

Expected: TypeScript compiles without errors.

- [ ] **Step 3: Commit**

```bash
git add src/group-queue.ts
git commit -m "refactor: allow IPC messages to task containers in sendMessage"
```

---

### Task 5: Rewrite `runTask()` in task-scheduler to use unified path

**Files:**
- Modify: `src/task-scheduler.ts:1-23` (imports), `src/task-scheduler.ts:66-77` (SchedulerDependencies), `src/task-scheduler.ts:79-264` (runTask)

- [ ] **Step 1: Update imports**

In `src/task-scheduler.ts`, update the imports (lines 1-23). Remove unused imports and add needed ones:

```typescript
import { ChildProcess } from 'child_process';
import { CronExpressionParser } from 'cron-parser';
import fs from 'fs';

import { ASSISTANT_NAME, SCHEDULER_POLL_INTERVAL, TIMEZONE } from './config.js';
import {
  runContainerAgent,
  writeTasksSnapshot,
} from './container-runner.js';
import {
  deleteSession,
  getAllTasks,
  getDueTasks,
  getMessagesSince,
  getTaskById,
  logTaskRun,
  storeMessage,
  updateTask,
  updateTaskAfterRun,
} from './db.js';
import { GroupQueue } from './group-queue.js';
import { resolveGroupFolderPath } from './group-folder.js';
import { logger } from './logger.js';
import { NewMessage, RegisteredGroup, ScheduledTask } from './types.js';
```

- [ ] **Step 2: Update `SchedulerDependencies` interface**

Replace lines 66-77. Add `processMessages` and `getLastAgentTimestamp`:

```typescript
export interface SchedulerDependencies {
  registeredGroups: () => Record<string, RegisteredGroup>;
  getSessions: () => Record<string, string>;
  queue: GroupQueue;
  onProcess: (
    groupJid: string,
    proc: ChildProcess,
    containerName: string,
    groupFolder: string,
  ) => void;
  sendMessage: (jid: string, text: string) => Promise<void>;
  processMessages: (
    group: RegisteredGroup,
    chatJid: string,
    messages: NewMessage[],
  ) => Promise<{ status: 'success' | 'error'; outputSentToUser: boolean }>;
  getLastAgentTimestamp: (chatJid: string) => string;
}
```

- [ ] **Step 3: Rewrite `runTask()`**

Replace the entire `runTask()` function (lines 79-264) with the new implementation that branches on `context_mode`:

```typescript
async function runTask(
  task: ScheduledTask,
  deps: SchedulerDependencies,
): Promise<void> {
  const startTime = Date.now();
  let groupDir: string;
  try {
    groupDir = resolveGroupFolderPath(task.group_folder);
  } catch (err) {
    const error = err instanceof Error ? err.message : String(err);
    updateTask(task.id, { status: 'paused' });
    logger.error(
      { taskId: task.id, groupFolder: task.group_folder, error },
      'Task has invalid group folder',
    );
    logTaskRun({
      task_id: task.id,
      run_at: new Date().toISOString(),
      duration_ms: Date.now() - startTime,
      status: 'error',
      result: null,
      error,
    });
    return;
  }
  fs.mkdirSync(groupDir, { recursive: true });

  logger.info(
    { taskId: task.id, group: task.group_folder, mode: task.context_mode },
    'Running scheduled task',
  );

  const groups = deps.registeredGroups();
  const group = Object.values(groups).find(
    (g) => g.folder === task.group_folder,
  );

  if (!group) {
    logger.error(
      { taskId: task.id, groupFolder: task.group_folder },
      'Group not found for task',
    );
    logTaskRun({
      task_id: task.id,
      run_at: new Date().toISOString(),
      duration_ms: Date.now() - startTime,
      status: 'error',
      result: null,
      error: `Group not found: ${task.group_folder}`,
    });
    return;
  }

  const isMain = group.isMain === true;

  // Update tasks snapshot for container to read
  const tasks = getAllTasks();
  writeTasksSnapshot(
    task.group_folder,
    isMain,
    tasks.map((t) => ({
      id: t.id,
      groupFolder: t.group_folder,
      prompt: t.prompt,
      schedule_type: t.schedule_type,
      schedule_value: t.schedule_value,
      status: t.status,
      next_run: t.next_run,
    })),
  );

  let result: string | null = null;
  let error: string | null = null;

  try {
    if (task.context_mode === 'group') {
      // --- Unified path: synthetic message + history + shared helper ---
      const taskMessage: NewMessage = {
        id: `task_${task.id}_${Date.now()}`,
        chat_jid: task.chat_jid,
        sender: 'system',
        sender_name: 'Scheduled Task',
        content: task.prompt,
        timestamp: new Date().toISOString(),
        is_from_me: true,
        scheduled_task_id: task.id,
      };
      storeMessage(taskMessage);

      const sinceTimestamp = deps.getLastAgentTimestamp(task.chat_jid);
      const historyMessages = getMessagesSince(
        task.chat_jid,
        sinceTimestamp,
        ASSISTANT_NAME,
      );
      const allMessages = [...historyMessages, taskMessage];

      const { status, outputSentToUser } = await deps.processMessages(
        group,
        task.chat_jid,
        allMessages,
      );

      if (status === 'error') {
        error = 'Agent processing failed';
      }
    } else {
      // --- Isolated path: direct runAgent, fresh session, no history ---
      const output = await runContainerAgent(
        group,
        {
          prompt: task.prompt,
          groupFolder: task.group_folder,
          chatJid: task.chat_jid,
          isMain,
          assistantName: ASSISTANT_NAME,
        },
        (proc, containerName) =>
          deps.onProcess(task.chat_jid, proc, containerName, task.group_folder),
        async (streamedOutput) => {
          if (streamedOutput.result) {
            result = streamedOutput.result;
            await deps.sendMessage(task.chat_jid, streamedOutput.result);
          }
          if (streamedOutput.status === 'success') {
            deps.queue.notifyIdle(task.chat_jid);
          }
          if (streamedOutput.status === 'error') {
            error = streamedOutput.error || 'Unknown error';
            if (streamedOutput.error?.includes('No conversation found')) {
              deleteSession(task.group_folder);
            }
          }
        },
      );

      if (output.status === 'error') {
        error = output.error || 'Unknown error';
        if (output.error?.includes('No conversation found')) {
          deleteSession(task.group_folder);
        }
      } else if (output.result) {
        result = output.result;
      }
    }

    logger.info(
      { taskId: task.id, durationMs: Date.now() - startTime },
      'Task completed',
    );
  } catch (err) {
    error = err instanceof Error ? err.message : String(err);
    logger.error({ taskId: task.id, error }, 'Task failed');
  }

  const durationMs = Date.now() - startTime;

  logTaskRun({
    task_id: task.id,
    run_at: new Date().toISOString(),
    duration_ms: durationMs,
    status: error ? 'error' : 'success',
    result,
    error,
  });

  const nextRun = computeNextRun(task);
  const resultSummary = error
    ? `Error: ${error}`
    : result
      ? result.slice(0, 200)
      : 'Completed';
  updateTaskAfterRun(task.id, nextRun, resultSummary);
}
```

- [ ] **Step 4: Build and verify**

```bash
npm run build
```

Expected: TypeScript compiles without errors.

- [ ] **Step 5: Commit**

```bash
git add src/task-scheduler.ts
git commit -m "feat: unify task execution with message processing pipeline"
```

---

### Task 6: Wire up new dependencies in `main()`

**Files:**
- Modify: `src/index.ts:854-869` (startSchedulerLoop call)

- [ ] **Step 1: Add `processMessages` and `getLastAgentTimestamp` to scheduler deps**

In `src/index.ts`, update the `startSchedulerLoop()` call (lines 854-869). Add the two new fields:

```typescript
  startSchedulerLoop({
    registeredGroups: () => registeredGroups,
    getSessions: () => sessions,
    queue,
    onProcess: (groupJid, proc, containerName, groupFolder) =>
      queue.registerProcess(groupJid, proc, containerName, groupFolder),
    sendMessage: async (jid, rawText) => {
      const channel = findChannel(channels, jid);
      if (!channel) {
        logger.warn({ jid }, 'No channel owns JID, cannot send message');
        return;
      }
      const text = formatOutbound(rawText);
      if (text) await channel.sendMessage(jid, text);
    },
    processMessages: processMessagesForGroup,
    getLastAgentTimestamp: (chatJid: string) =>
      lastAgentTimestamp[chatJid] || '',
  });
```

- [ ] **Step 2: Build and verify**

```bash
npm run build
```

Expected: TypeScript compiles without errors.

- [ ] **Step 3: Run tests**

```bash
npm run test
```

Expected: All existing tests pass.

- [ ] **Step 4: Commit**

```bash
git add src/index.ts
git commit -m "feat: wire processMessagesForGroup into scheduler dependencies"
```

---

### Task 7: End-to-end verification

- [ ] **Step 1: Full build**

```bash
npm run build
```

Expected: Clean build, no errors.

- [ ] **Step 2: Run test suite**

```bash
npm run test
```

Expected: All tests pass.

- [ ] **Step 3: Manual review of the diff**

```bash
git diff main
```

Verify:
- `isScheduledTask` removed from all `ContainerInput` usages
- `TASK_CLOSE_DELAY_MS` / `scheduleClose()` gone from task-scheduler
- `scheduled_task_id IS NULL` filter in all three message queries
- `!state.isTaskContainer` removed from `sendMessage`
- `processMessagesForGroup` is the single code path for formatting → agent → output streaming

- [ ] **Step 4: Commit any final cleanup**

```bash
git add -A
git commit -m "chore: final cleanup for unified task-message processing"
```
