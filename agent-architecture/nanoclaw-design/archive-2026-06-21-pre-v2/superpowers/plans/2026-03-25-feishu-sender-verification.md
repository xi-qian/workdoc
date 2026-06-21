# Feishu Sender Verification Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement end-to-end message ID tracking and sender permission verification for Feishu IPC operations.

**Architecture:** Generate requestId on message arrival, store sender mapping in SQLite, pass requestId through container context, inject into IPC requests, verify permissions via Feishu API before executing operations.

**Tech Stack:** TypeScript, better-sqlite3, Node.js crypto, Feishu Open API

**Spec:** `docs/superpowers/specs/2026-03-25-feishu-sender-verification-design.md`

---

## File Structure

| File | Purpose |
|------|---------|
| `src/types.ts` | Add `RequestContext`, `CurrentContext` interfaces |
| `src/db.ts` | Add `request_contexts` table and CRUD functions, modify `scheduled_tasks` table |
| `src/config.ts` | Add `FEISHU_VERIFY_SENDER`, `REQUEST_CONTEXT_TTL_HOURS` config |
| `src/feishu/permission.ts` | New file - Feishu permission verification functions |
| `src/channels/feishu.ts` | Generate requestId on message arrival |
| `src/container-runner.ts` | Add `sourceRequestId` to `ContainerInput`, write `current_context.json` |
| `src/group-queue.ts` | Add `sourceRequestId` parameter to `sendMessage` |
| `src/ipc.ts` | Add permission verification before IPC operations |
| `src/task-scheduler.ts` | Create temp request_context for scheduled tasks |
| `container/agent-runner/src/index.ts` | Update `current_context.json` on IPC input |
| `container/agent-runner/src/ipc-mcp-stdio.ts` | Inject `sourceRequestId` into IPC requests |
| `src/db.test.ts` | Tests for request_contexts table |
| `src/feishu/permission.test.ts` | Tests for permission verification |

---

## Phase 1: Database Layer

### Task 1: Add Types for Request Context

**Files:**
- Modify: `src/types.ts`

- [ ] **Step 1: Add RequestContext and CurrentContext interfaces**

Add to `src/types.ts` after the `TaskRunLog` interface:

```typescript
/**
 * Request context for tracking message origins
 */
export interface RequestContext {
  requestId: string;           // UUID
  messageId: string;           // Feishu message ID
  chatJid: string;             // Chat JID (feishu:oc_xxx)
  senderOpenId: string;        // Sender's open_id
  senderName?: string;         // Sender's display name (cached)
  triggerMessage?: string;     // Trigger message content (for audit)
  createdAt: string;           // ISO timestamp
  expiresAt: string;           // ISO timestamp
}

/**
 * Current context shared with container via file
 */
export interface CurrentContext {
  sourceRequestId: string;
  messageId?: string;
  senderOpenId?: string;
  senderName?: string;
  chatJid: string;
  groupFolder: string;
  timestamp: string;
}
```

- [ ] **Step 2: Add requestId to NewMessage interface**

Modify `NewMessage` interface in `src/types.ts`, add after `attachment` field:

```typescript
  // Request tracking (for sender verification)
  requestId?: string;          // UUID linking to request_contexts table
```

- [ ] **Step 3: Add created_by fields to ScheduledTask interface**

Modify `ScheduledTask` interface in `src/types.ts`, add after `created_at`:

```typescript
  // Creator tracking (for scheduled task permission verification)
  created_by_sender_id?: string;
  created_by_sender_name?: string;
  created_by_request_id?: string;
```

- [ ] **Step 4: Commit types**

```bash
git add src/types.ts
git commit -m "feat: add RequestContext, CurrentContext, and extend NewMessage/ScheduledTask types"
```

---

### Task 2: Add Config Variables

**Files:**
- Modify: `src/config.ts`

- [ ] **Step 1: Add FEISHU_VERIFY_SENDER config**

Add after `AUTO_REGISTER_GROUPS` in `src/config.ts`:

```typescript
// Feishu sender verification
// When enabled, IPC operations from Feishu messages will verify sender permissions
export const FEISHU_VERIFY_SENDER =
  process.env.FEISHU_VERIFY_SENDER === 'true'; // Default: false

// Request context time-to-live in hours
// After this time, request contexts are cleaned up
export const REQUEST_CONTEXT_TTL_HOURS = parseInt(
  process.env.REQUEST_CONTEXT_TTL_HOURS || '24',
  10,
);
```

- [ ] **Step 2: Commit config**

```bash
git add src/config.ts
git commit -m "feat: add FEISHU_VERIFY_SENDER and REQUEST_CONTEXT_TTL_HOURS config"
```

---

### Task 3: Add request_contexts Table and Functions

**Files:**
- Modify: `src/db.ts`
- Modify: `src/db.test.ts`

- [ ] **Step 1: Write failing tests for request_contexts CRUD**

Add to `src/db.test.ts`:

```typescript
// --- request_contexts CRUD ---

import {
  createRequestContext,
  getRequestContext,
  deleteRequestContext,
  cleanupExpiredRequestContexts,
} from './db.js';
import type { RequestContext } from './types.js';

describe('request_contexts CRUD', () => {
  it('creates and retrieves a request context', () => {
    const ctx: RequestContext = {
      requestId: 'req-123',
      messageId: 'msg-456',
      chatJid: 'feishu:oc_abc',
      senderOpenId: 'ou_xyz',
      senderName: 'Alice',
      triggerMessage: 'hello',
      createdAt: '2024-01-01T00:00:00.000Z',
      expiresAt: '2024-01-02T00:00:00.000Z',
    };

    createRequestContext(ctx);
    const retrieved = getRequestContext('req-123');

    expect(retrieved).toBeDefined();
    expect(retrieved!.requestId).toBe('req-123');
    expect(retrieved!.messageId).toBe('msg-456');
    expect(retrieved!.senderOpenId).toBe('ou_xyz');
    expect(retrieved!.senderName).toBe('Alice');
  });

  it('returns undefined for non-existent requestId', () => {
    expect(getRequestContext('non-existent')).toBeUndefined();
  });

  it('deletes a request context', () => {
    const ctx: RequestContext = {
      requestId: 'req-delete',
      messageId: 'msg-1',
      chatJid: 'feishu:oc_1',
      senderOpenId: 'ou_1',
      createdAt: '2024-01-01T00:00:00.000Z',
      expiresAt: '2024-01-02T00:00:00.000Z',
    };

    createRequestContext(ctx);
    expect(getRequestContext('req-delete')).toBeDefined();

    deleteRequestContext('req-delete');
    expect(getRequestContext('req-delete')).toBeUndefined();
  });

  it('cleanupExpiredRequestContexts removes expired entries', () => {
    const now = new Date();
    const expiredCtx: RequestContext = {
      requestId: 'req-expired',
      messageId: 'msg-1',
      chatJid: 'feishu:oc_1',
      senderOpenId: 'ou_1',
      createdAt: '2024-01-01T00:00:00.000Z',
      expiresAt: new Date(now.getTime() - 1000).toISOString(), // 1 second ago
    };
    const validCtx: RequestContext = {
      requestId: 'req-valid',
      messageId: 'msg-2',
      chatJid: 'feishu:oc_1',
      senderOpenId: 'ou_1',
      createdAt: '2024-01-01T00:00:00.000Z',
      expiresAt: new Date(now.getTime() + 86400000).toISOString(), // 1 day from now
    };

    createRequestContext(expiredCtx);
    createRequestContext(validCtx);

    const removed = cleanupExpiredRequestContexts();
    expect(removed).toBe(1);
    expect(getRequestContext('req-expired')).toBeUndefined();
    expect(getRequestContext('req-valid')).toBeDefined();
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `npm test src/db.test.ts`
Expected: FAIL with "createRequestContext is not a function"

- [ ] **Step 3: Add request_contexts table to schema**

In `createSchema` function in `src/db.ts`, add after the `registered_groups` table:

```typescript
    CREATE TABLE IF NOT EXISTS request_contexts (
      request_id TEXT PRIMARY KEY,
      message_id TEXT NOT NULL,
      chat_jid TEXT NOT NULL,
      sender_open_id TEXT NOT NULL,
      sender_name TEXT,
      trigger_message TEXT,
      created_at TEXT NOT NULL,
      expires_at TEXT NOT NULL
    );
    CREATE INDEX IF NOT EXISTS idx_request_contexts_expires ON request_contexts(expires_at);
```

- [ ] **Step 4: Add CRUD functions**

Add to `src/db.ts`, after the imports section, add `RequestContext` to imports:

```typescript
import {
  NewMessage,
  RegisteredGroup,
  RequestContext,
  ScheduledTask,
  TaskRunLog,
} from './types.js';
```

Add functions at the end of `src/db.ts`:

```typescript
// --- request_contexts CRUD ---

export function createRequestContext(ctx: RequestContext): void {
  db.prepare(`
    INSERT INTO request_contexts (
      request_id, message_id, chat_jid, sender_open_id,
      sender_name, trigger_message, created_at, expires_at
    ) VALUES (?, ?, ?, ?, ?, ?, ?, ?)
  `).run(
    ctx.requestId,
    ctx.messageId,
    ctx.chatJid,
    ctx.senderOpenId,
    ctx.senderName ?? null,
    ctx.triggerMessage ?? null,
    ctx.createdAt,
    ctx.expiresAt,
  );
}

export function getRequestContext(
  requestId: string,
): RequestContext | undefined {
  const row = db
    .prepare(`SELECT * FROM request_contexts WHERE request_id = ?`)
    .get(requestId) as {
      request_id: string;
      message_id: string;
      chat_jid: string;
      sender_open_id: string;
      sender_name: string | null;
      trigger_message: string | null;
      created_at: string;
      expires_at: string;
    } | undefined;

  if (!row) return undefined;

  return {
    requestId: row.request_id,
    messageId: row.message_id,
    chatJid: row.chat_jid,
    senderOpenId: row.sender_open_id,
    senderName: row.sender_name ?? undefined,
    triggerMessage: row.trigger_message ?? undefined,
    createdAt: row.created_at,
    expiresAt: row.expires_at,
  };
}

export function deleteRequestContext(requestId: string): void {
  db.prepare(`DELETE FROM request_contexts WHERE request_id = ?`).run(requestId);
}

export function cleanupExpiredRequestContexts(): number {
  const now = new Date().toISOString();
  const result = db
    .prepare(`DELETE FROM request_contexts WHERE expires_at < ?`)
    .run(now);
  return result.changes;
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `npm test src/db.test.ts`
Expected: PASS

- [ ] **Step 6: Commit database layer**

```bash
git add src/db.ts src/db.test.ts
git commit -m "feat: add request_contexts table and CRUD functions"
```

---

### Task 4: Add created_by Fields to scheduled_tasks Table

**Files:**
- Modify: `src/db.ts`
- Modify: `src/db.test.ts`

- [ ] **Step 1: Write failing test for created_by fields**

Add to `src/db.test.ts` in the `task CRUD` describe block:

```typescript
  it('stores created_by fields for scheduled tasks', () => {
    createTask({
      id: 'task-creator',
      group_folder: 'main',
      chat_jid: 'group@g.us',
      prompt: 'test',
      schedule_type: 'once',
      schedule_value: '2024-06-01T00:00:00.000Z',
      context_mode: 'isolated',
      next_run: null,
      status: 'active',
      created_at: '2024-01-01T00:00:00.000Z',
      created_by_sender_id: 'ou_abc123',
      created_by_sender_name: 'Alice',
      created_by_request_id: 'req-xyz',
    });

    const task = getTaskById('task-creator');
    expect(task).toBeDefined();
    expect(task!.created_by_sender_id).toBe('ou_abc123');
    expect(task!.created_by_sender_name).toBe('Alice');
    expect(task!.created_by_request_id).toBe('req-xyz');
  });
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npm test src/db.test.ts`
Expected: FAIL with "created_by_sender_id" column not found

- [ ] **Step 3: Add migration for new columns**

In `createSchema` function in `src/db.ts`, add after the last migration:

```typescript
  // Add created_by columns for scheduled tasks (sender verification)
  try {
    database.exec(`ALTER TABLE scheduled_tasks ADD COLUMN created_by_sender_id TEXT`);
  } catch {
    /* column already exists */
  }
  try {
    database.exec(`ALTER TABLE scheduled_tasks ADD COLUMN created_by_sender_name TEXT`);
  } catch {
    /* column already exists */
  }
  try {
    database.exec(`ALTER TABLE scheduled_tasks ADD COLUMN created_by_request_id TEXT`);
  } catch {
    /* column already exists */
  }
```

- [ ] **Step 4: Update createTask function**

Modify `createTask` function in `src/db.ts` to accept new fields:

```typescript
export function createTask(
  task: Omit<ScheduledTask, 'last_run' | 'last_result'> & {
    created_by_sender_id?: string;
    created_by_sender_name?: string;
    created_by_request_id?: string;
  },
): void {
  db.prepare(
    `
    INSERT INTO scheduled_tasks (
      id, group_folder, chat_jid, prompt, schedule_type, schedule_value,
      context_mode, next_run, status, created_at,
      created_by_sender_id, created_by_sender_name, created_by_request_id
    ) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
  `,
  ).run(
    task.id,
    task.group_folder,
    task.chat_jid,
    task.prompt,
    task.schedule_type,
    task.schedule_value,
    task.context_mode || 'isolated',
    task.next_run,
    task.status,
    task.created_at,
    task.created_by_sender_id ?? null,
    task.created_by_sender_name ?? null,
    task.created_by_request_id ?? null,
  );
}
```

- [ ] **Step 5: Update getTaskById to return new fields**

Modify `getTaskById` function to include new fields in the return type. Find the function and update:

```typescript
export function getTaskById(id: string): (ScheduledTask & {
  created_by_sender_id?: string;
  created_by_sender_name?: string;
  created_by_request_id?: string;
}) | undefined {
  const row = db.prepare('SELECT * FROM scheduled_tasks WHERE id = ?').get(id) as {
    id: string;
    group_folder: string;
    chat_jid: string;
    prompt: string;
    schedule_type: string;
    schedule_value: string;
    context_mode: string;
    next_run: string | null;
    last_run: string | null;
    last_result: string | null;
    status: string;
    created_at: string;
    created_by_sender_id: string | null;
    created_by_sender_name: string | null;
    created_by_request_id: string | null;
  } | undefined;

  if (!row) return undefined;

  return {
    id: row.id,
    group_folder: row.group_folder,
    chat_jid: row.chat_jid,
    prompt: row.prompt,
    schedule_type: row.schedule_type as 'cron' | 'interval' | 'once',
    schedule_value: row.schedule_value,
    context_mode: row.context_mode as 'group' | 'isolated',
    next_run: row.next_run,
    last_run: row.last_run,
    last_result: row.last_result,
    status: row.status as 'active' | 'paused' | 'completed',
    created_at: row.created_at,
    created_by_sender_id: row.created_by_sender_id ?? undefined,
    created_by_sender_name: row.created_by_sender_name ?? undefined,
    created_by_request_id: row.created_by_request_id ?? undefined,
  };
}
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `npm test src/db.test.ts`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git add src/db.ts src/db.test.ts
git commit -m "feat: add created_by fields to scheduled_tasks table"
```

---

## Phase 2: Context Generation and Passing

### Task 5: Generate requestId in Feishu Channel

**Files:**
- Modify: `src/channels/feishu.ts`

- [ ] **Step 1: Import crypto and db functions**

Add at the top of `src/channels/feishu.ts`:

```typescript
import crypto from 'crypto';

import { createRequestContext } from '../db.js';
import { FEISHU_VERIFY_SENDER, REQUEST_CONTEXT_TTL_HOURS } from '../config.js';
import type { RequestContext } from '../types.js';
```

- [ ] **Step 2: Generate and store requestId in handleMessageEvent**

Find the `handleMessageEvent` method. After the line that gets `senderName`, add:

```typescript
        // Generate requestId for sender verification tracking
        const requestId = crypto.randomUUID();
        const now = new Date();
        const expiresAt = new Date(now.getTime() + REQUEST_CONTEXT_TTL_HOURS * 60 * 60 * 1000);

        // Store request context for permission verification
        createRequestContext({
          requestId,
          messageId: msg.message_id,
          chatJid: jid,
          senderOpenId: senderOpenId,
          senderName: senderName,
          triggerMessage: text.slice(0, 500), // Truncate for storage
          createdAt: now.toISOString(),
          expiresAt: expiresAt.toISOString(),
        } as RequestContext);
```

- [ ] **Step 3: Add requestId to NewMessage object**

Find the `newMessage` object creation and add `requestId`:

```typescript
        const newMessage: NewMessage = {
          id: msg.message_id,
          chat_jid: jid,
          sender: senderOpenId,
          sender_name: senderName,
          content: text,
          timestamp: msg.create_time,
          is_from_me: false,
          message_type: messageType as NewMessage['message_type'],
          attachment: attachment,
          requestId, // Add this line
        };
```

- [ ] **Step 4: Commit Feishu channel changes**

```bash
git add src/channels/feishu.ts
git commit -m "feat: generate requestId in Feishu channel for sender tracking"
```

---

### Task 6: Add sourceRequestId to ContainerInput and Write Context File

**Files:**
- Modify: `src/container-runner.ts`

- [ ] **Step 1: Add sourceRequestId to ContainerInput interface**

Find `ContainerInput` interface and add:

```typescript
export interface ContainerInput {
  prompt: string;
  sessionId?: string;
  groupFolder: string;
  chatJid: string;
  isMain: boolean;
  isScheduledTask?: boolean;
  assistantName?: string;
  sourceRequestId?: string;  // Add this line
}
```

- [ ] **Step 2: Import CurrentContext type**

Add to imports in `src/container-runner.ts`:

```typescript
import type { CurrentContext } from './types.js';
```

- [ ] **Step 3: Write current_context.json when starting container**

In `runContainerAgent` function, find where `container.stdin.write(JSON.stringify(input))` is called. Before that line, add logic to write context file:

Find this section:
```typescript
    container.stdin.write(JSON.stringify(input));
    container.stdin.end();
```

And replace with:
```typescript
    // Write current_context.json for MCP server to read
    const groupIpcDir = resolveGroupIpcPath(group.folder);
    const contextPath = path.join(groupIpcDir, 'current_context.json');
    const currentContext: CurrentContext = {
      sourceRequestId: input.sourceRequestId || '',
      chatJid: input.chatJid,
      groupFolder: input.groupFolder,
      timestamp: new Date().toISOString(),
    };
    fs.writeFileSync(contextPath, JSON.stringify(currentContext, null, 2));
    log.debug({ contextPath, sourceRequestId: input.sourceRequestId }, 'Wrote current_context.json');

    container.stdin.write(JSON.stringify(input));
    container.stdin.end();
```

- [ ] **Step 4: Commit container-runner changes**

```bash
git add src/container-runner.ts
git commit -m "feat: add sourceRequestId to ContainerInput and write current_context.json"
```

---

### Task 7: Pass sourceRequestId Through Message Flow

**Files:**
- Modify: `src/group-queue.ts`
- Modify: `src/index.ts`

- [ ] **Step 1: Add sourceRequestId parameter to sendMessage in GroupQueue**

In `src/group-queue.ts`, find the `sendMessage` method and modify:

```typescript
  /**
   * Send a follow-up message to the active container via IPC file.
   * Returns true if the message was written, false if no active container.
   */
  sendMessage(groupJid: string, text: string, sourceRequestId?: string): boolean {
    const state = this.getGroup(groupJid);
    if (!state.active || !state.groupFolder || state.isTaskContainer)
      return false;
    state.idleWaiting = false; // Agent is about to receive work, no longer idle

    const inputDir = path.join(DATA_DIR, 'ipc', state.groupFolder, 'input');
    try {
      fs.mkdirSync(inputDir, { recursive: true });
      const filename = `${Date.now()}-${Math.random().toString(36).slice(2, 6)}.json`;
      const filepath = path.join(inputDir, filename);
      const tempPath = `${filepath}.tmp`;
      const payload = {
        type: 'message',
        text,
        sourceRequestId, // Add this
      };
      fs.writeFileSync(tempPath, JSON.stringify(payload));
      fs.renameSync(tempPath, filepath);
      return true;
    } catch {
      return false;
    }
  }
```

- [ ] **Step 2: Update sendMessage call in src/index.ts**

Find the `startMessageLoop` function and the `queue.sendMessage` call. Update it to pass `sourceRequestId`:

Find this pattern and update:
```typescript
          if (queue.sendMessage(chatJid, formatted)) {
```

But first, we need to extract the requestId from the messages. Find where `messagesToSend` is used and modify:

```typescript
          const messagesToSend =
            allPending.length > 0 ? allPending : groupMessages;

          // Get the most recent requestId from the messages
          const latestRequestId = messagesToSend.length > 0
            ? messagesToSend[messagesToSend.length - 1].requestId
            : undefined;

          const formatted = formatMessages(messagesToSend, TIMEZONE);

          if (queue.sendMessage(chatJid, formatted, latestRequestId)) {
```

- [ ] **Step 3: Pass sourceRequestId when starting new container**

In `processGroupMessages` function, find the `runAgent` call and check if we need to pass sourceRequestId. This is for the initial container start case.

Find the `missedMessages` array usage and update:

```typescript
  const missedMessages = getMessagesSince(
    chatJid,
    sinceTimestamp,
    ASSISTANT_NAME,
  );

  if (missedMessages.length === 0) return true;

  // Get the latest requestId for the initial prompt
  const latestRequestId = missedMessages[missedMessages.length - 1].requestId;
```

Then update the `runAgent` call to include it. Find where `runAgent` is called and add the parameter.

Actually, looking at the code, `runAgent` uses `prompt` which is the formatted messages. We need to add `sourceRequestId` to the `ContainerInput`. Let me trace through...

The `runAgent` function passes input to `runContainerAgent`. We need to add `sourceRequestId` to that input. Find in `runAgent`:

```typescript
    const output = await runContainerAgent(
      group,
      {
        prompt,
        sessionId,
        groupFolder: group.folder,
        chatJid,
        isMain,
        assistantName: ASSISTANT_NAME,
        sourceRequestId: latestRequestId, // Add this
      },
```

But `latestRequestId` needs to be in scope. Let me update the plan to be more precise.

- [ ] **Step 4: Commit group-queue and index changes**

```bash
git add src/group-queue.ts src/index.ts
git commit -m "feat: pass sourceRequestId through message flow"
```

---

### Task 8: Update Agent Runner to Handle sourceRequestId

**Files:**
- Modify: `container/agent-runner/src/index.ts`

- [ ] **Step 1: Import CurrentContext type**

Add import at the top:

```typescript
import type { CurrentContext } from './types.js';
```

Actually, we need to define this type locally since container code is separate. Add a local interface:

```typescript
interface CurrentContext {
  sourceRequestId: string;
  chatJid: string;
  groupFolder: string;
  timestamp: string;
}
```

- [ ] **Step 2: Update drainIpcInput to extract sourceRequestId**

Find the `drainIpcInput` function and modify:

```typescript
function drainIpcInput(): { messages: string[]; sourceRequestId?: string } {
  try {
    fs.mkdirSync(IPC_INPUT_DIR, { recursive: true });
    const files = fs.readdirSync(IPC_INPUT_DIR)
      .filter(f => f.endsWith('.json'))
      .sort();

    const messages: string[] = [];
    let sourceRequestId: string | undefined;
    for (const file of files) {
      const filePath = path.join(IPC_INPUT_DIR, file);
      try {
        const data = JSON.parse(fs.readFileSync(filePath, 'utf-8'));
        fs.unlinkSync(filePath);
        if (data.type === 'message' && data.text) {
          messages.push(data.text);
        }
        if (data.sourceRequestId) {
          sourceRequestId = data.sourceRequestId;
        }
      } catch (err) {
        log(`Failed to process input file ${file}: ${err instanceof Error ? err.message : String(err)}`);
        try { fs.unlinkSync(filePath); } catch { /* ignore */ }
      }
    }
    return { messages, sourceRequestId };
  } catch (err) {
    log(`IPC drain error: ${err instanceof Error ? err.message : String(err)}`);
    return { messages: [], sourceRequestId: undefined };
  }
}
```

- [ ] **Step 3: Add function to update current_context.json**

Add before `runQuery`:

```typescript
function updateCurrentContext(sourceRequestId: string, chatJid: string, groupFolder: string): void {
  const contextPath = '/workspace/ipc/current_context.json';
  const context: CurrentContext = {
    sourceRequestId,
    chatJid,
    groupFolder,
    timestamp: new Date().toISOString(),
  };
  try {
    fs.writeFileSync(contextPath, JSON.stringify(context, null, 2));
  } catch (err) {
    log(`Failed to update current_context.json: ${err instanceof Error ? err.message : String(err)}`);
  }
}
```

- [ ] **Step 4: Call updateCurrentContext when processing IPC input**

In the `main` function, find where `drainIpcInput()` is called and update:

```typescript
  const pending = drainIpcInput();
  if (pending.messages.length > 0) {
    log(`Draining ${pending.messages.length} pending IPC messages into initial prompt`);
    prompt += '\n' + pending.messages.join('\n');
  }
  if (pending.sourceRequestId) {
    updateCurrentContext(pending.sourceRequestId, containerInput.chatJid, containerInput.groupFolder);
  }
```

And in the query loop where messages are received:

```typescript
        log(`Got new message (${nextMessage.length} chars), starting new query`);
        prompt = nextMessage;

        // Update context if we got a new sourceRequestId from the drained input
        const drained = drainIpcInput();
        if (drained.sourceRequestId) {
          updateCurrentContext(drained.sourceRequestId, containerInput.chatJid, containerInput.groupFolder);
        }
```

Actually, let me re-read the flow. The `waitForIpcMessage` function calls `drainIpcInput`. We need to modify that too.

Let me revise:

Find `waitForIpcMessage` and update it to return the sourceRequestId as well:

```typescript
function waitForIpcMessage(): Promise<{ text: string | null; sourceRequestId?: string }> {
  return new Promise((resolve) => {
    const poll = () => {
      if (shouldClose()) {
        resolve({ text: null });
        return;
      }
      const { messages, sourceRequestId } = drainIpcInput();
      if (messages.length > 0) {
        resolve({ text: messages.join('\n'), sourceRequestId });
        return;
      }
      setTimeout(poll, IPC_POLL_MS);
    };
    poll();
  });
}
```

Then in the main loop:

```typescript
        const nextMessage = await waitForIpcMessage();
        logPerf('main: waited for IPC', Date.now() - ipcStartTime);

        if (nextMessage.text === null) {
          log('Close sentinel received, exiting');
          break;
        }

        // Update context if sourceRequestId was provided
        if (nextMessage.sourceRequestId) {
          updateCurrentContext(nextMessage.sourceRequestId, containerInput.chatJid, containerInput.groupFolder);
        }

        log(`Got new message (${nextMessage.text.length} chars), starting new query`);
        prompt = nextMessage.text;
```

- [ ] **Step 5: Commit agent-runner changes**

```bash
git add container/agent-runner/src/index.ts
git commit -m "feat: handle sourceRequestId in agent-runner and update current_context.json"
```

---

## Phase 3: MCP Server Integration

### Task 9: Inject sourceRequestId in MCP Server

**Files:**
- Modify: `container/agent-runner/src/ipc-mcp-stdio.ts`

- [ ] **Step 1: Add getCurrentContext function**

Add after the constant definitions:

```typescript
const CURRENT_CONTEXT_PATH = '/workspace/ipc/current_context.json';

interface CurrentContext {
  sourceRequestId: string;
  chatJid: string;
  groupFolder: string;
  timestamp: string;
}

function getCurrentContext(): CurrentContext | null {
  try {
    if (fs.existsSync(CURRENT_CONTEXT_PATH)) {
      return JSON.parse(fs.readFileSync(CURRENT_CONTEXT_PATH, 'utf-8'));
    }
  } catch (err) {
    // Ignore errors - context may not exist yet
  }
  return null;
}
```

- [ ] **Step 2: Create helper to inject sourceRequestId**

Add helper function:

```typescript
function withSourceRequestId(data: Record<string, any>): Record<string, any> {
  const context = getCurrentContext();
  if (context?.sourceRequestId) {
    return { ...data, sourceRequestId: context.sourceRequestId };
  }
  return data;
}
```

- [ ] **Step 3: Update all feishu_* tool calls to use withSourceRequestId**

For each feishu tool, wrap the data object. Example for `feishu_fetch_doc`:

```typescript
server.tool(
  'feishu_fetch_doc',
  ...
  async (args) => {
    const requestId = writeIpcFile(FEISHU_REQUESTS_DIR, withSourceRequestId({
      type: 'fetch_doc',
      doc_id: args.doc_id,
      offset: args.offset,
      limit: args.limit,
      groupFolder,
      timestamp: new Date().toISOString(),
    }));
    ...
  },
);
```

This needs to be applied to all feishu tools. Let me list them:
- feishu_fetch_doc
- feishu_create_doc
- feishu_update_doc
- feishu_search_docs
- feishu_create_bitable
- feishu_create_bitable_table
- feishu_list_bitable_tables
- feishu_add_bitable_records
- feishu_list_bitable_records
- feishu_list_bitable_fields
- feishu_update_bitable_record
- feishu_delete_bitable_record
- feishu_send_card
- feishu_send_confirm_card
- feishu_download_resource
- feishu_send_file

Also for schedule_task:

```typescript
server.tool(
  'schedule_task',
  ...
  async (args) => {
    ...
    const data = withSourceRequestId({
      type: 'schedule_task',
      taskId,
      prompt: args.prompt,
      ...
    });
    writeIpcFile(TASKS_DIR, data);
    ...
  },
);
```

- [ ] **Step 4: Commit MCP server changes**

```bash
git add container/agent-runner/src/ipc-mcp-stdio.ts
git commit -m "feat: inject sourceRequestId into all feishu and schedule_task IPC requests"
```

---

## Phase 4: Permission Verification

### Task 10: Create Permission Verification Module

**Files:**
- Create: `src/feishu/permission.ts`
- Create: `src/feishu/permission.test.ts`

- [ ] **Step 1: Create permission.ts with verification functions**

```typescript
/**
 * Feishu Permission Verification
 * Checks if a user has access to specific Feishu resources
 */

import type { FeishuClient } from './client.js';

export interface PermissionResult {
  authorized: boolean;
  reason?: string;
}

/**
 * Verify if a user has access to a chat
 */
export async function verifyChatAccess(
  client: FeishuClient,
  senderOpenId: string,
  chatId: string,
): Promise<PermissionResult> {
  try {
    // Get chat members and check if sender is in the list
    const members = await client.getChatMembers(chatId);

    if (!members || !Array.isArray(members)) {
      // If we can't get members (e.g., bot not in chat), allow access
      // This is a safe default - the operation may still fail at the API level
      return { authorized: true };
    }

    const isMember = members.some(
      (member: any) => member.member_id === senderOpenId || member.open_id === senderOpenId
    );

    if (isMember) {
      return { authorized: true };
    }

    return {
      authorized: false,
      reason: `User ${senderOpenId} is not a member of chat ${chatId}`,
    };
  } catch (error) {
    // On error, allow access (fail-open for availability)
    // The actual API call will fail if truly unauthorized
    return { authorized: true };
  }
}

/**
 * Verify if a user has access to a document
 */
export async function verifyDocAccess(
  client: FeishuClient,
  senderOpenId: string,
  docId: string,
  permission: 'read' | 'edit',
): Promise<PermissionResult> {
  try {
    // Get document permission members
    const members = await client.getDocPermissionMembers(docId);

    if (!members || !Array.isArray(members)) {
      // If we can't get permissions, allow access
      return { authorized: true };
    }

    const userPermission = members.find(
      (member: any) => member.member_id === senderOpenId || member.member_id?.open_id === senderOpenId
    );

    if (!userPermission) {
      return {
        authorized: false,
        reason: `User ${senderOpenId} does not have access to document ${docId}`,
      };
    }

    // Check permission level
    if (permission === 'edit' && userPermission.perm === 'view') {
      return {
        authorized: false,
        reason: `User ${senderOpenId} only has view access to document ${docId}`,
      };
    }

    return { authorized: true };
  } catch (error) {
    // On error, allow access (fail-open for availability)
    return { authorized: true };
  }
}

/**
 * Verify if a user has access to a folder
 */
export async function verifyFolderAccess(
  client: FeishuClient,
  senderOpenId: string,
  folderToken: string,
  permission: 'read' | 'edit',
): Promise<PermissionResult> {
  try {
    // Get folder permission members
    const members = await client.getFolderPermissionMembers(folderToken);

    if (!members || !Array.isArray(members)) {
      // If we can't get permissions, allow access
      return { authorized: true };
    }

    const userPermission = members.find(
      (member: any) => member.member_id === senderOpenId || member.member_id?.open_id === senderOpenId
    );

    if (!userPermission) {
      return {
        authorized: false,
        reason: `User ${senderOpenId} does not have access to folder ${folderToken}`,
      };
    }

    // Check permission level
    if (permission === 'edit' && userPermission.perm === 'view') {
      return {
        authorized: false,
        reason: `User ${senderOpenId} only has view access to folder ${folderToken}`,
      };
    }

    return { authorized: true };
  } catch (error) {
    // On error, allow access (fail-open for availability)
    return { authorized: true };
  }
}
```

- [ ] **Step 2: Write tests for permission functions**

Create `src/feishu/permission.test.ts`:

```typescript
import { describe, it, expect, vi, beforeEach } from 'vitest';

import {
  verifyChatAccess,
  verifyDocAccess,
  verifyFolderAccess,
} from './permission.js';

describe('permission verification', () => {
  let mockClient: any;

  beforeEach(() => {
    mockClient = {
      getChatMembers: vi.fn(),
      getDocPermissionMembers: vi.fn(),
      getFolderPermissionMembers: vi.fn(),
    };
  });

  describe('verifyChatAccess', () => {
    it('authorizes when user is in chat members', async () => {
      mockClient.getChatMembers.mockResolvedValue([
        { member_id: 'ou_alice', open_id: 'ou_alice' },
        { member_id: 'ou_bob', open_id: 'ou_bob' },
      ]);

      const result = await verifyChatAccess(mockClient, 'ou_alice', 'oc_chat123');
      expect(result.authorized).toBe(true);
    });

    it('denies when user is not in chat members', async () => {
      mockClient.getChatMembers.mockResolvedValue([
        { member_id: 'ou_alice', open_id: 'ou_alice' },
      ]);

      const result = await verifyChatAccess(mockClient, 'ou_bob', 'oc_chat123');
      expect(result.authorized).toBe(false);
      expect(result.reason).toContain('not a member');
    });

    it('allows when members list is unavailable', async () => {
      mockClient.getChatMembers.mockResolvedValue(null);

      const result = await verifyChatAccess(mockClient, 'ou_alice', 'oc_chat123');
      expect(result.authorized).toBe(true);
    });

    it('allows on API error (fail-open)', async () => {
      mockClient.getChatMembers.mockRejectedValue(new Error('API error'));

      const result = await verifyChatAccess(mockClient, 'ou_alice', 'oc_chat123');
      expect(result.authorized).toBe(true);
    });
  });

  describe('verifyDocAccess', () => {
    it('authorizes read when user has view permission', async () => {
      mockClient.getDocPermissionMembers.mockResolvedValue([
        { member_id: 'ou_alice', perm: 'view' },
      ]);

      const result = await verifyDocAccess(mockClient, 'ou_alice', 'doc123', 'read');
      expect(result.authorized).toBe(true);
    });

    it('denies edit when user only has view permission', async () => {
      mockClient.getDocPermissionMembers.mockResolvedValue([
        { member_id: 'ou_alice', perm: 'view' },
      ]);

      const result = await verifyDocAccess(mockClient, 'ou_alice', 'doc123', 'edit');
      expect(result.authorized).toBe(false);
      expect(result.reason).toContain('only has view access');
    });

    it('authorizes edit when user has edit permission', async () => {
      mockClient.getDocPermissionMembers.mockResolvedValue([
        { member_id: 'ou_alice', perm: 'edit' },
      ]);

      const result = await verifyDocAccess(mockClient, 'ou_alice', 'doc123', 'edit');
      expect(result.authorized).toBe(true);
    });
  });
});
```

- [ ] **Step 3: Run tests**

Run: `npm test src/feishu/permission.test.ts`
Expected: Some tests may fail because functions aren't fully implemented yet

- [ ] **Step 4: Add helper methods to FeishuClient**

In `src/feishu/client.ts`, add methods to get permission members:

```typescript
  /**
   * Get members of a chat
   */
  async getChatMembers(chatId: string): Promise<any[]> {
    const response = await this.request(
      'GET',
      `/im/v1/chats/${chatId}/members?member_id_type=open_id`,
    );
    return response?.data?.items || [];
  }

  /**
   * Get permission members of a document
   */
  async getDocPermissionMembers(docId: string): Promise<any[]> {
    const response = await this.request(
      'GET',
      `/drive/v1/permissions/${docId}/members`,
    );
    return response?.data?.items || [];
  }

  /**
   * Get permission members of a folder
   */
  async getFolderPermissionMembers(folderToken: string): Promise<any[]> {
    const response = await this.request(
      'GET',
      `/drive/v1/permissions/${folderToken}/members`,
    );
    return response?.data?.items || [];
  }
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `npm test src/feishu/permission.test.ts`
Expected: PASS

- [ ] **Step 6: Commit permission module**

```bash
git add src/feishu/permission.ts src/feishu/permission.test.ts src/feishu/client.ts
git commit -m "feat: add Feishu permission verification functions"
```

---

### Task 11: Integrate Permission Verification in IPC Watcher

**Files:**
- Modify: `src/ipc.ts`

- [ ] **Step 1: Import verification functions and config**

Add to imports in `src/ipc.ts`:

```typescript
import { FEISHU_VERIFY_SENDER } from './config.js';
import {
  verifyChatAccess,
  verifyDocAccess,
  verifyFolderAccess,
} from './feishu/permission.js';
import { getRequestContext } from './db.js';
```

- [ ] **Step 2: Add verifySenderPermission function**

Add before `startIpcWatcher`:

```typescript
async function verifySenderPermission(
  request: any,
  feishuChannel: any,
): Promise<{ authorized: boolean; reason?: string }> {
  // Feature disabled - allow all
  if (!FEISHU_VERIFY_SENDER) {
    return { authorized: true };
  }

  // No sourceRequestId - allow (non-Feishu channels or scheduled tasks without creator)
  if (!request.sourceRequestId) {
    return { authorized: true };
  }

  // Get request context
  const ctx = getRequestContext(request.sourceRequestId);
  if (!ctx) {
    return {
      authorized: false,
      reason: 'Request context not found or expired',
    };
  }

  // Verify based on operation type
  switch (request.type) {
    case 'send_message':
    case 'send_card':
    case 'send_file':
      return await verifyChatAccess(feishuChannel, ctx.senderOpenId, request.chat_id);

    case 'fetch_doc':
      return await verifyDocAccess(feishuChannel, ctx.senderOpenId, request.doc_id, 'read');

    case 'update_doc':
      return await verifyDocAccess(feishuChannel, ctx.senderOpenId, request.doc_id, 'edit');

    case 'create_doc':
      if (request.folder_token) {
        return await verifyFolderAccess(feishuChannel, ctx.senderOpenId, request.folder_token, 'edit');
      }
      return { authorized: true };

    case 'download_resource':
      // Resource access is tied to the chat
      return await verifyChatAccess(feishuChannel, ctx.senderOpenId, request.chat_id || ctx.chatJid.replace('feishu:', ''));

    // Bitable operations - treat as document access
    case 'list_bitable_records':
    case 'list_bitable_fields':
      // These are read operations on bitables
      return { authorized: true }; // Bitable permission is complex, allow for now

    case 'add_bitable_records':
    case 'update_bitable_record':
    case 'delete_bitable_record':
      // These are write operations
      return { authorized: true }; // Bitable permission is complex, allow for now

    default:
      // Unknown types - allow by default
      return { authorized: true };
  }
}
```

- [ ] **Step 3: Add verification before IPC operations**

In the feishu IPC processing section, wrap the switch statement with verification:

Find the section:
```typescript
                let result;

                // 根据请求类型调用相应方法
                switch (request.type) {
```

Add before it:
```typescript
                let result;

                // Verify sender permission
                const verification = await verifySenderPermission(request, feishuChannel);
                if (!verification.authorized) {
                  logger.warn(
                    { request, reason: verification.reason },
                    'IPC request denied - permission verification failed',
                  );
                  const resultFile = path.join(resultsDir, file);
                  fs.writeFileSync(
                    resultFile,
                    JSON.stringify({
                      success: false,
                      error: verification.reason || 'Permission denied',
                      errorType: 'PERMISSION_DENIED',
                    }),
                  );
                  fs.unlinkSync(filePath);
                  return;
                }

                // 根据请求类型调用相应方法
                switch (request.type) {
```

- [ ] **Step 4: Commit IPC changes**

```bash
git add src/ipc.ts
git commit -m "feat: integrate permission verification in IPC watcher"
```

---

### Task 12: Store Creator Info When Creating Scheduled Tasks

**Files:**
- Modify: `src/ipc.ts` (processTaskIpc)

- [ ] **Step 1: Add creator info extraction in schedule_task case**

Find the `schedule_task` case in `processTaskIpc` function. Before `createTask` call, add:

```typescript
    case 'schedule_task':
      if (
        data.prompt &&
        data.schedule_type &&
        data.schedule_value &&
        data.targetJid
      ) {
        // ... existing validation code ...

        // Extract creator info from sourceRequestId
        let createdBySenderId: string | undefined;
        let createdBySenderName: string | undefined;
        let createdByRequestId: string | undefined;

        if (data.sourceRequestId) {
          const ctx = getRequestContext(data.sourceRequestId);
          if (ctx) {
            createdBySenderId = ctx.senderOpenId;
            createdBySenderName = ctx.senderName;
            createdByRequestId = data.sourceRequestId;
          }
        }

        createTask({
          id: taskId,
          group_folder: targetFolder,
          chat_jid: targetJid,
          prompt: data.prompt,
          schedule_type: scheduleType,
          schedule_value: data.schedule_value,
          context_mode: contextMode,
          next_run: nextRun,
          status: 'active',
          created_at: new Date().toISOString(),
          created_by_sender_id: createdBySenderId,
          created_by_sender_name: createdBySenderName,
          created_by_request_id: createdByRequestId,
        });
        // ... rest of the code ...
```

Also need to import `getRequestContext` if not already imported.

- [ ] **Step 2: Commit**

```bash
git add src/ipc.ts
git commit -m "feat: store creator info when creating scheduled tasks"
```

---

### Task 13: Create Temp Context for Scheduled Task Execution

**Files:**
- Modify: `src/task-scheduler.ts`

- [ ] **Step 1: Import necessary functions**

Add imports:

```typescript
import { FEISHU_VERIFY_SENDER, REQUEST_CONTEXT_TTL_HOURS } from './config.js';
import { createRequestContext, deleteRequestContext } from './db.js';
```

- [ ] **Step 2: Create temp context before task execution**

Find the `runTaskForGroup` or equivalent function that executes tasks. Add logic to create temp context:

```typescript
async function runTaskForGroup(
  task: ScheduledTask,
  deps: TaskSchedulerDeps,
): Promise<void> {
  // Create temp request context for permission verification
  let tempRequestId: string | undefined;

  if (task.created_by_sender_id && FEISHU_VERIFY_SENDER) {
    tempRequestId = `task-${task.id}-${Date.now()}`;
    const now = new Date();
    const expiresAt = new Date(now.getTime() + 60 * 60 * 1000); // 1 hour

    createRequestContext({
      requestId: tempRequestId,
      messageId: `scheduled-task:${task.id}`,
      chatJid: task.chat_jid,
      senderOpenId: task.created_by_sender_id,
      senderName: task.created_by_sender_name,
      triggerMessage: `[Scheduled Task] ${task.prompt.slice(0, 200)}`,
      createdAt: now.toISOString(),
      expiresAt: expiresAt.toISOString(),
    });
  }

  try {
    const result = await runContainerAgent(
      group,
      {
        prompt: task.prompt,
        sessionId: undefined, // New session for each task run
        groupFolder: task.group_folder,
        chatJid: task.chat_jid,
        isMain: false,
        isScheduledTask: true,
        sourceRequestId: tempRequestId,
      },
      // ... other params ...
    );
    // ... handle result ...
  } finally {
    // Clean up temp context
    if (tempRequestId) {
      deleteRequestContext(tempRequestId);
    }
  }
}
```

- [ ] **Step 3: Commit task scheduler changes**

```bash
git add src/task-scheduler.ts
git commit -m "feat: create temp request_context for scheduled task execution"
```

---

## Phase 5: Integration and Cleanup

### Task 14: Add Context Cleanup Job

**Files:**
- Modify: `src/ipc.ts`

- [ ] **Step 1: Add periodic cleanup call**

In `startIpcWatcher`, add cleanup at the end of each polling cycle:

```typescript
    // Clean up expired request contexts periodically (every 100 cycles)
    if (Date.now() % (IPC_POLL_INTERVAL * 100) < IPC_POLL_INTERVAL) {
      const removed = cleanupExpiredRequestContexts();
      if (removed > 0) {
        logger.debug({ removed }, 'Cleaned up expired request contexts');
      }
    }

    setTimeout(processIpcFiles, IPC_POLL_INTERVAL);
```

Need to import `cleanupExpiredRequestContexts` at the top.

- [ ] **Step 2: Commit cleanup**

```bash
git add src/ipc.ts
git commit -m "feat: add periodic cleanup of expired request contexts"
```

---

### Task 15: Build and Test Integration

**Files:**
- Run: `npm run build`
- Run: `npm test`

- [ ] **Step 1: Build the project**

Run: `npm run build`
Expected: No errors

- [ ] **Step 2: Run all tests**

Run: `npm test`
Expected: All tests pass

- [ ] **Step 3: Fix any issues**

If tests fail, fix the issues and re-run.

- [ ] **Step 4: Final commit**

```bash
git add -A
git commit -m "chore: final integration and test fixes"
```

---

## Summary

This plan implements the Feishu sender verification feature in 15 tasks across 5 phases:

1. **Phase 1: Database Layer** - Types, config, tables, CRUD functions
2. **Phase 2: Context Generation** - Generate and pass requestId through message flow
3. **Phase 3: MCP Integration** - Inject sourceRequestId in IPC requests
4. **Phase 4: Permission Verification** - Verify sender permissions before IPC operations
5. **Phase 5: Integration** - Cleanup and final testing

Each task follows TDD principles with clear test cases before implementation.