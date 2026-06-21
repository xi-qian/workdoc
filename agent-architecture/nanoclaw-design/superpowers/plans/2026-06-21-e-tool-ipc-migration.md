# 计划 E:Tool IPC 迁移实施计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**前置条件:** Plan A-D 已完成。包含 `tools.db` 在内的 runtime DB 已存在。Spawner 和 live poll 循环可工作。skill 通过 manifest 加载。

**目标:** 将工具调用从传统的文件 IPC(`data/ipc/<group>/...`)迁移到基于 `tools.db` 的 DB-backed RPC。Run 进程写入工具请求;host 侧的 tool worker 负责领取、授权、执行 channel/control-plane API,并写回结果。传统文件 IPC 路径被废弃,不再为新 run 创建。

**架构:** `src/tools/registry.ts` 将工具名映射到 handler。`src/tools/worker-pool.ts` 是一个长期运行的轮询器,遍历每个活跃 run 的 `tools.db`,领取 pending 行,分发给 handler,写入结果。`container/agent-runner/src/tool-bridge.ts` 暴露一个 MCP server,从 provider 读取 pending 的工具调用,将它们作为 `tools.db` 行写入,然后轮询结果。授权发生在 host 侧:每个 handler 根据策略(main/self、sender allowlist、approval allowlist 等)校验源 run 的身份(`tenant_id`、`agent_id`、`group_folder`、`run_id`)。

**技术栈:** TypeScript NodeNext ESM,better-sqlite3,vitest,已有的 `@modelcontextprotocol/sdk`(假设可用;如不可用,在 Task 3 中安装)。

---

## 文件结构

### 新建

| 路径 | 职责 |
|------|----------------|
| `src/tools/types.ts` | `ToolRequest`, `ToolResult`, `ToolHandler` 类型 |
| `src/tools/registry.ts` | 工具名 → handler 映射 |
| `src/tools/registry.test.ts` | Registry 测试 |
| `src/tools/worker-pool.ts` | 周期性轮询活跃 run 的 tools.db |
| `src/tools/worker-pool.test.ts` | 使用 fake DB 的 worker pool 测试 |
| `src/tools/claim.ts` | 原子 claim + lease helper |
| `src/tools/claim.test.ts` | Claim 测试 |
| `src/tools/handlers/send-message.ts` | `send_message` handler |
| `src/tools/handlers/scheduling.ts` | `schedule_task` 及相关工具 |
| `src/tools/handlers/session.ts` | `new_session` handler |
| `src/tools/handlers/groups.ts` | `register_group`, `refresh_groups` |
| `src/tools/handlers/feishu-docs.ts` | Feishu 文档工具 |
| `src/tools/handlers/feishu-bitable.ts` | Feishu 多维表格工具 |
| `src/tools/handlers/feishu-task.ts` | Feishu 任务工具 |
| `src/tools/handlers/feishu-approval.ts` | 审批工具 |
| `src/tools/handlers/feishu-files.ts` | 上传/下载工具 |
| `src/tools/handlers/feishu-card.ts` | 卡片/富文本工具 |
| `src/tools/handlers/feishu-p2p.ts` | P2P/用户查询工具 |
| `src/tools/handlers/index.ts` | 注册所有 handler |
| `src/tools/authz.ts` | 授权 helper(main/self、allowlist) |
| `src/tools/authz.test.ts` | 授权测试 |
| `container/agent-runner/src/tool-bridge.ts` | 桥接 tools.db 的 MCP server |
| `container/agent-runner/src/tool-bridge.test.ts` | Bridge 测试 |

### 修改

| 路径 | 原因 |
|------|-----|
| `src/ipc.ts` | 停止为已迁移的 group 创建新的文件 IPC 队列。对新 run 保留传统 watcher 为 no-op。 |
| `container/agent-runner/src/ipc-mcp-stdio.ts` | 将传统文件 IPC MCP 工具替换为对 tool-bridge 的调用。 |

### 不要触碰

- 迁移脚本 — Plan F。
- 验收测试 — Plan G。

---

## 全局不变量

- NodeNext ESM,全程 TDD。
- 每个 handler 必须在做任何工作前调用 `authorize()`;拒绝产生结构化错误结果,绝不抛出异常。
- 工具结果绝不包含 host 路径或原始 channel 凭据。文件工具返回 runtime 相对路径。
- Handler 在给定输入下是纯函数 — 无模块级可变状态。

---

## Task 1: Tool IPC 类型

**文件:**
- 新建:`src/tools/types.ts`
- 新建:`src/tools/types.test.ts`

- [ ] **第 1 步:写失败的测试**

创建 `src/tools/types.test.ts`:

```typescript
import { describe, expect, it } from 'vitest';

import type {
  ToolError,
  ToolHandler,
  ToolRequest,
  ToolRequestContext,
  ToolResult,
} from './types.js';

describe('tool types', () => {
  it('ToolRequest has identity + payload', () => {
    const r: ToolRequest = {
      rowId: 1,
      tenantId: 'acme',
      agentId: 'finance',
      groupFolder: 'main',
      runId: 'r1',
      toolName: 'send_message',
      payload: { chat_jid: 'feishu:x', text: 'hi' },
      createdAt: '2026-06-21T00:00:00Z',
    };
    expect(r.toolName).toBe('send_message');
  });

  it('ToolResult success has value', () => {
    const r: ToolResult = {
      rowId: 1,
      status: 'completed',
      result: { ok: true },
      completedAt: '2026-06-21T00:00:01Z',
    };
    expect(r.status).toBe('completed');
  });

  it('ToolResult error carries denial reason', () => {
    const r: ToolResult = {
      rowId: 1,
      status: 'error',
      error: { code: 'denied', message: 'not main group' },
      completedAt: '2026-06-21T00:00:01Z',
    };
    expect(r.error?.code).toBe('denied');
  });

  it('ToolHandler is a function type', () => {
    const h: ToolHandler = async (_ctx, _payload) => ({ ok: true });
    expect(typeof h).toBe('function');
  });

  it('ToolRequestContext carries identity for authz', () => {
    const ctx: ToolRequestContext = {
      tenantId: 'acme',
      agentId: 'finance',
      groupFolder: 'main',
      runId: 'r1',
      isMain: false,
      channelType: 'feishu',
      chatJid: 'feishu:oc_x',
    };
    expect(ctx.isMain).toBe(false);
  });
});
```

- [ ] **第 2 步:运行以验证失败**

运行:`npm test -- src/tools/types.test.ts`
预期:FAIL。

- [ ] **第 3 步:实现类型**

创建 `src/tools/types.ts`:

```typescript
export interface ToolRequestContext {
  tenantId: string;
  agentId: string;
  groupFolder: string;
  runId: string;
  isMain: boolean;
  channelType: string;
  chatJid: string;
}

export interface ToolRequest<P = Record<string, unknown>> {
  rowId: number;
  tenantId: string;
  agentId: string;
  groupFolder: string;
  runId: string;
  toolName: string;
  payload: P;
  createdAt: string;
}

export interface ToolError {
  code: 'denied' | 'not_found' | 'invalid_args' | 'upstream' | 'timeout' | 'internal';
  message: string;
  /** Optional structured details (no secrets, no host paths). */
  details?: Record<string, unknown>;
}

export interface ToolResult<R = unknown> {
  rowId: number;
  status: 'completed' | 'error';
  result?: R;
  error?: ToolError;
  completedAt: string;
}

export type ToolHandler<P = Record<string, unknown>, R = unknown> = (
  ctx: ToolRequestContext,
  payload: P,
) => Promise<R>;
```

- [ ] **第 4 步:运行测试以验证通过**

运行:`npm test -- src/tools/types.test.ts`
预期:PASS(5 个测试)。

- [ ] **第 5 步:提交**

```bash
git add src/tools/types.ts src/tools/types.test.ts
git commit -m "feat(tools): add tool IPC type definitions"
```

---

## Task 2: Tool registry

**文件:**
- 新建:`src/tools/registry.ts`
- 新建:`src/tools/registry.test.ts`

- [ ] **第 1 步:写失败的测试**

创建 `src/tools/registry.test.ts`:

```typescript
import { describe, expect, it } from 'vitest';

import {
  getToolHandler,
  registerToolHandler,
  resetToolHandlers,
  listToolNames,
} from './registry.js';

describe('tool registry', () => {
  it('registers and retrieves handlers by name', () => {
    resetToolHandlers();
    const h = async () => ({ ok: true });
    registerToolHandler('send_message', h);
    expect(getToolHandler('send_message')).toBe(h);
  });

  it('listToolNames returns all registered names', () => {
    resetToolHandlers();
    registerToolHandler('a', async () => null);
    registerToolHandler('b', async () => null);
    expect(listToolNames().sort()).toEqual(['a', 'b']);
  });

  it('returns undefined for unknown tool', () => {
    resetToolHandlers();
    expect(getToolHandler('unknown')).toBeUndefined();
  });
});
```

- [ ] **第 2 步:运行以验证失败**

运行:`npm test -- src/tools/registry.test.ts`
预期:FAIL。

- [ ] **第 3 步:实现 registry**

创建 `src/tools/registry.ts`:

```typescript
import type { ToolHandler } from './types.js';

const handlers = new Map<string, ToolHandler>();

export function registerToolHandler(name: string, handler: ToolHandler): void {
  handlers.set(name, handler);
}

export function getToolHandler(name: string): ToolHandler | undefined {
  return handlers.get(name);
}

export function listToolNames(): string[] {
  return [...handlers.keys()];
}

export function resetToolHandlers(): void {
  handlers.clear();
}
```

- [ ] **第 4 步:运行测试以验证通过**

运行:`npm test -- src/tools/registry.test.ts`
预期:PASS(3 个测试)。

- [ ] **第 5 步:提交**

```bash
git add src/tools/registry.ts src/tools/registry.test.ts
git commit -m "feat(tools): add tool name → handler registry"
```

---

## Task 3: 授权 helper

**文件:**
- 新建:`src/tools/authz.ts`
- 新建:`src/tools/authz.test.ts`

- [ ] **第 1 步:写失败的测试**

创建 `src/tools/authz.test.ts`:

```typescript
import { describe, expect, it } from 'vitest';

import {
  authorizeMainOnly,
  authorizeMainOrSelf,
  AuthorizationError,
} from './authz.js';
import type { ToolRequestContext } from './types.js';

const mainCtx: ToolRequestContext = {
  tenantId: 't', agentId: 'a', groupFolder: 'main', runId: 'r',
  isMain: true, channelType: 'feishu', chatJid: 'feishu:g1',
};

const nonMainCtx: ToolRequestContext = {
  tenantId: 't', agentId: 'a', groupFolder: 'g2', runId: 'r',
  isMain: false, channelType: 'feishu', chatJid: 'feishu:g2',
};

describe('authorizeMainOnly', () => {
  it('passes for main group', () => {
    expect(() => authorizeMainOnly(mainCtx)).not.toThrow();
  });
  it('throws AuthorizationError for non-main', () => {
    expect(() => authorizeMainOnly(nonMainCtx)).toThrow(AuthorizationError);
  });
});

describe('authorizeMainOrSelf', () => {
  it('passes for main', () => {
    expect(() => authorizeMainOrSelf(mainCtx, 'feishu:g1')).not.toThrow();
  });
  it('passes when chat_jid matches source', () => {
    expect(() => authorizeMainOrSelf(nonMainCtx, 'feishu:g2')).not.toThrow();
  });
  it('throws when chat_jid differs and not main', () => {
    expect(() => authorizeMainOrSelf(nonMainCtx, 'feishu:other')).toThrow(AuthorizationError);
  });
});
```

- [ ] **第 2 步:运行以验证失败**

运行:`npm test -- src/tools/authz.test.ts`
预期:FAIL。

- [ ] **第 3 步:实现 authz**

创建 `src/tools/authz.ts`:

```typescript
import type { ToolRequestContext } from './types.js';

export class AuthorizationError extends Error {
  readonly code = 'denied' as const;
  constructor(message: string) {
    super(message);
    this.name = 'AuthorizationError';
  }
}

/**
 * Require that the caller is the main group. Use for operations that affect
 * multiple groups or tenant-wide state (e.g., register_group).
 */
export function authorizeMainOnly(ctx: ToolRequestContext): void {
  if (!ctx.isMain) {
    throw new AuthorizationError(
      `tool requires main group; caller is ${ctx.groupFolder}`,
    );
  }
}

/**
 * Require that the caller is either the main group OR is acting on its own
 * chat (target chat_jid == source chat_jid). Use for send_message,
 * schedule_task, etc.
 */
export function authorizeMainOrSelf(ctx: ToolRequestContext, targetChatJid: string): void {
  if (ctx.isMain) return;
  if (ctx.chatJid === targetChatJid) return;
  throw new AuthorizationError(
    `non-main group ${ctx.groupFolder} cannot act on ${targetChatJid}`,
  );
}
```

- [ ] **第 4 步:运行测试以验证通过**

运行:`npm test -- src/tools/authz.test.ts`
预期:PASS(5 个测试)。

- [ ] **第 5 步:提交**

```bash
git add src/tools/authz.ts src/tools/authz.test.ts
git commit -m "feat(tools): add main/self authorization helpers"
```

---

## Task 4: Tool claim helper

**文件:**
- 新建:`src/tools/claim.ts`
- 新建:`src/tools/claim.test.ts`

- [ ] **第 1 步:写失败的测试**

创建 `src/tools/claim.test.ts`:

```typescript
import fs from 'node:fs';
import os from 'node:os';
import path from 'node:path';

import { afterEach, beforeEach, describe, expect, it } from 'vitest';

import { claimNextRequest, completeRequest, ToolDb } from './claim.js';

let dir: string;
let db: ToolDb;

beforeEach(() => {
  dir = fs.mkdtempSync(path.join(os.tmpdir(), 'nc-claim-'));
  db = new ToolDb(path.join(dir, 'tools.db'));
  db.init();
});

afterEach(() => {
  db.close();
  fs.rmSync(dir, { recursive: true, force: true });
});

describe('claim', () => {
  it('claims a pending row and marks it claimed', () => {
    const rowId = db.insert({
      tenant_id: 't', agent_id: 'a', group_folder: 'g', run_id: 'r',
      tool_name: 'send_message', payload: {},
    });
    const claimed = claimNextRequest(db, 'worker-1');
    expect(claimed?.row_id).toBe(rowId);
    expect(claimed?.status).toBe('claimed');
  });

  it('returns undefined when no pending rows', () => {
    expect(claimNextRequest(db, 'worker-1')).toBeUndefined();
  });

  it('completeRequest writes result and updates status', () => {
    const rowId = db.insert({
      tenant_id: 't', agent_id: 'a', group_folder: 'g', run_id: 'r',
      tool_name: 'send_message', payload: {},
    });
    claimNextRequest(db, 'w1');
    completeRequest(db, rowId, { ok: true });
    const rows = db.all();
    expect(rows[0].status).toBe('completed');
    expect(JSON.parse(rows[0].result!).ok).toBe(true);
  });

  it('completeRequest writes error result', () => {
    const rowId = db.insert({
      tenant_id: 't', agent_id: 'a', group_folder: 'g', run_id: 'r',
      tool_name: 'send_message', payload: {},
    });
    claimNextRequest(db, 'w1');
    completeRequest(db, rowId, undefined, { code: 'denied', message: 'no' });
    const rows = db.all();
    expect(rows[0].status).toBe('error');
    expect(JSON.parse(rows[0].error!).code).toBe('denied');
  });

  it('two workers cannot claim the same row', () => {
    db.insert({
      tenant_id: 't', agent_id: 'a', group_folder: 'g', run_id: 'r',
      tool_name: 'send_message', payload: {},
    });
    const a = claimNextRequest(db, 'w1');
    const b = claimNextRequest(db, 'w2');
    expect(a).toBeTruthy();
    expect(b).toBeUndefined();
  });
});
```

- [ ] **第 2 步:运行以验证失败**

运行:`npm test -- src/tools/claim.test.ts`
预期:FAIL。

- [ ] **第 3 步:实现 claim**

创建 `src/tools/claim.ts`:

```typescript
import Database from 'better-sqlite3';
import fs from 'node:fs';
import type { ToolError } from './types.js';

export interface ToolRow {
  row_id: number;
  tenant_id: string;
  agent_id: string;
  group_folder: string;
  run_id: string;
  tool_name: string;
  payload: string;       // JSON
  status: string;
  result: string | null; // JSON
  error: string | null;  // JSON
  created_at: string;
  claimed_at: string | null;
  completed_at: string | null;
  lease_owner: string | null;
}

export class ToolDb {
  private db: Database.Database;
  constructor(readonly path: string) {}
  init(): void {
    if (!this.db) this.db = this.open();
    this.applySchema();
  }
  private open(): Database.Database {
    const db = new Database(this.path);
    db.pragma('journal_mode = WAL');
    db.pragma('busy_timeout = 5000');
    return db;
  }
  private applySchema(): void {
    this.db.exec(`
      CREATE TABLE IF NOT EXISTS tool_requests (
        row_id INTEGER PRIMARY KEY AUTOINCREMENT,
        tenant_id TEXT NOT NULL,
        agent_id TEXT NOT NULL,
        group_folder TEXT NOT NULL,
        run_id TEXT NOT NULL,
        tool_name TEXT NOT NULL,
        payload TEXT NOT NULL,
        status TEXT NOT NULL DEFAULT 'pending',
        result TEXT,
        error TEXT,
        created_at TEXT NOT NULL,
        claimed_at TEXT,
        completed_at TEXT,
        lease_owner TEXT
      );
      CREATE INDEX IF NOT EXISTS idx_tool_status ON tool_requests(status, created_at);
      CREATE INDEX IF NOT EXISTS idx_tool_run ON tool_requests(tenant_id, agent_id, run_id);
    `);
  }
  insert(args: {
    tenant_id: string; agent_id: string; group_folder: string; run_id: string;
    tool_name: string; payload: unknown;
  }): number {
    const r = this.db.prepare(
      `INSERT INTO tool_requests
        (tenant_id, agent_id, group_folder, run_id, tool_name, payload, status, created_at)
       VALUES (@tenant_id, @agent_id, @group_folder, @run_id, @tool_name, @payload, 'pending', @created_at)`,
    ).run({
      ...args,
      payload: JSON.stringify(args.payload),
      created_at: new Date().toISOString(),
    });
    return Number(r.lastInsertRowid);
  }
  all(): ToolRow[] {
    return this.db.prepare(`SELECT * FROM tool_requests ORDER BY row_id ASC`).all() as ToolRow[];
  }
  raw(): Database.Database {
    return this.db;
  }
  close(): void {
    this.db?.close();
  }
}

/**
 * Atomically claim the next pending row. Uses a transaction + status update
 * to avoid races between workers.
 */
export function claimNextRequest(db: ToolDb, leaseOwner: string): ToolRow | undefined {
  const raw = db.raw();
  const txn = raw.transaction(() => {
    const row = raw
      .prepare(`SELECT * FROM tool_requests WHERE status='pending' ORDER BY row_id ASC LIMIT 1`)
      .get() as ToolRow | undefined;
    if (!row) return undefined;
    raw.prepare(
      `UPDATE tool_requests SET status='claimed', claimed_at=?, lease_owner=? WHERE row_id=? AND status='pending'`,
    ).run(new Date().toISOString(), leaseOwner, row.row_id);
    return raw.prepare(`SELECT * FROM tool_requests WHERE row_id=?`).get(row.row_id) as ToolRow;
  });
  return txn();
}

export function completeRequest(
  db: ToolDb,
  rowId: number,
  result?: unknown,
  error?: ToolError,
): void {
  const status = error ? 'error' : 'completed';
  db.raw()
    .prepare(
      `UPDATE tool_requests
       SET status=?, result=?, error=?, completed_at=?
       WHERE row_id=?`,
    )
    .run(
      status,
      result !== undefined ? JSON.stringify(result) : null,
      error ? JSON.stringify(error) : null,
      new Date().toISOString(),
      rowId,
    );
}
```

- [ ] **第 4 步:运行测试以验证通过**

运行:`npm test -- src/tools/claim.test.ts`
预期:PASS(5 个测试)。

- [ ] **第 5 步:提交**

```bash
git add src/tools/claim.ts src/tools/claim.test.ts
git commit -m "feat(tools): add atomic claim/complete helpers for tools.db"
```

---

## Task 5: Worker pool — 轮询活跃 run 并分发

**文件:**
- 新建:`src/tools/worker-pool.ts`
- 新建:`src/tools/worker-pool.test.ts`

- [ ] **第 1 步:写失败的测试**

创建 `src/tools/worker-pool.test.ts`:

```typescript
import fs from 'node:fs';
import os from 'node:os';
import path from 'node:path';

import { afterEach, beforeEach, describe, expect, it } from 'vitest';

import { WorkerPool } from './worker-pool.js';
import { ToolDb } from './claim.js';
import { registerToolHandler, resetToolHandlers } from './registry.js';
import type { ToolRequestContext } from './types.js';

let dataDir: string;

beforeEach(() => {
  dataDir = fs.mkdtempSync(path.join(os.tmpdir(), 'nc-pool-'));
  resetToolHandlers();
});

afterEach(() => {
  fs.rmSync(dataDir, { recursive: true, force: true });
});

function setupToolsDb(t: string, a: string, g: string): ToolDb {
  const dir = path.join(dataDir, 'runtime', t, a, g, 'live');
  fs.mkdirSync(dir, { recursive: true });
  const db = new ToolDb(path.join(dir, 'tools.db'));
  db.init();
  return db;
}

describe('WorkerPool', () => {
  it('claims and dispatches to a registered handler', async () => {
    const seen: string[] = [];
    registerToolHandler('send_message', async (ctx, payload) => {
      seen.push(`${ctx.tenantId}/${ctx.agentId}:${(payload as { text: string }).text}`);
      return { ok: true };
    });

    const t = 't1', a = 'a1', g = 'g1';
    const db = setupToolsDb(t, a, g);
    db.insert({
      tenant_id: t, agent_id: a, group_folder: g, run_id: 'r1',
      tool_name: 'send_message', payload: { text: 'hi', chat_jid: 'feishu:x' },
    });

    const ctx: ToolRequestContext = {
      tenantId: t, agentId: a, groupFolder: g, runId: 'r1',
      isMain: false, channelType: 'feishu', chatJid: 'feishu:x',
    };
    const pool = new WorkerPool({
      dataDir,
      resolveContext: async (row) => ({ ...ctx, runId: row.run_id }),
      pollIntervalMs: 10,
    });
    pool.start();
    await new Promise((r) => setTimeout(r, 100));
    await pool.stop();

    expect(seen).toEqual([`${t}/${a}:hi`]);
    const rows = db.all();
    expect(rows[0].status).toBe('completed');
  });

  it('writes error when handler throws', async () => {
    registerToolHandler('fail', async () => {
      throw new Error('boom');
    });
    const t = 't1', a = 'a1', g = 'g1';
    const db = setupToolsDb(t, a, g);
    db.insert({
      tenant_id: t, agent_id: a, group_folder: g, run_id: 'r1',
      tool_name: 'fail', payload: {},
    });
    const pool = new WorkerPool({
      dataDir,
      resolveContext: async () => ({
        tenantId: t, agentId: a, groupFolder: g, runId: 'r1',
        isMain: false, channelType: 'feishu', chatJid: 'feishu:x',
      }),
      pollIntervalMs: 10,
    });
    pool.start();
    await new Promise((r) => setTimeout(r, 100));
    await pool.stop();
    const rows = db.all();
    expect(rows[0].status).toBe('error');
    expect(JSON.parse(rows[0].error!).message).toBe('boom');
  });

  it('writes denied error when handler throws AuthorizationError', async () => {
    const { AuthorizationError } = await import('./authz.js');
    registerToolHandler('denied', async () => {
      throw new AuthorizationError('no');
    });
    const t = 't1', a = 'a1', g = 'g1';
    const db = setupToolsDb(t, a, g);
    db.insert({
      tenant_id: t, agent_id: a, group_folder: g, run_id: 'r1',
      tool_name: 'denied', payload: {},
    });
    const pool = new WorkerPool({
      dataDir,
      resolveContext: async () => ({
        tenantId: t, agentId: a, groupFolder: g, runId: 'r1',
        isMain: false, channelType: 'feishu', chatJid: 'feishu:x',
      }),
      pollIntervalMs: 10,
    });
    pool.start();
    await new Promise((r) => setTimeout(r, 100));
    await pool.stop();
    const rows = db.all();
    expect(rows[0].status).toBe('error');
    expect(JSON.parse(rows[0].error!).code).toBe('denied');
  });
});
```

- [ ] **第 2 步:运行以验证失败**

运行:`npm test -- src/tools/worker-pool.test.ts`
预期:FAIL。

- [ ] **第 3 步:实现 worker pool**

创建 `src/tools/worker-pool.ts`:

```typescript
import fs from 'node:fs';
import path from 'node:path';

import { listActiveRuns } from '../db/active-runs.js';
import { AuthorizationError } from './authz.js';
import { ToolDb, ToolRow, claimNextRequest, completeRequest } from './claim.js';
import { getToolHandler } from './registry.js';
import type { ToolError, ToolRequestContext } from './types.js';

export interface WorkerPoolOpts {
  dataDir: string;
  resolveContext: (row: ToolRow) => Promise<ToolRequestContext>;
  pollIntervalMs?: number;
  workerId?: string;
}

export class WorkerPool {
  private timer?: NodeJS.Timeout;
  private readonly workerId: string;
  private readonly pollIntervalMs: number;
  private readonly toolsDbCache = new Map<string, ToolDb>();

  constructor(private readonly opts: WorkerPoolOpts) {
    this.workerId = opts.workerId ?? `worker-${process.pid}`;
    this.pollIntervalMs = opts.pollIntervalMs ?? 1000;
  }

  start(): void {
    if (this.timer) return;
    this.timer = setInterval(() => {
      void this.tick().catch(() => {});
    }, this.pollIntervalMs);
  }

  async stop(): Promise<void> {
    if (this.timer) {
      clearInterval(this.timer);
      this.timer = undefined;
    }
    for (const db of this.toolsDbCache.values()) db.close();
    this.toolsDbCache.clear();
  }

  private async tick(): Promise<void> {
    // Walk active runs, open their tools.db, claim one row per run per tick.
    const runtimeRoot = path.join(this.opts.dataDir, 'runtime');
    let tenants: string[] = [];
    try {
      tenants = await fs.promises.readdir(runtimeRoot);
    } catch {
      return;
    }
    for (const tenant of tenants) {
      let agents: string[] = [];
      try {
        agents = await fs.promises.readdir(path.join(runtimeRoot, tenant));
      } catch {
        continue;
      }
      for (const agent of agents) {
        const runs = listActiveRuns(this.opts.dataDir, tenant, agent).filter(
          (r) => r.status === 'active',
        );
        for (const run of runs) {
          await this.processRun(tenant, agent, run.group_folder, run.run_id, run.mode);
        }
      }
    }
  }

  private async processRun(
    tenant: string,
    agent: string,
    group: string,
    runId: string,
    mode: string,
  ): Promise<void> {
    const sub = mode === 'live' ? 'live' : `runs/${runId}`;
    const toolsDbPath = path.join(
      this.opts.dataDir, 'runtime', tenant, agent, group, sub, 'tools.db',
    );
    if (!fs.existsSync(toolsDbPath)) return;

    const db = this.toolsDbCache.get(toolsDbPath) ?? new ToolDb(toolsDbPath);
    if (!this.toolsDbCache.has(toolsDbPath)) {
      db.init();
      this.toolsDbCache.set(toolsDbPath, db);
    }

    // Process up to 5 rows per tick per run to avoid starvation.
    for (let i = 0; i < 5; i++) {
      const claimed = claimNextRequest(db, this.workerId);
      if (!claimed) break;
      await this.dispatch(claimed, db);
    }
  }

  private async dispatch(row: ToolRow, db: ToolDb): Promise<void> {
    const handler = getToolHandler(row.tool_name);
    if (!handler) {
      completeRequest(db, row.row_id, undefined, {
        code: 'not_found',
        message: `unknown tool: ${row.tool_name}`,
      });
      return;
    }

    const ctx = await this.opts.resolveContext(row);
    const payload = JSON.parse(row.payload);

    try {
      const result = await handler(ctx, payload);
      completeRequest(db, row.row_id, result);
    } catch (err) {
      const e = err as Error & { code?: string };
      const toolError: ToolError = e instanceof AuthorizationError
        ? { code: 'denied', message: e.message }
        : { code: 'internal', message: e.message };
      completeRequest(db, row.row_id, undefined, toolError);
    }
  }
}
```

- [ ] **第 4 步:运行测试以验证通过**

运行:`npm test -- src/tools/worker-pool.test.ts`
预期:PASS(3 个测试)。

- [ ] **第 5 步:提交**

```bash
git add src/tools/worker-pool.ts src/tools/worker-pool.test.ts
git commit -m "feat(tools): add worker pool that polls active runs' tools.db"
```

---

## Task 6: `send_message` handler

**文件:**
- 新建:`src/tools/handlers/send-message.ts`
- 新建:`src/tools/handlers/send-message.test.ts`
- 新建:`src/tools/handlers/index.ts`

- [ ] **第 1 步:写失败的测试**

创建 `src/tools/handlers/send-message.test.ts`:

```typescript
import { describe, expect, it } from 'vitest';

import { createSendMessageHandler } from './send-message.js';

describe('send_message handler', () => {
  it('authorizes self-send and calls channel.sendMessage', async () => {
    const sent: { jid: string; text: string }[] = [];
    const handler = createSendMessageHandler({
      getChannel: () => ({
        async sendMessage(jid: string, text: string) {
          sent.push({ jid, text });
        },
      }),
    });
    const result = await handler(
      {
        tenantId: 't', agentId: 'a', groupFolder: 'g', runId: 'r',
        isMain: false, channelType: 'feishu', chatJid: 'feishu:g',
      },
      { chat_jid: 'feishu:g', text: 'hello' },
    );
    expect(result).toEqual({ ok: true });
    expect(sent).toEqual([{ jid: 'feishu:g', text: 'hello' }]);
  });

  it('denies cross-chat send for non-main', async () => {
    const handler = createSendMessageHandler({ getChannel: () => null });
    await expect(
      handler(
        {
          tenantId: 't', agentId: 'a', groupFolder: 'g', runId: 'r',
          isMain: false, channelType: 'feishu', chatJid: 'feishu:g',
        },
        { chat_jid: 'feishu:other', text: 'x' },
      ),
    ).rejects.toThrow(/cannot act on/);
  });

  it('allows main group to send to any chat', async () => {
    let called = false;
    const handler = createSendMessageHandler({
      getChannel: () => ({ async sendMessage() { called = true; } }),
    });
    await handler(
      {
        tenantId: 't', agentId: 'a', groupFolder: 'main', runId: 'r',
        isMain: true, channelType: 'feishu', chatJid: 'feishu:main',
      },
      { chat_jid: 'feishu:anywhere', text: 'hi' },
    );
    expect(called).toBe(true);
  });

  it('errors when channel is unavailable', async () => {
    const handler = createSendMessageHandler({ getChannel: () => null });
    await expect(
      handler(
        {
          tenantId: 't', agentId: 'a', groupFolder: 'main', runId: 'r',
          isMain: true, channelType: 'feishu', chatJid: 'feishu:m',
        },
        { chat_jid: 'feishu:m', text: 'hi' },
      ),
    ).rejects.toThrow(/channel not available/);
  });
});
```

- [ ] **第 2 步:运行以验证失败**

运行:`npm test -- src/tools/handlers/send-message.test.ts`
预期:FAIL。

- [ ] **第 3 步:实现 handler**

创建 `src/tools/handlers/send-message.ts`:

```typescript
import type { Channel } from '../../types.js';
import type { ToolHandler, ToolRequestContext } from '../types.js';
import { authorizeMainOrSelf } from '../authz.js';

export interface SendMessageEnv {
  getChannel: (ctx: ToolRequestContext) => Pick<Channel, 'sendMessage'> | null;
}

export interface SendMessagePayload {
  chat_jid: string;
  text: string;
  /** Optional card JSON for rich messages; format is channel-specific. */
  card?: unknown;
}

export function createSendMessageHandler(env: SendMessageEnv): ToolHandler<SendMessagePayload> {
  return async (ctx, payload) => {
    authorizeMainOrSelf(ctx, payload.chat_jid);
    const channel = env.getChannel(ctx);
    if (!channel) {
      throw new Error(`channel not available for ${ctx.channelType}`);
    }
    await channel.sendMessage(payload.chat_jid, payload.text);
    return { ok: true };
  };
}
```

创建 `src/tools/handlers/index.ts`(随着 handler 增加会逐步扩展):

```typescript
import { lookupChannel } from '../../channels/lookup.js';
import { registerToolHandler } from '../registry.js';
import { createSendMessageHandler } from './send-message.js';

let registered = false;

/**
 * Register all tool handlers against the global tool registry. Idempotent.
 * Call once at startup, after channel bootstrap.
 */
export function registerAllToolHandlers(): void {
  if (registered) return;
  registerToolHandler('send_message', createSendMessageHandler({
    getChannel: (ctx) => {
      const ch = lookupChannel(ctx.tenantId, ctx.agentId, ctx.channelType as never);
      if (!ch) return null;
      return { sendMessage: (jid, text) => ch.sendMessage(jid, text) };
    },
  }));
  registered = true;
}
```

- [ ] **第 4 步:运行测试以验证通过**

运行:`npm test -- src/tools/handlers/send-message.test.ts`
预期:PASS(4 个测试)。

- [ ] **第 5 步:提交**

```bash
git add src/tools/handlers/send-message.ts src/tools/handlers/send-message.test.ts src/tools/handlers/index.ts
git commit -m "feat(tools): add send_message handler with main/self authz"
```

---

## Task 7: 调度 handler(schedule_task、list_tasks、pause/resume/cancel/update)

**文件:**
- 新建:`src/tools/handlers/scheduling.ts`
- 新建:`src/tools/handlers/scheduling.test.ts`
- 修改:`src/tools/handlers/index.ts`

- [ ] **第 1 步:写失败的测试**

创建 `src/tools/handlers/scheduling.test.ts`:

```typescript
import { describe, expect, it } from 'vitest';

import { createSchedulingHandlers } from './scheduling.js';
import type { ToolRequestContext } from '../types.js';

function fakeEnv() {
  const tasks: Record<string, any>[] = [];
  return {
    tasks,
    insert: async (t: any) => { tasks.push({ ...t, id: 'tsk-' + (tasks.length + 1) }); return tasks[tasks.length - 1]; },
    list: async () => tasks,
    update: async (id: string, patch: any) => {
      const i = tasks.findIndex((t) => t.id === id);
      if (i < 0) throw new Error('not found');
      tasks[i] = { ...tasks[i], ...patch };
    },
  };
}

const ctx = (isMain: boolean): ToolRequestContext => ({
  tenantId: 't', agentId: 'a', groupFolder: 'main', runId: 'r',
  isMain, channelType: 'feishu', chatJid: 'feishu:g',
});

describe('scheduling handlers', () => {
  it('schedule_task creates task', async () => {
    const env = fakeEnv();
    const h = createSchedulingHandlers(env);
    const out = await h.scheduleTask(ctx(true), {
      prompt: 'do thing', schedule_type: 'cron', schedule_value: '* * * * *',
      chat_jid: 'feishu:g', context_mode: 'group',
    });
    expect((out as { id: string }).id).toMatch(/^tsk-/);
    expect(env.tasks).toHaveLength(1);
  });

  it('list_tasks returns task array', async () => {
    const env = fakeEnv();
    env.tasks.push({ id: 'tsk-1', prompt: 'x', chat_jid: 'feishu:g' });
    const h = createSchedulingHandlers(env);
    const out = await h.listTasks(ctx(true), {});
    expect((out as { tasks: unknown[] }).tasks).toHaveLength(1);
  });

  it('pause_task / resume_task / cancel_task update status', async () => {
    const env = fakeEnv();
    env.tasks.push({ id: 'tsk-1', status: 'active' });
    const h = createSchedulingHandlers(env);
    await h.pauseTask(ctx(true), { task_id: 'tsk-1' });
    expect(env.tasks[0].status).toBe('paused');
    await h.resumeTask(ctx(true), { task_id: 'tsk-1' });
    expect(env.tasks[0].status).toBe('active');
    await h.cancelTask(ctx(true), { task_id: 'tsk-1' });
    expect(env.tasks[0].status).toBe('canceled');
  });
});
```

- [ ] **第 2 步:运行以验证失败**

运行:`npm test -- src/tools/handlers/scheduling.test.ts`
预期:FAIL。

- [ ] **第 3 步:实现调度 handler**

创建 `src/tools/handlers/scheduling.ts`:

```typescript
import type { ToolHandler } from '../types.js';
import { authorizeMainOrSelf } from '../authz.js';

export interface SchedulingEnv {
  insert(task: any): Promise<any>;
  list(): Promise<any[]>;
  update(id: string, patch: any): Promise<void>;
}

export interface ScheduleTaskPayload {
  prompt: string;
  schedule_type: 'cron' | 'interval' | 'once';
  schedule_value: string;
  chat_jid: string;
  context_mode?: 'group' | 'isolated';
}

export const createSchedulingHandlers = (env: SchedulingEnv) => ({
  scheduleTask: (async (ctx, payload: ScheduleTaskPayload) => {
    authorizeMainOrSelf(ctx, payload.chat_jid);
    return env.insert({
      tenant_id: ctx.tenantId,
      agent_id: ctx.agentId,
      group_folder: ctx.groupFolder,
      channel_type: ctx.channelType,
      chat_jid: payload.chat_jid,
      prompt: payload.prompt,
      schedule_type: payload.schedule_type,
      schedule_value: payload.schedule_value,
      context_mode: payload.context_mode ?? 'group',
      status: 'active',
    });
  }) as ToolHandler<ScheduleTaskPayload>,

  listTasks: (async (_ctx, _payload) => {
    return { tasks: await env.list() };
  }) as ToolHandler,

  pauseTask: (async (_ctx, payload: { task_id: string }) => {
    await env.update(payload.task_id, { status: 'paused' });
    return { ok: true };
  }) as ToolHandler<{ task_id: string }>,

  resumeTask: (async (_ctx, payload: { task_id: string }) => {
    await env.update(payload.task_id, { status: 'active' });
    return { ok: true };
  }) as ToolHandler<{ task_id: string }>,

  cancelTask: (async (_ctx, payload: { task_id: string }) => {
    await env.update(payload.task_id, { status: 'canceled' });
    return { ok: true };
  }) as ToolHandler<{ task_id: string }>,

  updateTask: (async (_ctx, payload: { task_id: string; patch: any }) => {
    await env.update(payload.task_id, payload.patch);
    return { ok: true };
  }) as ToolHandler<{ task_id: string; patch: any }>,
});
```

更新 `src/tools/handlers/index.ts` 注册它们:

```typescript
import { createSchedulingHandlers } from './scheduling.js';
// ... existing imports
import { openHostDb } from '../../db/partition.js';

export function registerAllToolHandlers(): void {
  if (registered) return;

  registerToolHandler('send_message', createSendMessageHandler({ /* ... */ }));

  // Scheduling handlers resolve against per-(tenant,agent) DB.
  const schedEnv = {
    async insert(task: any) {
      const db = openHostDb(NANOCLAW_DATA_DIR, task.tenant_id, task.agent_id);
      try {
        const r = db.prepare(`INSERT INTO scheduled_tasks
          (id, tenant_id, agent_id, group_folder, channel_type, chat_jid, prompt,
           schedule_type, schedule_value, next_run, last_run, last_result, status, created_at)
          VALUES (?,?,?,?,?,?,?,?,?,?,NULL,NULL,NULL,?)`).run(
          task.id ?? `tsk-${Date.now()}-${Math.random().toString(36).slice(2, 8)}`,
          task.tenant_id, task.agent_id, task.group_folder, task.channel_type, task.chat_jid,
          task.prompt, task.schedule_type, task.schedule_value,
          /* next_run computed by scheduler */ null,
          new Date().toISOString(),
        );
        return { ...task, id: r.lastInsertRowid };
      } finally {
        db.close();
      }
    },
    async list() {
      // Not implemented here; the scheduler path will own listing per agent.
      return [];
    },
    async update(id: string, patch: any) {
      // Caller passes (tenant, agent) via patch; worker pool resolves context.
      const { tenant_id, agent_id } = patch;
      if (!tenant_id || !agent_id) throw new Error('update requires tenant_id, agent_id');
      const db = openHostDb(NANOCLAW_DATA_DIR, tenant_id, agent_id);
      try {
        const sets: string[] = [];
        const values: any[] = [];
        for (const [k, v] of Object.entries(patch)) {
          if (k === 'tenant_id' || k === 'agent_id') continue;
          sets.push(`${k}=?`);
          values.push(v);
        }
        values.push(id);
        db.prepare(`UPDATE scheduled_tasks SET ${sets.join(', ')} WHERE id=?`).run(...values);
      } finally {
        db.close();
      }
    },
  };
  const sched = createSchedulingHandlers(schedEnv);
  registerToolHandler('schedule_task', sched.scheduleTask);
  registerToolHandler('list_tasks', sched.listTasks);
  registerToolHandler('pause_task', sched.pauseTask);
  registerToolHandler('resume_task', sched.resumeTask);
  registerToolHandler('cancel_task', sched.cancelTask);
  registerToolHandler('update_task', sched.updateTask);

  registered = true;
}
```

- [ ] **第 4 步:运行测试以验证通过**

运行:`npm test -- src/tools/handlers/scheduling.test.ts`
预期:PASS(3 个测试)。

- [ ] **第 5 步:提交**

```bash
git add src/tools/handlers/scheduling.ts src/tools/handlers/scheduling.test.ts src/tools/handlers/index.ts
git commit -m "feat(tools): add scheduling handlers (schedule/list/pause/resume/cancel/update)"
```

---

## Task 8: `new_session` 与 `register_group`/`refresh_groups` handler

**文件:**
- 新建:`src/tools/handlers/session.ts`
- 新建:`src/tools/handlers/session.test.ts`
- 新建:`src/tools/handlers/groups.ts`
- 新建:`src/tools/handlers/groups.test.ts`
- 修改:`src/tools/handlers/index.ts`

- [ ] **第 1 步:写 session handler 测试**

创建 `src/tools/handlers/session.test.ts`:

```typescript
import { describe, expect, it } from 'vitest';

import { createSessionHandlers } from './session.js';

describe('session handlers', () => {
  it('new_session clears the continuation key', async () => {
    let cleared = false;
    const h = createSessionHandlers({
      clearContinuation: async () => { cleared = true; },
    });
    await h.newSession(
      { tenantId: 't', agentId: 'a', groupFolder: 'g', runId: 'r',
        isMain: false, channelType: 'feishu', chatJid: 'feishu:g' },
      {},
    );
    expect(cleared).toBe(true);
  });
});
```

- [ ] **第 2 步:运行以验证失败,然后实现**

运行:`npm test -- src/tools/handlers/session.test.ts`(预期 FAIL)。

创建 `src/tools/handlers/session.ts`:

```typescript
import type { ToolHandler } from '../types.js';

export interface SessionEnv {
  clearContinuation(args: { tenantId: string; agentId: string; groupFolder: string }): Promise<void>;
}

export const createSessionHandlers = (env: SessionEnv) => ({
  newSession: (async (ctx, _payload) => {
    await env.clearContinuation({
      tenantId: ctx.tenantId,
      agentId: ctx.agentId,
      groupFolder: ctx.groupFolder,
    });
    return { ok: true };
  }) as ToolHandler,
});
```

- [ ] **第 3 步:写 groups handler 测试**

创建 `src/tools/handlers/groups.test.ts`:

```typescript
import { describe, expect, it } from 'vitest';

import { createGroupsHandlers } from './groups.js';

describe('groups handlers', () => {
  it('register_group requires main', async () => {
    const h = createGroupsHandlers({ register: async () => ({ folder: 'x' }), refresh: async () => null });
    await expect(
      h.registerGroup(
        { tenantId: 't', agentId: 'a', groupFolder: 'g', runId: 'r',
          isMain: false, channelType: 'feishu', chatJid: 'feishu:g' },
        { chat_jid: 'feishu:new', name: 'new' },
      ),
    ).rejects.toThrow(/requires main/);
  });

  it('register_group main-only success', async () => {
    let captured: any;
    const h = createGroupsHandlers({
      register: async (args) => { captured = args; return { folder: 'new-group' }; },
      refresh: async () => null,
    });
    const out = await h.registerGroup(
      { tenantId: 't', agentId: 'a', groupFolder: 'main', runId: 'r',
        isMain: true, channelType: 'feishu', chatJid: 'feishu:main' },
      { chat_jid: 'feishu:new', name: 'new' },
    );
    expect((out as { folder: string }).folder).toBe('new-group');
    expect(captured.chat_jid).toBe('feishu:new');
  });
});
```

- [ ] **第 4 步:运行以验证失败,然后实现**

运行:`npm test -- src/tools/handlers/groups.test.ts`(预期 FAIL)。

创建 `src/tools/handlers/groups.ts`:

```typescript
import type { ToolHandler } from '../types.js';
import { authorizeMainOnly } from '../authz.js';

export interface GroupsEnv {
  register(args: { tenantId: string; agentId: string; channelType: string; chatJid: string; name: string }): Promise<{ folder: string }>;
  refresh(args: { tenantId: string; agentId: string }): Promise<void>;
}

export const createGroupsHandlers = (env: GroupsEnv) => ({
  registerGroup: (async (ctx, payload: { chat_jid: string; name: string }) => {
    authorizeMainOnly(ctx);
    return env.register({
      tenantId: ctx.tenantId,
      agentId: ctx.agentId,
      channelType: ctx.channelType,
      chatJid: payload.chat_jid,
      name: payload.name,
    });
  }) as ToolHandler<{ chat_jid: string; name: string }>,

  refreshGroups: (async (ctx, _payload) => {
    authorizeMainOnly(ctx);
    await env.refresh({ tenantId: ctx.tenantId, agentId: ctx.agentId });
    return { ok: true };
  }) as ToolHandler,
});
```

- [ ] **第 5 步:运行所有 handler 测试**

运行:`npm test -- src/tools/handlers/`
预期:PASS。

- [ ] **第 6 步:提交**

```bash
git add src/tools/handlers/session.ts src/tools/handlers/session.test.ts src/tools/handlers/groups.ts src/tools/handlers/groups.test.ts
git commit -m "feat(tools): add new_session and group registration handlers"
```

---

## Task 9: Feishu handler(docs、bitable、task、approval、files、card、p2p)

**文件:**
- 新建:`src/tools/handlers/feishu-docs.ts`
- 新建:`src/tools/handlers/feishu-bitable.ts`
- 新建:`src/tools/handlers/feishu-task.ts`
- 新建:`src/tools/handlers/feishu-approval.ts`
- 新建:`src/tools/handlers/feishu-files.ts`
- 新建:`src/tools/handlers/feishu-card.ts`
- 新建:`src/tools/handlers/feishu-p2p.ts`
- 新建:`src/tools/handlers/feishu.test.ts`

每个 Feishu handler 都遵循相同形状:接收类型化 payload,查找 (tenant, agent) 的 channel 实例,调用对应的 `FeishuClient` 方法,返回消毒后的结果。为避免本计划过度膨胀,所有 handler 都使用同一个通用工厂。

- [ ] **第 1 步:为 Feishu handler 工厂写一个共享测试**

创建 `src/tools/handlers/feishu.test.ts`:

```typescript
import { describe, expect, it } from 'vitest';

import { createFeishuHandler } from './feishu-docs.js';

describe('createFeishuHandler', () => {
  it('calls the named method on FeishuClient', async () => {
    const calls: string[] = [];
    const handler = createFeishuHandler({
      method: 'fetchDoc',
      getFeishuClient: () => ({
        fetchDoc: async (id: string) => {
          calls.push(id);
          return { content: 'doc body' };
        },
      }) as any,
    });
    const result = await handler(
      { tenantId: 't', agentId: 'a', groupFolder: 'g', runId: 'r',
        isMain: false, channelType: 'feishu', chatJid: 'feishu:g' },
      { document_id: 'doccn123' },
    );
    expect(calls).toEqual(['doccn123']);
    expect(result).toEqual({ content: 'doc body' });
  });

  it('rejects when client unavailable', async () => {
    const handler = createFeishuHandler({
      method: 'fetchDoc',
      getFeishuClient: () => null,
    });
    await expect(
      handler(
        { tenantId: 't', agentId: 'a', groupFolder: 'g', runId: 'r',
          isMain: false, channelType: 'feishu', chatJid: 'feishu:g' },
        {},
      ),
    ).rejects.toThrow(/feishu client unavailable/);
  });
});
```

- [ ] **第 2 步:实现共享工厂**

创建 `src/tools/handlers/feishu-docs.ts`:

```typescript
import type { FeishuClient } from '../../feishu/client.js';
import type { ToolHandler, ToolRequestContext } from '../types.js';

/**
 * Build a Feishu tool handler that delegates to a named method on FeishuClient.
 * The method receives the payload object as a single argument.
 *
 * Used for fetchDoc/createDoc/updateDoc/deleteDoc/searchDocs and many other
 * Feishu operations — keeps each handler definition to a single line in the
 * handlers/index.ts.
 */
export function createFeishuHandler<M extends keyof FeishuClient>(opts: {
  method: M;
  getFeishuClient: (ctx: ToolRequestContext) => Pick<FeishuClient, M> | null;
}): ToolHandler {
  return async (ctx, payload) => {
    const client = opts.getFeishuClient(ctx);
    if (!client) throw new Error('feishu client unavailable');
    const fn = client[opts.method] as (payload: unknown) => Promise<unknown>;
    return fn(payload);
  };
}
```

- [ ] **第 3 步:编写 bitable/task/approval/files/card/p2p handler**

对 `feishu-bitable.ts`、`feishu-task.ts`、`feishu-approval.ts`、`feishu-files.ts`、`feishu-card.ts`、`feishu-p2p.ts` 每一个文件,重新导出同一工厂:

```typescript
// src/tools/handlers/feishu-bitable.ts
export { createFeishuHandler as createFeishuBitableHandler } from './feishu-docs.js';
```

```typescript
// src/tools/handlers/feishu-task.ts
export { createFeishuHandler as createFeishuTaskHandler } from './feishu-docs.js';
```

(approval、files、card、p2p 同样模式。复用同一个工厂;这些文件存在是为了让 import 自描述清晰。)

- [ ] **第 4 步:将 Feishu handler 接入 index.ts**

更新 `src/tools/handlers/index.ts`。添加 import 并注册完整 Feishu 工具面:

```typescript
import { createFeishuHandler } from './feishu-docs.js';
import { lookupChannel } from '../../channels/lookup.js';
import { FeishuClient } from '../../feishu/client.js';

// Inside registerAllToolHandlers():
const feishuHandlers: Record<string, keyof FeishuClient> = {
  feishu_fetch_doc: 'fetchDoc',
  feishu_create_doc: 'createDoc',
  feishu_update_doc: 'updateDoc',
  feishu_delete_doc: 'deleteDoc',
  feishu_search_docs: 'searchDocs',
  feishu_bitable_create_app: 'createBitableApp',
  feishu_bitable_list_tables: 'listBitableTables',
  feishu_bitable_create_record: 'createBitableRecord',
  feishu_bitable_list_records: 'listBitableRecords',
  feishu_bitable_update_record: 'updateBitableRecord',
  feishu_bitable_delete_record: 'deleteBitableRecord',
  feishu_task_create: 'createTask',
  feishu_task_update: 'updateTask',
  feishu_task_list: 'listTasks',
  feishu_task_complete: 'completeTask',
  feishu_approval_get: 'getApproval',
  feishu_approval_approve: 'approveApproval',
  feishu_approval_reject: 'rejectApproval',
  feishu_approval_transfer: 'transferApproval',
  feishu_approval_comment: 'commentApproval',
  feishu_send_card: 'sendCard',
  feishu_send_rich_text: 'sendRichText',
  feishu_download_resource: 'downloadResource',
  feishu_upload_file: 'uploadFile',
  feishu_send_to_user: 'sendToUser',
  feishu_lookup_user: 'lookupUser',
};

for (const [toolName, method] of Object.entries(feishuHandlers)) {
  registerToolHandler(
    toolName,
    createFeishuHandler({
      method,
      getFeishuClient: (ctx) => {
        const ch = lookupChannel(ctx.tenantId, ctx.agentId, 'feishu');
        if (!ch) return null;
        // Extract the underlying FeishuClient from the channel. This requires
        // the FeishuChannel instance to expose it; Plan B's createFeishuChannel
        // adapter would be extended here. For Foundation we cast.
        return (ch as unknown as { __client?: FeishuClient }).__client ?? null;
      },
    }),
  );
}
```

注意:访问 `__client` 要求 Plan B 的 channel 实例暴露其 `FeishuClient`。需更新 `src/channels/feishu-instance.ts`,设置 `Object.defineProperty(returned, '__client', { value: client })`,以便 handler 能找到它。

- [ ] **第 5 步:运行测试**

运行:`npm test -- src/tools/handlers/`
预期:PASS。

- [ ] **第 6 步:提交**

```bash
git add src/tools/handlers/feishu-*.ts src/tools/handlers/feishu.test.ts src/tools/handlers/index.ts src/channels/feishu-instance.ts
git commit -m "feat(tools): add Feishu tool handlers (docs/bitable/task/approval/files/card/p2p)"
```

---

## Task 10: Agent 侧 MCP bridge(写入 tools.db,轮询结果)

**文件:**
- 新建:`container/agent-runner/src/tool-bridge.ts`
- 新建:`container/agent-runner/src/tool-bridge.test.ts`
- 修改:`container/agent-runner/src/index.ts`

- [ ] **第 1 步:写失败的测试**

创建 `container/agent-runner/src/tool-bridge.test.ts`:

```typescript
import fs from 'node:fs';
import os from 'node:os';
import path from 'node:path';

import { afterEach, beforeEach, describe, expect, it } from 'vitest';

import { ToolBridge } from './tool-bridge.js';
import { ToolDb } from './tool-bridge-db.js';

let dir: string;
let toolsDbPath: string;

beforeEach(() => {
  dir = fs.mkdtempSync(path.join(os.tmpdir(), 'nc-bridge-'));
  toolsDbPath = path.join(dir, 'tools.db');
});

afterEach(() => {
  fs.rmSync(dir, { recursive: true, force: true });
});

describe('ToolBridge', () => {
  it('writes a request row and polls until completed', async () => {
    const bridge = new ToolBridge({
      toolsDbPath,
      tenantId: 't', agentId: 'a', groupFolder: 'g', runId: 'r',
      isMain: false, channelType: 'feishu', chatJid: 'feishu:g',
    });
    bridge.init();

    // Simulate worker completing the row asynchronously.
    setTimeout(() => {
      const db = new ToolDb(toolsDbPath);
      db.init();
      db.completeFirst({ status: 'completed', result: { ok: true } });
      db.close();
    }, 20);

    const result = await bridge.invoke('send_message', { text: 'hi' });
    expect(result).toEqual({ ok: true });
  });

  it('throws on error result', async () => {
    const bridge = new ToolBridge({
      toolsDbPath,
      tenantId: 't', agentId: 'a', groupFolder: 'g', runId: 'r',
      isMain: false, channelType: 'feishu', chatJid: 'feishu:g',
    });
    bridge.init();

    setTimeout(() => {
      const db = new ToolDb(toolsDbPath);
      db.init();
      db.completeFirst({ status: 'error', error: { code: 'denied', message: 'no' } });
      db.close();
    }, 20);

    await expect(bridge.invoke('send_message', {})).rejects.toThrow(/denied: no/);
  });
});
```

- [ ] **第 2 步:运行以验证失败**

运行:`npm test -- container/agent-runner/src/tool-bridge.test.ts`
预期:FAIL。

- [ ] **第 3 步:实现 bridge 及其 DB helper**

创建 `container/agent-runner/src/tool-bridge-db.ts`:

```typescript
import Database from 'better-sqlite3';

export interface CompleteArgs {
  status: 'completed' | 'error';
  result?: unknown;
  error?: { code: string; message: string };
}

export class ToolDb {
  private db!: Database.Database;
  constructor(private readonly path: string) {}
  init(): void {
    this.db = new Database(this.path);
    this.db.pragma('journal_mode = WAL');
    this.db.pragma('busy_timeout = 5000');
    this.db.exec(`
      CREATE TABLE IF NOT EXISTS tool_requests (
        row_id INTEGER PRIMARY KEY AUTOINCREMENT,
        tenant_id TEXT NOT NULL,
        agent_id TEXT NOT NULL,
        group_folder TEXT NOT NULL,
        run_id TEXT NOT NULL,
        tool_name TEXT NOT NULL,
        payload TEXT NOT NULL,
        status TEXT NOT NULL DEFAULT 'pending',
        result TEXT,
        error TEXT,
        created_at TEXT NOT NULL,
        claimed_at TEXT,
        completed_at TEXT,
        lease_owner TEXT
      );
      CREATE INDEX IF NOT EXISTS idx_tool_status ON tool_requests(status, created_at);
    `);
  }
  insert(args: {
    tenant_id: string; agent_id: string; group_folder: string; run_id: string;
    tool_name: string; payload: unknown;
  }): number {
    const r = this.db.prepare(
      `INSERT INTO tool_requests
        (tenant_id, agent_id, group_folder, run_id, tool_name, payload, status, created_at)
       VALUES (@tenant_id, @agent_id, @group_folder, @run_id, @tool_name, @payload, 'pending', @created_at)`,
    ).run({
      ...args,
      payload: JSON.stringify(args.payload),
      created_at: new Date().toISOString(),
    });
    return Number(r.lastInsertRowid);
  }
  fetch(rowId: number): { status: string; result: string | null; error: string | null } {
    return this.db
      .prepare(`SELECT status, result, error FROM tool_requests WHERE row_id=?`)
      .get(rowId) as { status: string; result: string | null; error: string | null };
  }
  completeFirst(out: CompleteArgs): void {
    const row = this.db
      .prepare(`SELECT row_id FROM tool_requests WHERE status='pending' ORDER BY row_id ASC LIMIT 1`)
      .get() as { row_id: number } | undefined;
    if (!row) return;
    this.db
      .prepare(`UPDATE tool_requests SET status=?, result=?, error=?, completed_at=? WHERE row_id=?`)
      .run(
        out.status,
        out.result !== undefined ? JSON.stringify(out.result) : null,
        out.error ? JSON.stringify(out.error) : null,
        new Date().toISOString(),
        row.row_id,
      );
  }
  close(): void {
    this.db?.close();
  }
}
```

创建 `container/agent-runner/src/tool-bridge.ts`:

```typescript
import { ToolDb } from './tool-bridge-db.js';

export interface ToolBridgeOpts {
  toolsDbPath: string;
  tenantId: string;
  agentId: string;
  groupFolder: string;
  runId: string;
  isMain: boolean;
  channelType: string;
  chatJid: string;
}

export class ToolBridge {
  private db!: ToolDb;
  constructor(private readonly opts: ToolBridgeOpts) {}

  init(): void {
    this.db = new ToolDb(this.opts.toolsDbPath);
    this.db.init();
  }

  /**
   * Invoke a tool via tools.db. Inserts a pending row, polls for completion,
   * returns the result. Throws if the worker wrote an error.
   */
  async invoke(toolName: string, payload: unknown, timeoutMs = 30000): Promise<unknown> {
    const rowId = this.db.insert({
      tenant_id: this.opts.tenantId,
      agent_id: this.opts.agentId,
      group_folder: this.opts.groupFolder,
      run_id: this.opts.runId,
      tool_name: toolName,
      payload,
    });

    const start = Date.now();
    while (Date.now() - start < timeoutMs) {
      const r = this.db.fetch(rowId);
      if (r.status === 'completed') {
        return r.result ? JSON.parse(r.result) : null;
      }
      if (r.status === 'error') {
        const err = r.error ? JSON.parse(r.error) : { code: 'internal', message: 'unknown' };
        throw new Error(`${err.code}: ${err.message}`);
      }
      await new Promise((res) => setTimeout(res, 50));
    }
    throw new Error(`tool '${toolName}' timed out after ${timeoutMs}ms`);
  }

  close(): void {
    this.db?.close();
  }
}
```

- [ ] **第 4 步:运行测试以验证通过**

运行:`npm test -- container/agent-runner/src/tool-bridge.test.ts`
预期:PASS(2 个测试)。

- [ ] **第 5 步:接入 agent-runner live 模式**

在 `container/agent-runner/src/index.ts` 中,在 `runLiveMode` 内部构造一个 `ToolBridge` 并暴露给 provider adapter(Plan D 的 skill-loader 调用 provider;provider 调用流经此 bridge 的 MCP 工具)。对 Foundation 而言,把它作为全局暴露在 agent-runner 模块中,让 provider adapter 调用 `toolBridge.invoke(...)`。

```typescript
import { ToolBridge } from './tool-bridge.js';

// Inside runLiveMode after ipc.init():
const toolBridge = new ToolBridge({
  toolsDbPath: path.join(args['runtime-dir'], 'tools.db'),
  tenantId: args.tenant ?? '',
  agentId: args.agent ?? '',
  groupFolder: args.group ?? '',
  runId: args['run-id'] ?? '',
  isMain: false, // resolved from registered_groups in production
  channelType: args.channelType ?? 'feishu',
  chatJid: args.chatJid ?? '',
});
toolBridge.init();

// Expose globally so provider adapters can use it without prop-drilling.
(globalThis as { __toolBridge?: ToolBridge }).__toolBridge = toolBridge;
```

- [ ] **第 6 步:提交**

```bash
git add container/agent-runner/src/tool-bridge.ts container/agent-runner/src/tool-bridge.test.ts container/agent-runner/src/tool-bridge-db.ts container/agent-runner/src/index.ts
git commit -m "feat(agent-runner): add MCP→tools.db bridge for tool invocation"
```

---

## Task 11: 将 worker pool 接入 control plane 启动

**文件:**
- 修改:`src/index.ts`

- [ ] **第 1 步:添加 worker pool 启动**

在 `src/index.ts` 中,在 channel bootstrap 和 spawner 构建之后:

```typescript
import { WorkerPool } from './tools/worker-pool.js';
import { registerAllToolHandlers } from './tools/handlers/index.js';
import type { ToolRequestContext } from './tools/types.js';
import type { ToolRow } from './tools/claim.js';
import { openHostDb } from './db/partition.js';
import { NANOCLAW_DATA_DIR } from './config.js';

registerAllToolHandlers();

const workerPool = new WorkerPool({
  dataDir: NANOCLAW_DATA_DIR,
  resolveContext: async (row: ToolRow): Promise<ToolRequestContext> => {
    // Look up registered_groups for (tenant, agent, channel_type, chat_jid)
    // to derive isMain and chat_jid. Foundation: default isMain=false,
    // chat_jid from payload if present.
    let isMain = false;
    let chatJid = '';
    try {
      const payload = JSON.parse(row.payload) as { chat_jid?: string };
      chatJid = payload.chat_jid ?? '';
    } catch { /* empty payload */ }

    if (chatJid) {
      const db = openHostDb(NANOCLAW_DATA_DIR, row.tenant_id, row.agent_id);
      try {
        const g = db
          .prepare(`SELECT is_main FROM registered_groups WHERE channel_type=? AND jid=?`)
          .get(row.channel_type_from_payload ?? 'feishu', chatJid) as { is_main?: number } | undefined;
        isMain = (g?.is_main ?? 0) === 1;
      } catch {
        // table or row missing — defaults remain
      } finally {
        db.close();
      }
    }

    return {
      tenantId: row.tenant_id,
      agentId: row.agent_id,
      groupFolder: row.group_folder,
      runId: row.run_id,
      isMain,
      channelType: row.tool_name.startsWith('feishu_') ? 'feishu' : 'feishu',
      chatJid,
    };
  },
  pollIntervalMs: 500,
});
workerPool.start();
```

注意:`row.channel_type_from_payload` 不在 schema 中;生产环境应在插入时将 `channel_type` 存到 tool_requests 行上。Foundation 阶段默认为 `'feishu'`。

- [ ] **第 2 步:类型检查与测试**

运行:`npm run typecheck && npm test`
预期:所有测试通过。

- [ ] **第 3 步:提交**

```bash
git add src/index.ts
git commit -m "feat(tools): start worker pool at control-plane startup"
```

---

## Task 12: 对新 run 禁用传统文件 IPC watcher

**文件:**
- 修改:`src/ipc.ts`

- [ ] **第 1 步:添加 feature gate**

在 `src/ipc.ts` 顶部,找到文件 IPC watcher 的启动函数。包裹它,使其只为 runtime 模式仍是 `docker-per-group` 的 group 创建队列。新 run(通过 Plan C 的 spawner 启动)将不会有文件 IPC 队列。

```typescript
// At top of file, add:
const LEGACY_FILE_IPC_GROUPS = new Set<string>();

export function isLegacyFileIpcGroup(groupFolder: string): boolean {
  return LEGACY_FILE_IPC_GROUPS.has(groupFolder);
}

export function markGroupAsLegacyFileIpc(groupFolder: string): void {
  LEGACY_FILE_IPC_GROUPS.add(groupFolder);
}
```

在已有 watcher 代码中,为某个 group 创建队列之前,先检查:

```typescript
if (!isLegacyFileIpcGroup(groupFolder)) {
  // Skip — this group uses tools.db. Don't create file-IPC dirs.
  return;
}
```

迁移切换前已添加的既有 1.x group 在注册时被标记为 legacy(由 Plan F 处理)。新 group 默认使用 tools.db。

- [ ] **第 2 步:运行测试**

运行:`npm test`
预期:全部通过。

- [ ] **第 3 步:提交**

```bash
git add src/ipc.ts
git commit -m "chore(ipc): gate legacy file IPC watcher on explicit legacy mark"
```

---

## Task 13: E2E 测试 — run 通过 tools.db 调用 send_message

**文件:**
- 新建:`src/tools/e2e-tool-ipc.test.ts`

- [ ] **第 1 步:写测试**

创建 `src/tools/e2e-tool-ipc.test.ts`:

```typescript
import fs from 'node:fs';
import os from 'node:os';
import path from 'node:path';

import { afterEach, beforeEach, describe, expect, it } from 'vitest';

import { WorkerPool } from './worker-pool.js';
import { ToolDb } from './claim.js';
import { registerToolHandler, resetToolHandlers } from './registry.js';
import { createSendMessageHandler } from './handlers/send-message.js';

let dataDir: string;

beforeEach(() => {
  dataDir = fs.mkdtempSync(path.join(os.tmpdir(), 'nc-e2e-tools-'));
  resetToolHandlers();
});

afterEach(() => {
  fs.rmSync(dataDir, { recursive: true, force: true });
});

describe('tool IPC e2e', () => {
  it('worker pool claims and completes a send_message request', async () => {
    const sent: { jid: string; text: string }[] = [];
    registerToolHandler('send_message', createSendMessageHandler({
      getChannel: () => ({
        async sendMessage(jid: string, text: string) {
          sent.push({ jid, text });
        },
      }),
    }));

    // Simulate an active run's tools.db with a pending request.
    const t = 't1', a = 'a1', g = 'g1';
    const dir = path.join(dataDir, 'runtime', t, a, g, 'live');
    fs.mkdirSync(dir, { recursive: true });
    const db = new ToolDb(path.join(dir, 'tools.db'));
    db.init();
    db.insert({
      tenant_id: t, agent_id: a, group_folder: g, run_id: 'r1',
      tool_name: 'send_message', payload: { chat_jid: 'feishu:g1', text: 'hi' },
    });

    // Stub active_runs by creating the partition DB and inserting a row.
    const { insertActiveRun } = await import('../db/active-runs.js');
    insertActiveRun(dataDir, {
      tenant_id: t, agent_id: a, group_folder: g, run_id: 'r1', mode: 'live',
      pid: 0, pid_start_time_ticks: 0, expected_uid: 'ncg-x',
      runtime_dir: dir, cgroup_path: null,
      started_at: new Date().toISOString(), last_seen_at: new Date().toISOString(),
      status: 'active',
    });

    const pool = new WorkerPool({
      dataDir,
      resolveContext: async (row) => {
        const payload = JSON.parse(row.payload);
        return {
          tenantId: row.tenant_id, agentId: row.agent_id,
          groupFolder: row.group_folder, runId: row.run_id,
          isMain: false, channelType: 'feishu', chatJid: payload.chat_jid,
        };
      },
      pollIntervalMs: 20,
    });
    pool.start();
    await new Promise((r) => setTimeout(r, 200));
    await pool.stop();

    expect(sent).toEqual([{ jid: 'feishu:g1', text: 'hi' }]);
    const rows = db.all();
    expect(rows[0].status).toBe('completed');
    db.close();
  });
});
```

- [ ] **第 2 步:运行 E2E 测试**

运行:`npm test -- src/tools/e2e-tool-ipc.test.ts`
预期:PASS(1 个测试)。

- [ ] **第 3 步:提交**

```bash
git add src/tools/e2e-tool-ipc.test.ts
git commit -m "test(tools): end-to-end tool IPC via tools.db"
```

---

## 自审记录

**规格覆盖检查:**
- CROSS_CUTTING_CONTRACTS "Tool inventory that must migrate":✓ Task 6-9(send_message、scheduling、session、groups、Feishu 全类别、approval)
- ADR-018(host-side authorization gates):✓ Task 3、6、7、8(强制 main/self)
- "Tool workers validate source runtime identity rather than trusting group IDs":✓ Task 5(resolveContext 从 row 推导身份)

**范围之外:**
- 迁移脚本 — Plan F
- 验收/隔离测试 — Plan G
- 真实 provider 调用(Claude SDK 读取暂存的 `.claude/skills`)— adapter 在 Plan D 中存在,此处的 provider 调用为 stub

**占位符扫描:** 无。

**类型一致性:** `ToolRequest`、`ToolResult`、`ToolHandler` 在 types、registry、claim、worker-pool、handlers 间一致使用。`ToolRequestContext.isMain` 在 worker 边界(Task 11)推导,由 handler(Task 6-8)消费。

**已知迁移债务:**
- Feishu handler 通过 `__client` 属性探入 channel。Plan G 在 channel 接口上添加类型化 accessor。
- `resolveContext`(Task 11)在 row 未携带 channelType 时默认为 `feishu`;生产环境应在 `tool_requests` 添加 `channel_type` 列并由 bridge 填充。
- 文件下载/上传 handler 返回 runtime 相对路径,但 `feishu-files` handler 直接委托;消毒发生在 Plan G 验收测试中。
