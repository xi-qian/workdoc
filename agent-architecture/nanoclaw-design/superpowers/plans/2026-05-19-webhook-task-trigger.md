# Webhook Task Trigger Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add HTTP endpoints that trigger configurable tasks via the existing Feishu webhook server.

**Architecture:** Extend the Feishu webhook HTTP server with additional routes. A new `webhook-tasks.ts` module reads task definitions from `webhook-tasks.json`, interpolates URL query parameters into prompt templates, and creates one-shot `ScheduledTask` entries in the DB. The existing scheduler loop picks them up and executes them.

**Tech Stack:** Node.js `http` module (already in use), SQLite via `better-sqlite3`, TypeScript.

---

## File Structure

| File | Action | Purpose |
|------|--------|---------|
| `src/webhook-tasks.ts` | Create | Config loader, prompt interpolation, request handler |
| `src/webhook-tasks.test.ts` | Create | Unit tests for interpolation, matching, handler |
| `src/feishu/client.ts` | Modify | Delegate non-Feishu paths to webhook task handler |
| `src/config.ts` | Modify | Add `WEBHOOK_TASKS_CONFIG` path constant |
| `webhook-tasks.json` | Create | Default empty task config |

---

### Task 1: Add config constant

**Files:**
- Modify: `src/config.ts`

- [ ] **Step 1: Add `WEBHOOK_TASKS_CONFIG` constant**

In `src/config.ts`, add after line 38 (`export const DATA_DIR = ...`):

```typescript
export const WEBHOOK_TASKS_CONFIG = path.resolve(
  PROJECT_ROOT,
  process.env.WEBHOOK_TASKS_CONFIG || 'webhook-tasks.json',
);
```

- [ ] **Step 2: Verify build**

Run: `npm run build`
Expected: clean compile, no errors

- [ ] **Step 3: Commit**

```bash
git add src/config.ts
git commit -m "feat(webhook-tasks): add WEBHOOK_TASKS_CONFIG path constant"
```

---

### Task 2: Create `webhook-tasks.ts` with interpolation and matching logic

**Files:**
- Create: `src/webhook-tasks.ts`

- [ ] **Step 1: Write `src/webhook-tasks.ts`**

```typescript
import fs from 'fs';
import http from 'http';

import { logger } from './logger.js';
import { ScheduledTask } from './types.js';
import { WEBHOOK_TASKS_CONFIG } from './config.js';
import { createTask } from './db.js';

const log = logger.child({ module: 'webhook-tasks' });

export interface WebhookTaskConfig {
  name: string;
  path: string;
  group_folder?: string;
  prompt: string;
  context_mode?: 'group' | 'isolated';
}

export interface WebhookTasksFile {
  tasks: WebhookTaskConfig[];
}

/**
 * Load webhook task definitions from the JSON config file.
 * Returns empty array if file doesn't exist or is invalid.
 */
export function loadWebhookTasks(): WebhookTaskConfig[] {
  try {
    if (!fs.existsSync(WEBHOOK_TASKS_CONFIG)) {
      return [];
    }
    const raw = fs.readFileSync(WEBHOOK_TASKS_CONFIG, 'utf-8');
    const data: WebhookTasksFile = JSON.parse(raw);
    return Array.isArray(data.tasks) ? data.tasks : [];
  } catch (err) {
    log.error({ err }, 'Failed to load webhook tasks config');
    return [];
  }
}

/**
 * Match a request URL path against configured webhook task paths.
 * Returns the matching task config, or undefined.
 */
export function matchWebhookTask(
  urlPath: string,
  tasks: WebhookTaskConfig[],
): WebhookTaskConfig | undefined {
  return tasks.find((t) => urlPath === t.path);
}

/**
 * Replace {paramName} placeholders in a prompt template with
 * values from the params object. Unmatched placeholders remain as-is.
 */
export function interpolatePrompt(
  template: string,
  params: Record<string, string>,
): string {
  return template.replace(/\{(\w+)\}/g, (match, key: string) => {
    return key in params ? params[key] : match;
  });
}

/**
 * Parse URL query parameters from a request URL string.
 */
export function parseQueryParams(urlStr: string): Record<string, string> {
  const params: Record<string, string> = {};
  const idx = urlStr.indexOf('?');
  if (idx === -1) return params;
  const qs = urlStr.slice(idx + 1);
  for (const pair of qs.split('&')) {
    const eqIdx = pair.indexOf('=');
    if (eqIdx === -1) continue;
    const key = decodeURIComponent(pair.slice(0, eqIdx));
    const value = decodeURIComponent(pair.slice(eqIdx + 1));
    params[key] = value;
  }
  return params;
}

/**
 * Extract just the path portion from a URL string (before the ?).
 */
export function extractPath(urlStr: string): string {
  const idx = urlStr.indexOf('?');
  return idx === -1 ? urlStr : urlStr.slice(0, idx);
}

export interface WebhookTaskDeps {
  getMainGroup: () => { jid: string; folder: string } | undefined;
  registeredGroups: () => Record<string, { folder: string }>;
}

/**
 * Handle an incoming webhook task HTTP request.
 * Reads config, matches route, interpolates prompt, creates a
 * one-shot ScheduledTask, and responds.
 */
export function handleWebhookTaskRequest(
  req: http.IncomingMessage,
  res: http.ServerResponse,
  deps: WebhookTaskDeps,
): void {
  const urlStr = req.url || '/';
  const urlPath = extractPath(urlStr);
  const tasks = loadWebhookTasks();
  const task = matchWebhookTask(urlPath, tasks);

  if (!task) {
    res.writeHead(404, { 'Content-Type': 'application/json' });
    res.end(JSON.stringify({ ok: false, error: `Task not found: ${urlPath}` }));
    return;
  }

  // Resolve group_folder — default to main group
  let groupFolder = task.group_folder;
  let chatJid: string;
  if (!groupFolder) {
    const mainGroup = deps.getMainGroup();
    if (!mainGroup) {
      res.writeHead(500, { 'Content-Type': 'application/json' });
      res.end(
        JSON.stringify({
          ok: false,
          error: 'No main group configured and no group_folder specified',
        }),
      );
      return;
    }
    groupFolder = mainGroup.folder;
    chatJid = mainGroup.jid;
  } else {
    // Look up chat_jid from registered groups by folder
    const groups = deps.registeredGroups();
    const entry = Object.entries(groups).find(
      ([, g]) => g.folder === groupFolder,
    );
    if (!entry) {
      res.writeHead(400, { 'Content-Type': 'application/json' });
      res.end(
        JSON.stringify({
          ok: false,
          error: `Group folder not found: ${groupFolder}`,
        }),
      );
      return;
    }
    chatJid = entry[0];
  }

  // Interpolate prompt with URL query parameters
  const params = parseQueryParams(urlStr);
  const prompt = interpolatePrompt(task.prompt, params);

  // Create a one-shot task scheduled for now
  const now = new Date();
  const taskId = `webhook_${task.name}_${now.getTime()}`;

  const scheduledTask: Omit<ScheduledTask, 'last_run' | 'last_result'> = {
    id: taskId,
    group_folder: groupFolder,
    chat_jid: chatJid,
    prompt,
    schedule_type: 'once',
    schedule_value: now.toISOString(),
    context_mode: task.context_mode || 'group',
    next_run: now.toISOString(),
    status: 'active',
    created_at: now.toISOString(),
  };

  try {
    createTask(scheduledTask);
    log.info({ taskId, taskName: task.name, prompt }, 'Webhook task created');
    res.writeHead(200, { 'Content-Type': 'application/json' });
    res.end(
      JSON.stringify({ ok: true, taskId, message: 'Task triggered' }),
    );
  } catch (err) {
    log.error({ err, taskName: task.name }, 'Failed to create webhook task');
    res.writeHead(500, { 'Content-Type': 'application/json' });
    res.end(
      JSON.stringify({ ok: false, error: 'Failed to trigger task' }),
    );
  }
}
```

- [ ] **Step 2: Verify build**

Run: `npm run build`
Expected: clean compile

- [ ] **Step 3: Commit**

```bash
git add src/webhook-tasks.ts
git commit -m "feat(webhook-tasks): add core module with interpolation, matching, and handler"
```

---

### Task 3: Write unit tests for webhook-tasks

**Files:**
- Create: `src/webhook-tasks.test.ts`

- [ ] **Step 1: Write tests**

```typescript
import { describe, expect, it } from 'vitest';

import {
  extractPath,
  interpolatePrompt,
  matchWebhookTask,
  parseQueryParams,
} from './webhook-tasks.js';
import type { WebhookTaskConfig } from './webhook-tasks.js';

describe('webhook-tasks', () => {
  describe('interpolatePrompt', () => {
    it('replaces placeholders with provided params', () => {
      expect(
        interpolatePrompt('请生成 {date} 的周报', { date: '2024-01-01' }),
      ).toBe('请生成 2024-01-01 的周报');
    });

    it('leaves unmatched placeholders as-is', () => {
      expect(interpolatePrompt('hello {name}', {})).toBe('hello {name}');
    });

    it('ignores extra params not in template', () => {
      expect(
        interpolatePrompt('hello {name}', { name: 'world', extra: 'val' }),
      ).toBe('hello world');
    });

    it('replaces multiple placeholders', () => {
      expect(
        interpolatePrompt('{a} and {b}', { a: '1', b: '2' }),
      ).toBe('1 and 2');
    });

    it('handles empty template', () => {
      expect(interpolatePrompt('', { a: '1' })).toBe('');
    });
  });

  describe('matchWebhookTask', () => {
    const tasks: WebhookTaskConfig[] = [
      { name: 'daily', path: '/webhook/task/daily', prompt: 'report' },
      { name: 'cleanup', path: '/webhook/task/cleanup', prompt: 'clean' },
    ];

    it('matches exact path', () => {
      expect(matchWebhookTask('/webhook/task/daily', tasks)?.name).toBe(
        'daily',
      );
    });

    it('returns undefined for no match', () => {
      expect(matchWebhookTask('/webhook/task/unknown', tasks)).toBeUndefined();
    });

    it('returns undefined for empty tasks', () => {
      expect(matchWebhookTask('/webhook/task/daily', [])).toBeUndefined();
    });
  });

  describe('parseQueryParams', () => {
    it('parses single param', () => {
      expect(parseQueryParams('/path?date=2024-01-01')).toEqual({
        date: '2024-01-01',
      });
    });

    it('parses multiple params', () => {
      expect(parseQueryParams('/path?a=1&b=2')).toEqual({ a: '1', b: '2' });
    });

    it('returns empty for no query string', () => {
      expect(parseQueryParams('/path')).toEqual({});
    });

    it('decodes URL-encoded values', () => {
      expect(parseQueryParams('/path?q=hello%20world')).toEqual({
        q: 'hello world',
      });
    });
  });

  describe('extractPath', () => {
    it('returns path without query string', () => {
      expect(extractPath('/webhook/task/daily?date=2024-01-01')).toBe(
        '/webhook/task/daily',
      );
    });

    it('returns full URL when no query string', () => {
      expect(extractPath('/webhook/task/daily')).toBe('/webhook/task/daily');
    });
  });
});
```

- [ ] **Step 2: Run tests**

Run: `npm run test -- src/webhook-tasks.test.ts`
Expected: all tests pass

- [ ] **Step 3: Commit**

```bash
git add src/webhook-tasks.test.ts
git commit -m "test(webhook-tasks): add unit tests for interpolation, matching, parsing"
```

---

### Task 4: Integrate webhook task handler into Feishu HTTP server

**Files:**
- Modify: `src/feishu/client.ts:3177-3188` (the `handleWebhookRequest` method)

- [ ] **Step 1: Add import at top of `src/feishu/client.ts`**

Near the top imports, add:

```typescript
import { handleWebhookTaskRequest, type WebhookTaskDeps } from '../webhook-tasks.js';
```

- [ ] **Step 2: Add `webhookTaskDeps` property to FeishuClient class**

In the `FeishuClient` class (around line 130), add a new property after `private eventHandlers`:

```typescript
private webhookTaskDeps: WebhookTaskDeps | null = null;

/** Set dependencies for webhook task triggering */
setWebhookTaskDeps(deps: WebhookTaskDeps): void {
  this.webhookTaskDeps = deps;
}
```

- [ ] **Step 3: Modify `handleWebhookRequest` to delegate non-Feishu paths**

Replace the current rejection block (lines 3183-3188):

```typescript
    // Only accept POST to the configured path
    if (req.method !== 'POST' || req.url !== path) {
      res.writeHead(404, { 'Content-Type': 'text/plain' });
      res.end('Not Found');
      return;
    }
```

With:

```typescript
    // Only accept POST to the configured path
    if (req.method !== 'POST' || req.url !== path) {
      // Delegate to webhook task handler if available
      if (this.webhookTaskDeps && req.method === 'POST') {
        handleWebhookTaskRequest(req, res, this.webhookTaskDeps);
        return;
      }
      res.writeHead(404, { 'Content-Type': 'text/plain' });
      res.end('Not Found');
      return;
    }
```

- [ ] **Step 4: Verify build**

Run: `npm run build`
Expected: clean compile

- [ ] **Step 5: Commit**

```bash
git add src/feishu/client.ts
git commit -m "feat(webhook-tasks): integrate handler into Feishu HTTP server"
```

---

### Task 5: Wire up webhook task deps in index.ts

**Files:**
- Modify: `src/index.ts`

- [ ] **Step 1: Find the Feishu channel instance and set webhook task deps**

In `src/index.ts`, after the `startSchedulerLoop(...)` call (around line 847), add the following block. This finds the Feishu channel and wires up the main group lookup:

```typescript
  // Wire webhook task deps into Feishu channel's HTTP server
  const feishuChannel = channels.find(
    (ch): ch is import('./channels/feishu.js').FeishuChannel =>
      ch.name === 'feishu',
  );
  if (feishuChannel) {
    feishuChannel.client.setWebhookTaskDeps({
      getMainGroup: () => {
        const entry = Object.entries(registeredGroups).find(
          ([, g]) => g.isMain,
        );
        if (!entry) return undefined;
        return { jid: entry[0], folder: entry[1].folder };
      },
      registeredGroups: () => registeredGroups,
    });
  }
```

- [ ] **Step 2: Verify build**

Run: `npm run build`
Expected: clean compile

- [ ] **Step 3: Run all tests**

Run: `npm run test`
Expected: all tests pass

- [ ] **Step 4: Commit**

```bash
git add src/index.ts
git commit -m "feat(webhook-tasks): wire up deps in main process"
```

---

### Task 6: Create default config file

**Files:**
- Create: `webhook-tasks.json`

- [ ] **Step 1: Create `webhook-tasks.json`**

```json
{
  "tasks": []
}
```

- [ ] **Step 2: Commit**

```bash
git add webhook-tasks.json
git commit -m "feat(webhook-tasks): add default empty config file"
```

---

### Task 7: Build, deploy, and verify

- [ ] **Step 1: Build**

Run: `npm run build`

- [ ] **Step 2: Restart service**

Run: `systemctl --user restart nanoclaw-test`

- [ ] **Step 3: Verify service is running**

Run: `systemctl --user status nanoclaw-test`

- [ ] **Step 4: Test with curl (if webhook server is active)**

Add a test task to `webhook-tasks.json` and call it:

```bash
# Add test task
cat > webhook-tasks.json << 'EOF'
{
  "tasks": [
    {
      "name": "test",
      "path": "/webhook/task/test",
      "prompt": "这是一个测试任务",
      "context_mode": "isolated"
    }
  ]
}
EOF

# Trigger it
curl -X POST http://127.0.0.1:8080/webhook/task/test
```

Expected response: `{"ok":true,"taskId":"webhook_test_...","message":"Task triggered"}`
