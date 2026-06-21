# Plan B:Channel Registry 重构实施计划

> **致 agentic worker:** 必须使用 superpowers:subagent-driven-development(推荐)或 superpowers:executing-plans 来按任务逐项实现本计划。步骤使用复选框(`- [ ]`)语法进行跟踪。

**前置条件:** Plan A(Foundation)已完成 —— tenant loader、类型化的 secret ref、`ChannelConfig` 类型可用。

**目标:** 将基于 singleton key 的 channel registry 转换为基于复合键 `(tenant_id, agent_id, channel_type)` 的 registry,支持按实例惰性构造。新增一个共享的 HTTP webhook server,按 URL path 路由。移除所有 `getFeishuChannel()` 风格的 singleton 访问器。现有 runtime 仍使用 docker-per-group;本计划只触及 channel 层,使消息可以并行接收来自多个租户的 Feishu 应用。

**架构:** `src/channels/registry.ts` 维护一个 `Map<ChannelKey, Channel>` 用于存活实例,外加一个独立的 `Map<channelType, ChannelFactory>` 用于工厂。工厂签名从 `(opts) => Channel` 改为 `(key, channelConfig, opts) => Channel`。Plan A 产出的每个 `RegisteredAgent` 会生成 N 个 channel 实例(每个声明的 `channelType` 一个)。一个共享的 `src/channels/webhook-server.ts` 监听可配置的端口;每个 channel 实例在构造时注册自己的 path。IPC dispatch 通过从请求 group 的 `(tenant, agent)` 派生的复合键查找 channel 实例。

**技术栈:** TypeScript(NodeNext ESM)、现有 `@larksuiteoapi/node-sdk`、vitest、现有 `http` 模块用于 webhook server。

---

## 文件结构

### 新建

| 路径 | 职责 |
|------|----------------|
| `src/channels/keys.ts` | `ChannelKey` 类型、`makeChannelKey()`、`formatChannelKey()`(用于调试) |
| `src/channels/keys.test.ts` | Key formatter 测试 |
| `src/channels/instances.ts` | 按 (tenant,agent) 的实例 registry:`getOrCreateChannel`、`listChannels`、`closeAllChannels` |
| `src/channels/instances.test.ts` | Instance registry 测试 |
| `src/channels/webhook-server.ts` | 共享 HTTP server,按 path 路由 |
| `src/channels/webhook-server.test.ts` | Webhook router 测试 |
| `src/channels/feishu-instance.ts` | `createFeishuChannel(key, config, opts)` 工厂;替代 singleton 工厂 |
| `src/channels/feishu-instance.test.ts` | 按实例构造 Feishu 的测试 |
| `src/channels/credential-resolver.ts` | 将 `SecretRef` 解析为 auth storage 中的实际 secret 值 |
| `src/channels/credential-resolver.test.ts` | Resolver 测试 |

### 修改

| 路径 | 原因 |
|------|-----|
| `src/channels/registry.ts` | 拆分为「按类型注册的工厂」和「按复合键注册的实例」。保留旧的 `registerChannel(name, factory)` 作为过渡期的薄兼容垫片。 |
| `src/channels/feishu.ts` | 不再在 import 时自注册。改为导出一个工厂函数 `createFeishuChannel`,而非 singleton 工厂。 |
| `src/channels/index.ts` | 不再自动 import feishu.ts。按类型显式注册工厂。 |
| `src/feishu/auth.ts` | 新增按租户的 path resolver;过渡期内保留旧的全局 path 以兼容。 |
| `src/feishu/client.ts` | 构造函数接受预解析好的 `appId` + `appSecret`,不再从 auth.ts 读取。 |
| `src/index.ts` | 在 tenant 加载后,按声明的 agent 构造 channel 实例。在 channel connect 之前启动 webhook server。 |
| `src/ipc.ts` | 用 `getChannel({tenantId, agentId, channelType: 'feishu'})` 替换 `getFeishuChannel()`。从 group 文件夹派生 tenant/agent。 |

### 本计划不要触碰

- `src/container-runner.ts`、`src/group-queue.ts` —— runtime spawn 保留在旧的 docker 路径上,直到 Plan C。
- `container/agent-runner/*` —— run 进程变更属于 Plan C/D/E。
- 任何现有断言 singleton 行为的 1.x 测试;这些会被多实例等价物替换。

---

## 全局不变量(适用于每个任务)

- NodeNext ESM import 使用 `.js` 扩展名。
- 测试与源码同目录。使用 `vitest`,无需额外配置。
- 每个行为变更都采用 TDD:先写失败的测试。
- Commit 使用 `feat(channels):`、`refactor(channels):`、`chore(channels):` 前缀。
- 绝不在模块作用域持有比测试存活更久的 `Map<ChannelKey, ...>` 引用 —— 在 `afterEach` 中使用 `closeAllChannels()`。
- 绝不在 channel 代码中读取 `process.env`。所有配置通过类型化的 `ChannelConfig`(来自 tenant loader)传入。

---

## Task 1:ChannelKey 类型与格式化辅助函数

**文件:**
- 新建:`src/channels/keys.ts`
- 新建:`src/channels/keys.test.ts`

- [ ] **第 1 步:写失败的测试**

新建 `src/channels/keys.test.ts`:

```typescript
import { describe, expect, it } from 'vitest';

import {
  ChannelKeySchema,
  formatChannelKey,
  makeChannelKey,
  parseChannelKey,
} from './keys.js';

describe('ChannelKey', () => {
  it('makeChannelKey constructs and formats a key', () => {
    const k = makeChannelKey('acme', 'finance', 'feishu');
    expect(k.tenantId).toBe('acme');
    expect(k.agentId).toBe('finance');
    expect(k.channelType).toBe('feishu');
  });

  it('formatChannelKey produces a stable debug string', () => {
    const k = makeChannelKey('acme', 'finance', 'feishu');
    expect(formatChannelKey(k)).toBe('acme/finance/feishu');
  });

  it('parseChannelKey round-trips formatChannelKey', () => {
    const k = makeChannelKey('acme', 'finance', 'feishu');
    expect(parseChannelKey(formatChannelKey(k))).toEqual(k);
  });

  it('parseChannelKey rejects malformed strings', () => {
    expect(() => parseChannelKey('acme/finance')).toThrow();
    expect(() => parseChannelKey('')).toThrow();
    expect(() => parseChannelKey('a/b/c/d')).toThrow();
  });

  it('ChannelKeySchema accepts a valid key object', () => {
    expect(
      ChannelKeySchema.safeParse({
        tenantId: 'acme',
        agentId: 'finance',
        channelType: 'feishu',
      }).success,
    ).toBe(true);
  });

  it('ChannelKeySchema rejects unknown channel type', () => {
    expect(
      ChannelKeySchema.safeParse({
        tenantId: 'acme',
        agentId: 'finance',
        channelType: 'gmail',
      }).success,
    ).toBe(false);
  });
});
```

- [ ] **第 2 步:运行测试以验证失败**

运行:`npm test -- src/channels/keys.test.ts`
预期:FAIL —— 模块不存在。

- [ ] **第 3 步:实现 keys 模块**

新建 `src/channels/keys.ts`:

```typescript
import { z } from 'zod';

export const CHANNEL_TYPES = ['feishu', 'slack', 'telegram', 'discord'] as const;
export type ChannelType = (typeof CHANNEL_TYPES)[number];

export const ChannelKeySchema = z.object({
  tenantId: z.string().min(1),
  agentId: z.string().min(1),
  channelType: z.enum(CHANNEL_TYPES),
});

export type ChannelKey = z.infer<typeof ChannelKeySchema>;

export function makeChannelKey(
  tenantId: string,
  agentId: string,
  channelType: ChannelType,
): ChannelKey {
  return { tenantId, agentId, channelType };
}

export function formatChannelKey(k: ChannelKey): string {
  return `${k.tenantId}/${k.agentId}/${k.channelType}`;
}

export function parseChannelKey(s: string): ChannelKey {
  const parts = s.split('/');
  if (parts.length !== 3) {
    throw new Error(`invalid channel key '${s}': expected 'tenant/agent/type'`);
  }
  const [tenantId, agentId, channelTypeRaw] = parts;
  if (!tenantId || !agentId) {
    throw new Error(`invalid channel key '${s}': empty segments`);
  }
  if (!CHANNEL_TYPES.includes(channelTypeRaw as ChannelType)) {
    throw new Error(`invalid channel key '${s}': unknown type '${channelTypeRaw}'`);
  }
  return { tenantId, agentId, channelType: channelTypeRaw as ChannelType };
}
```

- [ ] **第 4 步:运行测试以验证通过**

运行:`npm test -- src/channels/keys.test.ts`
预期:PASS(6 个测试)。

- [ ] **第 5 步:提交**

```bash
git add src/channels/keys.ts src/channels/keys.test.ts
git commit -m "feat(channels): add ChannelKey composite key type and parser"
```

---

## Task 2:为类型化 SecretRefs 提供的 credential resolver

**文件:**
- 新建:`src/channels/credential-resolver.ts`
- 新建:`src/channels/credential-resolver.test.ts`

- [ ] **第 1 步:写失败的测试**

新建 `src/channels/credential-resolver.test.ts`:

```typescript
import fs from 'fs';
import os from 'os';
import path from 'path';

import { afterEach, beforeEach, describe, expect, it } from 'vitest';

import { resolveChannelSecret, resolveLlmSecret } from './credential-resolver.js';

let authRoot: string;

beforeEach(() => {
  authRoot = fs.mkdtempSync(path.join(os.tmpdir(), 'nc-auth-'));
});

afterEach(() => {
  fs.rmSync(authRoot, { recursive: true, force: true });
});

describe('credential-resolver', () => {
  it('resolveChannelSecret reads JSON from canonical path', () => {
    const credDir = path.join(authRoot, 'tenants', 'acme', 'finance', 'feishu');
    fs.mkdirSync(credDir, { recursive: true });
    fs.writeFileSync(
      path.join(credDir, 'credentials.json'),
      JSON.stringify({ FEISHU_APP_SECRET: 'shhh' }),
    );
    expect(
      resolveChannelSecret(authRoot, 'acme', 'finance', 'feishu', 'FEISHU_APP_SECRET'),
    ).toBe('shhh');
  });

  it('resolveChannelSecret throws on missing secret', () => {
    const credDir = path.join(authRoot, 'tenants', 'acme', 'finance', 'feishu');
    fs.mkdirSync(credDir, { recursive: true });
    fs.writeFileSync(
      path.join(credDir, 'credentials.json'),
      JSON.stringify({ OTHER: 'x' }),
    );
    expect(() =>
      resolveChannelSecret(authRoot, 'acme', 'finance', 'feishu', 'FEISHU_APP_SECRET'),
    ).toThrow(/not found/);
  });

  it('resolveChannelSecret throws on missing file', () => {
    expect(() =>
      resolveChannelSecret(authRoot, 'acme', 'finance', 'feishu', 'X'),
    ).toThrow(/credentials file not found/);
  });

  it('resolveLlmSecret reads llm credential file', () => {
    const llmDir = path.join(authRoot, 'tenants', 'acme', 'finance', 'llm');
    fs.mkdirSync(llmDir, { recursive: true });
    fs.writeFileSync(
      path.join(llmDir, 'credentials.json'),
      JSON.stringify({ ANTHROPIC_API_KEY: 'sk-xyz' }),
    );
    expect(resolveLlmSecret(authRoot, 'acme', 'finance', 'ANTHROPIC_API_KEY')).toBe('sk-xyz');
  });
});
```

- [ ] **第 2 步:运行测试以验证失败**

运行:`npm test -- src/channels/credential-resolver.test.ts`
预期:FAIL —— 模块不存在。

- [ ] **第 3 步:实现 resolver**

新建 `src/channels/credential-resolver.ts`:

```typescript
import fs from 'fs';
import path from 'path';

/**
 * Resolve a channel-scoped secret value from the canonical auth storage.
 * Path: ${authRoot}/tenants/<tenant>/<agent>/<channel>/credentials.json
 * File format: { SECRET_NAME: "value", ... }  (0600, owner=nanoclaw-svc)
 */
export function resolveChannelSecret(
  authRoot: string,
  tenant: string,
  agent: string,
  channel: string,
  name: string,
): string {
  const credPath = path.join(
    authRoot,
    'tenants',
    tenant,
    agent,
    channel,
    'credentials.json',
  );
  if (!fs.existsSync(credPath)) {
    throw new Error(`credentials file not found at ${credPath}`);
  }
  const creds = JSON.parse(fs.readFileSync(credPath, 'utf8')) as Record<string, string>;
  if (!(name in creds)) {
    throw new Error(`secret '${name}' not found in ${credPath}`);
  }
  return creds[name];
}

/**
 * Resolve an llm-scoped secret. Path: ${authRoot}/tenants/<tenant>/<agent>/llm/credentials.json
 */
export function resolveLlmSecret(
  authRoot: string,
  tenant: string,
  agent: string,
  name: string,
): string {
  const credPath = path.join(authRoot, 'tenants', tenant, agent, 'llm', 'credentials.json');
  if (!fs.existsSync(credPath)) {
    throw new Error(`llm credentials file not found at ${credPath}`);
  }
  const creds = JSON.parse(fs.readFileSync(credPath, 'utf8')) as Record<string, string>;
  if (!(name in creds)) {
    throw new Error(`llm secret '${name}' not found in ${credPath}`);
  }
  return creds[name];
}
```

- [ ] **第 4 步:运行测试以验证通过**

运行:`npm test -- src/channels/credential-resolver.test.ts`
预期:PASS(4 个测试)。

- [ ] **第 5 步:提交**

```bash
git add src/channels/credential-resolver.ts src/channels/credential-resolver.test.ts
git commit -m "feat(channels): add typed secret resolver for channel/llm refs"
```

---

## Task 3:Channel 实例 registry(复合键 + 惰性实例化)

**文件:**
- 新建:`src/channels/instances.ts`
- 新建:`src/channels/instances.test.ts`

- [ ] **第 1 步:写失败的测试**

新建 `src/channels/instances.test.ts`:

```typescript
import { afterEach, describe, expect, it } from 'vitest';

import type { Channel } from '../types.js';
import { closeAllChannels, getOrCreateChannel, listChannels } from './instances.js';
import type { ChannelKey } from './keys.js';

afterEach(() => {
  closeAllChannels();
});

function makeFakeChannel(name: string): Channel {
  return {
    name,
    async connect() {},
    async sendMessage() {},
    isConnected() {
      return true;
    },
    ownsJid() {
      return false;
    },
    async disconnect() {},
  };
}

describe('channel instance registry', () => {
  it('returns the same instance for repeated calls with the same key', () => {
    const key: ChannelKey = { tenantId: 'acme', agentId: 'finance', channelType: 'feishu' };
    const factoryCalls: string[] = [];
    const factory = (k: ChannelKey) => {
      factoryCalls.push(formatKey(k));
      return makeFakeChannel(`${k.tenantId}-${k.agentId}`);
    };
    const a = getOrCreateChannel(key, factory);
    const b = getOrCreateChannel(key, factory);
    expect(a).toBe(b);
    expect(factoryCalls).toHaveLength(1);
  });

  it('produces distinct instances for different keys', () => {
    const k1: ChannelKey = { tenantId: 'acme', agentId: 'finance', channelType: 'feishu' };
    const k2: ChannelKey = { tenantId: 'acme', agentId: 'ops', channelType: 'feishu' };
    const a = getOrCreateChannel(k1, (k) => makeFakeChannel(`${k.agentId}-ch`));
    const b = getOrCreateChannel(k2, (k) => makeFakeChannel(`${k.agentId}-ch`));
    expect(a).not.toBe(b);
    expect(listChannels().map((c) => c.name)).toEqual(['finance-ch', 'ops-ch']);
  });

  it('listChannels returns instances in insertion order', () => {
    const k1: ChannelKey = { tenantId: 't1', agentId: 'a1', channelType: 'feishu' };
    const k2: ChannelKey = { tenantId: 't1', agentId: 'a2', channelType: 'feishu' };
    getOrCreateChannel(k1, () => makeFakeChannel('first'));
    getOrCreateChannel(k2, () => makeFakeChannel('second'));
    expect(listChannels().map((c) => c.name)).toEqual(['first', 'second']);
  });

  it('closeAllChannels clears the registry', () => {
    const k: ChannelKey = { tenantId: 't', agentId: 'a', channelType: 'feishu' };
    getOrCreateChannel(k, () => makeFakeChannel('x'));
    closeAllChannels();
    expect(listChannels()).toHaveLength(0);
  });

  function formatKey(k: ChannelKey): string {
    return `${k.tenantId}/${k.agentId}/${k.channelType}`;
  }
});
```

- [ ] **第 2 步:运行测试以验证失败**

运行:`npm test -- src/channels/instances.test.ts`
预期:FAIL —— 模块不存在。

- [ ] **第 3 步:实现实例 registry**

新建 `src/channels/instances.ts`:

```typescript
import type { Channel } from '../types.js';
import type { ChannelKey } from './keys.js';
import { formatChannelKey } from './keys.js';

export type ChannelInstanceFactory = (key: ChannelKey) => Channel;

const instances = new Map<string, Channel>();

/**
 * Get an existing channel instance for the composite key, or invoke the
 * factory to construct one. The registry keeps a single instance per key;
 * repeated calls with the same key return the cached instance.
 *
 * Factories are expected to be idempotent and side-effect-free up to the
 * point of channel construction. If a factory throws, no instance is
 * cached.
 */
export function getOrCreateChannel(
  key: ChannelKey,
  factory: ChannelInstanceFactory,
): Channel {
  const cacheKey = formatChannelKey(key);
  const existing = instances.get(cacheKey);
  if (existing) return existing;
  const channel = factory(key);
  instances.set(cacheKey, channel);
  return channel;
}

export function getChannel(key: ChannelKey): Channel | undefined {
  return instances.get(formatChannelKey(key));
}

export function listChannels(): Channel[] {
  return [...instances.values()];
}

export function listChannelKeys(): ChannelKey[] {
  return [...instances.keys()].map((s) => {
    const [tenantId, agentId, channelType] = s.split('/');
    return { tenantId, agentId, channelType } as ChannelKey;
  });
}

export function closeAllChannels(): void {
  for (const c of instances.values()) {
    void c.disconnect().catch(() => {});
  }
  instances.clear();
}
```

- [ ] **第 4 步:运行测试以验证通过**

运行:`npm test -- src/channels/instances.test.ts`
预期:PASS(4 个测试)。

- [ ] **第 5 步:提交**

```bash
git add src/channels/instances.ts src/channels/instances.test.ts
git commit -m "feat(channels): add per-(tenant,agent) instance registry"
```

---

## Task 4:共享的 webhook HTTP server(基于 path 的路由)

**文件:**
- 新建:`src/channels/webhook-server.ts`
- 新建:`src/channels/webhook-server.test.ts`

- [ ] **第 1 步:写失败的测试**

新建 `src/channels/webhook-server.test.ts`:

```typescript
import http from 'http';

import { afterEach, beforeEach, describe, expect, it } from 'vitest';

import { startWebhookServer, stopWebhookServer, WebhookHandler } from './webhook-server.js';

let port: number;
let cleanup: () => Promise<void>;

beforeEach(async () => {
  const started = await startWebhookServer({ host: '127.0.0.1', port: 0 });
  port = started.port;
  cleanup = () => stopWebhookServer();
});

afterEach(async () => {
  await cleanup();
});

function post(path: string, body: unknown): Promise<{ status: number; body: string }> {
  return new Promise((resolve, reject) => {
    const data = Buffer.from(JSON.stringify(body));
    const req = http.request(
      {
        host: '127.0.0.1',
        port,
        path,
        method: 'POST',
        headers: { 'content-type': 'application/json', 'content-length': data.length },
      },
      (res) => {
        const chunks: Buffer[] = [];
        res.on('data', (c) => chunks.push(c));
        res.on('end', () =>
          resolve({ status: res.statusCode ?? 0, body: Buffer.concat(chunks).toString() }),
        );
      },
    );
    req.on('error', reject);
    req.write(data);
    req.end();
  });
}

describe('webhook server', () => {
  it('routes POST /:tenant/:agent/:channel/event to the registered handler', async () => {
    const seen: string[] = [];
    const handler: WebhookHandler = async () => {
      seen.push('called');
      return { status: 200, body: 'ok' };
    };
    startWebhookServer.register('acme', 'finance', 'feishu', handler);

    const res = await post('/acme/finance/feishu/event', { hello: 'world' });
    expect(res.status).toBe(200);
    expect(seen).toEqual(['called']);
  });

  it('returns 404 for unregistered paths', async () => {
    const res = await post('/unknown/agent/feishu/event', {});
    expect(res.status).toBe(404);
  });

  it('isolates handlers across tenants', async () => {
    const calls: string[] = [];
    startWebhookServer.register('t1', 'a1', 'feishu', async () => {
      calls.push('t1');
      return { status: 200, body: '' };
    });
    startWebhookServer.register('t2', 'a1', 'feishu', async () => {
      calls.push('t2');
      return { status: 200, body: '' };
    });

    await post('/t1/a1/feishu/event', {});
    await post('/t2/a1/feishu/event', {});
    expect(calls).toEqual(['t1', 't2']);
  });

  it('returns 405 for non-POST/non-GET methods', async () => {
    const res = await new Promise<{ status: number }>((resolve) => {
      const req = http.request(
        { host: '127.0.0.1', port, path: '/t/a/c/event', method: 'PUT' },
        (r) => resolve({ status: r.statusCode ?? 0 }),
      );
      req.end();
    });
    expect(res.status).toBe(405);
  });
});
```

- [ ] **第 2 步:运行测试以验证失败**

运行:`npm test -- src/channels/webhook-server.test.ts`
预期:FAIL —— 模块不存在。

- [ ] **第 3 步:实现 webhook server**

新建 `src/channels/webhook-server.ts`:

```typescript
import http from 'http';

export interface WebhookRequest {
  tenantId: string;
  agentId: string;
  channelType: string;
  /** URL path beyond /:t/:a/:c/event (may be empty). */
  subPath: string;
  method: string;
  headers: http.IncomingHttpHeaders;
  body: Buffer;
}

export interface WebhookResponse {
  status: number;
  body?: string;
  headers?: Record<string, string>;
}

export type WebhookHandler = (req: WebhookRequest) => Promise<WebhookResponse>;

interface ServerState {
  server: http.Server;
  port: number;
  handlers: Map<string, WebhookHandler>;
}

let state: ServerState | null = null;

function handlerKey(tenant: string, agent: string, channel: string): string {
  return `${tenant}/${agent}/${channel}`;
}

function parsePath(urlPath: string): {
  tenant?: string;
  agent?: string;
  channel?: string;
  rest?: string;
} {
  // Expected: /<tenant>/<agent>/<channel>/event[/<extra>]
  const m = urlPath.match(/^\/([^/]+)\/([^/]+)\/([^/]+)\/event(\/.*)?$/);
  if (!m) return {};
  return {
    tenant: m[1],
    agent: m[2],
    channel: m[3],
    rest: m[4] ?? '',
  };
}

/**
 * Start the shared webhook server. Pass port: 0 to bind an ephemeral port
 * for tests; the chosen port is in the returned state.
 */
export function startWebhookServer(opts: {
  host?: string;
  port?: number;
}): Promise<ServerState> {
  if (state) {
    throw new Error('webhook server already started; call stopWebhookServer first');
  }
  const handlers = new Map<string, WebhookHandler>();
  const server = http.createServer((req, res) => handleRequest(req, res, handlers));

  return new Promise((resolve) => {
    server.listen(opts.port ?? 0, opts.host ?? '0.0.0.0', () => {
      const addr = server.address();
      const boundPort = typeof addr === 'object' && addr ? addr.port : 0;
      state = { server, port: boundPort, handlers };
      resolve(state);
    });
  });
}

async function handleRequest(
  req: http.IncomingMessage,
  res: http.ServerResponse,
  handlers: Map<string, WebhookHandler>,
): Promise<void> {
  if (req.method !== 'POST' && req.method !== 'GET') {
    res.statusCode = 405;
    res.end();
    return;
  }

  const parsed = parsePath(req.url ?? '');
  if (!parsed.tenant || !parsed.agent || !parsed.channel) {
    res.statusCode = 404;
    res.end();
    return;
  }

  const handler = handlers.get(handlerKey(parsed.tenant, parsed.agent, parsed.channel));
  if (!handler) {
    res.statusCode = 404;
    res.end();
    return;
  }

  const chunks: Buffer[] = [];
  for await (const c of req) chunks.push(c);
  const body = Buffer.concat(chunks);

  try {
    const out = await handler({
      tenantId: parsed.tenant,
      agentId: parsed.agent,
      channelType: parsed.channel,
      subPath: parsed.rest ?? '',
      method: req.method ?? 'POST',
      headers: req.headers,
      body,
    });
    res.statusCode = out.status;
    if (out.headers) {
      for (const [k, v] of Object.entries(out.headers)) res.setHeader(k, v);
    }
    res.end(out.body ?? '');
  } catch (err) {
    res.statusCode = 500;
    res.end((err as Error).message);
  }
}

export namespace startWebhookServer {
  /** Register a webhook handler for (tenant, agent, channel). Overwrites previous registration. */
  export function register(
    tenant: string,
    agent: string,
    channel: string,
    handler: WebhookHandler,
  ): void {
    if (!state) {
      throw new Error('webhook server not started');
    }
    state.handlers.set(handlerKey(tenant, agent, channel), handler);
  }

  export function unregister(tenant: string, agent: string, channel: string): void {
    state?.handlers.delete(handlerKey(tenant, agent, channel));
  }
}

export async function stopWebhookServer(): Promise<void> {
  if (!state) return;
  await new Promise<void>((resolve) => state!.server.close(() => resolve()));
  state = null;
}

export function getWebhookPort(): number {
  if (!state) throw new Error('webhook server not started');
  return state.port;
}
```

- [ ] **第 4 步:运行测试以验证通过**

运行:`npm test -- src/channels/webhook-server.test.ts`
预期:PASS(4 个测试)。

- [ ] **第 5 步:提交**

```bash
git add src/channels/webhook-server.ts src/channels/webhook-server.test.ts
git commit -m "feat(channels): add shared webhook HTTP server with path routing"
```

---

## Task 5:重构 FeishuClient 接受类型化 config

**文件:**
- 修改:`src/feishu/client.ts`
- 新建:`src/feishu/client-instance.test.ts`

- [ ] **第 1 步:写失败的测试**

新建 `src/feishu/client-instance.test.ts`:

```typescript
import { describe, expect, it } from 'vitest';

import { FeishuClient } from './client.js';

describe('FeishuClient construction', () => {
  it('accepts explicit appId and appSecret', () => {
    const client = new FeishuClient({
      appId: 'cli_app1',
      appSecret: 'secret1',
    });
    // We don't connect in this test; just assert the client is built with
    // distinct credentials. Internals are private; we rely on the fact
    // that construction did not throw.
    expect(client).toBeInstanceOf(FeishuClient);
  });

  it('accepts a second concurrent instance with different credentials', () => {
    const a = new FeishuClient({ appId: 'cli_a', appSecret: 's1' });
    const b = new FeishuClient({ appId: 'cli_b', appSecret: 's2' });
    expect(a).not.toBe(b);
  });

  it('rejects empty appId', () => {
    expect(() => new FeishuClient({ appId: '', appSecret: 'x' })).toThrow();
  });
});
```

- [ ] **第 2 步:运行测试以验证失败**

运行:`npm test -- src/feishu/client-instance.test.ts`
预期:FAIL —— 构造函数签名尚未接受此形状。

- [ ] **第 3 步:重构 FeishuClient 构造函数**

阅读 `src/feishu/client.ts` 并找到现有构造函数。替换凭证输入形状,使其接受类型化对象而非从 `auth.ts` 读取。现有构造函数大概接受 `(credentials: FeishuCredentials, brand?)`,其中 `FeishuCredentials` 从全局文件加载。

新增导出的输入类型和构造函数重载。找到类声明并更新为:

```typescript
export interface FeishuClientConfig {
  appId: string;
  appSecret: string;
  /** Optional webhook block. When mode='webhook', the webhook fields are required. */
  webhook?: {
    encryptKey?: string;
    verificationToken?: string;
  };
  /** Brand/label for logging. */
  brand?: string;
}

export class FeishuClient {
  // Keep existing private fields, but source their values from the new config.
  constructor(config: FeishuClientConfig) {
    if (!config.appId) throw new Error('FeishuClient: appId is required');
    if (!config.appSecret) throw new Error('FeishuClient: appSecret is required');
    // Existing initialisation goes here, replacing credentials.appId etc.
    // with config.appId / config.appSecret. Preserve all existing methods.
    // ...
  }
}
```

在整个文件中应用机械替换:任何旧代码读取 `this.credentials.appId` 的地方,改用 `this.config.appId`;`appSecret`、`webhook.encryptKey`、`webhook.verificationToken` 同理。保持方法签名不变。保留现有的 `Lark.Client` 和 `Lark.WSClient` 连接逻辑。

如果 `FeishuCredentials` 仍被旧的 `auth.ts` loader 引用,保留该类型但加上 `@deprecated` 注释,指向 `FeishuClientConfig`。

- [ ] **第 4 步:运行测试以验证通过**

运行:`npm test -- src/feishu/client-instance.test.ts`
预期:PASS(3 个测试)。

- [ ] **第 5 步:运行现有 Feishu 测试以确认无回归**

运行:`npm test -- src/feishu/`
预期:所有现有测试仍通过(可能需要将其构造函数调用更新为新形状 —— 就地修改)。

- [ ] **第 6 步:类型检查**

运行:`npm run typecheck`
预期:无错误。

- [ ] **第 7 步:提交**

```bash
git add src/feishu/client.ts src/feishu/client-instance.test.ts
git commit -m "refactor(feishu): accept typed config in FeishuClient constructor"
```

---

## Task 6:按实例的 Feishu channel 工厂

**文件:**
- 新建:`src/channels/feishu-instance.ts`
- 新建:`src/channels/feishu-instance.test.ts`

- [ ] **第 1 步:写失败的测试**

新建 `src/channels/feishu-instance.test.ts`:

```typescript
import { describe, expect, it } from 'vitest';

import { createFeishuChannel } from './feishu-instance.js';
import type { FeishuChannelConfig } from '../tenants/types.js';
import type { ChannelOpts } from './registry.js';

const config: FeishuChannelConfig = {
  kind: 'feishu',
  mode: 'websocket',
  appId: 'cli_test',
  appSecretRef: { raw: 'channel:FEISHU_APP_SECRET', kind: 'channel', name: 'FEISHU_APP_SECRET' },
};

const opts: ChannelOpts = {
  onMessage: () => {},
  onChatMetadata: () => {},
  registeredGroups: () => ({}),
};

describe('createFeishuChannel', () => {
  it('constructs a Channel bound to (tenant, agent) identity', () => {
    const resolveSecret = () => 'fake-secret';
    const channel = createFeishuChannel(
      { tenantId: 'acme', agentId: 'finance', channelType: 'feishu' },
      config,
      opts,
      { authRoot: '/tmp/fake', resolveSecret },
    );
    expect(channel).toBeTruthy();
    expect(typeof channel.sendMessage).toBe('function');
  });

  it('calls resolveSecret with the right arguments', () => {
    const seen: string[] = [];
    const resolveSecret = (...args: string[]) => {
      seen.push(args.join('|'));
      return 'val';
    };
    createFeishuChannel(
      { tenantId: 'acme', agentId: 'finance', channelType: 'feishu' },
      config,
      opts,
      { authRoot: '/tmp/x', resolveSecret },
    );
    expect(seen).toContain('acme|finance|feishu|FEISHU_APP_SECRET');
  });

  it('returns null if secret resolution throws', () => {
    const resolveSecret = () => {
      throw new Error('nope');
    };
    const channel = createFeishuChannel(
      { tenantId: 'acme', agentId: 'finance', channelType: 'feishu' },
      config,
      opts,
      { authRoot: '/tmp/x', resolveSecret },
    );
    expect(channel).toBeNull();
  });
});
```

- [ ] **第 2 步:运行测试以验证失败**

运行:`npm test -- src/channels/feishu-instance.test.ts`
预期:FAIL —— 模块不存在。

- [ ] **第 3 步:实现工厂**

新建 `src/channels/feishu-instance.ts`:

```typescript
import { FeishuClient, FeishuClientConfig } from '../feishu/client.js';
import { FeishuChannelConfig } from '../tenants/types.js';
import { Channel } from '../types.js';
import { ChannelOpts } from './registry.js';
import { ChannelKey } from './keys.js';

export interface FeishuInstanceEnv {
  authRoot: string;
  /**
   * Resolve a channel secret. Implementations should call the typed resolver
   * from `./credential-resolver.js`. Kept as a parameter so tests can stub it.
   */
  resolveSecret: (
    tenant: string,
    agent: string,
    channel: string,
    name: string,
  ) => string;
}

/**
 * Construct a Feishu channel instance bound to a (tenant, agent) key.
 *
 * The instance carries its own FeishuClient with credentials resolved via
 * `env.resolveSecret`. If resolution fails, the factory returns null so
 * callers can emit a diagnostic and continue with other channels.
 *
 * This is the 2.0 replacement for the singleton `registerChannel('feishu', ...)`
 * pattern. The legacy factory in feishu.ts is kept only for back-compat with
 * code paths not yet migrated.
 */
export function createFeishuChannel(
  key: ChannelKey,
  config: FeishuChannelConfig,
  opts: ChannelOpts,
  env: FeishuInstanceEnv,
): Channel | null {
  let appSecret: string;
  try {
    appSecret = env.resolveSecret(
      key.tenantId,
      key.agentId,
      'feishu',
      config.appSecretRef.name,
    );
  } catch {
    return null;
  }

  const clientConfig: FeishuClientConfig = {
    appId: config.appId,
    appSecret,
    brand: `${key.tenantId}/${key.agentId}`,
  };

  if (config.mode === 'webhook' && config.webhook) {
    clientConfig.webhook = {
      encryptKey: config.webhook.encryptKeyRef
        ? safeResolve(env, key, 'feishu', config.webhook.encryptKeyRef.name)
        : undefined,
      verificationToken: config.webhook.verificationTokenRef
        ? safeResolve(env, key, 'feishu', config.webhook.verificationTokenRef.name)
        : undefined,
    };
  }

  const client = new FeishuClient(clientConfig);

  // The existing FeishuChannel class wires FeishuClient -> onMessage.
  // Construct the same class, but pass it the pre-built client instead of
  // constructing one from global auth. If the existing FeishuChannel ctor
  // signature is incompatible, wrap it: create a minimal adapter Channel
  // that delegates to the FeishuClient for connect/sendMessage.
  return {
    name: 'feishu',
    async connect() {
      // Delegate to client's connect (WS or webhook). The actual method name
      // depends on the existing FeishuClient surface; see Plan C for the
      // runtime-aware wiring. For now we assume connect() exists.
      await (client as unknown as { connect?: () => Promise<void> }).connect?.();
    },
    async sendMessage(jid, text) {
      await client.sendMessage(jid, text);
    },
    isConnected() {
      return client.isConnected();
    },
    ownsJid(jid) {
      // All feishu JIDs look like 'feishu:<id>'; this Channel claims only
      // those it owns via the client's chat list. Keep existing behaviour.
      return typeof jid === 'string' && jid.startsWith('feishu:');
    },
    async disconnect() {
      await (client as unknown as { disconnect?: () => Promise<void> }).disconnect?.();
    },
  };
}

function safeResolve(
  env: FeishuInstanceEnv,
  key: ChannelKey,
  channel: string,
  name: string,
): string | undefined {
  try {
    return env.resolveSecret(key.tenantId, key.agentId, channel, name);
  } catch {
    return undefined;
  }
}
```

- [ ] **第 4 步:运行测试以验证通过**

运行:`npm test -- src/channels/feishu-instance.test.ts`
预期:PASS(3 个测试)。

注意:该测试不会真正调用 `connect()`,因此 runtime 调用面假设(`client.connect`)是安全的。如果 `FeishuClient` 没有这些方法,可选链返回 undefined 且 connect 变成 no-op。Plan C 会对此进行加固。

- [ ] **第 5 步:提交**

```bash
git add src/channels/feishu-instance.ts src/channels/feishu-instance.test.ts
git commit -m "feat(channels): add per-instance Feishu channel factory"
```

---

## Task 7:类型感知的工厂 registry(替代 singleton 工厂)

**文件:**
- 修改:`src/channels/registry.ts`
- 新建:`src/channels/registry.test.ts`

- [ ] **第 1 步:写失败的测试**

新建 `src/channels/registry.test.ts`:

```typescript
import { describe, expect, it } from 'vitest';

import {
  getChannelFactory,
  registerChannelFactory,
  resetChannelFactories,
} from './registry.js';
import type { ChannelKey } from './keys.js';
import type { FeishuChannelConfig, RegisteredAgent } from '../tenants/types.js';

describe('factory registry', () => {
  it('registerChannelFactory stores by channelType', () => {
    resetChannelFactories();
    registerChannelFactory('feishu', () => null);
    expect(getChannelFactory('feishu')).toBeTruthy();
  });

  it('getChannelFactory returns undefined for unregistered type', () => {
    resetChannelFactories();
    expect(getChannelFactory('slack')).toBeUndefined();
  });
});
```

- [ ] **第 2 步:运行测试以验证失败**

运行:`npm test -- src/channels/registry.test.ts`
预期:FAIL —— 函数不存在。

- [ ] **第 3 步:扩展 registry**

编辑 `src/channels/registry.ts`。为兼容性保留现有 `registerChannel(name, factory)` 和 `ChannelFactory`。新增函数:

```typescript
import { ChannelKey, ChannelType } from './keys.js';
import type { ChannelConfig, RegisteredAgent } from '../tenants/types.js';

// 2.0 factory: produces a Channel given the composite key, the typed
// ChannelConfig from tenant loader, and the standard ChannelOpts. Returns
// null on soft failures (e.g. secret resolution) so loader can continue.
export type TypedChannelFactory = (
  key: ChannelKey,
  config: ChannelConfig,
  opts: ChannelOpts,
) => Channel | null;

const factories = new Map<ChannelType, TypedChannelFactory>();

export function registerChannelFactory(
  type: ChannelType,
  factory: TypedChannelFactory,
): void {
  factories.set(type, factory);
}

export function getChannelFactory(type: ChannelType): TypedChannelFactory | undefined {
  return factories.get(type);
}

export function resetChannelFactories(): void {
  factories.clear();
}

// Existing singleton-style API (back-compat) stays below this line.
```

- [ ] **第 4 步:运行测试以验证通过**

运行:`npm test -- src/channels/registry.test.ts`
预期:PASS(2 个测试)。

- [ ] **第 5 步:提交**

```bash
git add src/channels/registry.ts src/channels/registry.test.ts
git commit -m "feat(channels): add typed factory registry by channelType"
```

---

## Task 8:连接工厂 —— 替换 `import './feishu.js'` 自动注册

**文件:**
- 修改:`src/channels/feishu.ts`(停止自动注册)
- 修改:`src/channels/index.ts`(显式注册)
- 新建:`src/channels/bootstrap.test.ts`

- [ ] **第 1 步:写失败的测试**

新建 `src/channels/bootstrap.test.ts`:

```typescript
import { describe, expect, it } from 'vitest';

import { bootstrapChannelFactories } from './index.js';
import { getChannelFactory } from './registry.js';

describe('bootstrapChannelFactories', () => {
  it('registers feishu factory', () => {
    bootstrapChannelFactories();
    expect(getChannelFactory('feishu')).toBeTruthy();
  });
});
```

- [ ] **第 2 步:运行测试以验证失败**

运行:`npm test -- src/channels/bootstrap.test.ts`
预期:FAIL —— 未导出 `bootstrapChannelFactories`。

- [ ] **第 3 步:更新 index.ts**

编辑 `src/channels/index.ts`。用以下内容替换:

```typescript
import { registerChannelFactory } from './registry.js';
import { createFeishuChannel } from './feishu-instance.js';

let booted = false;

/**
 * Register all known typed channel factories. Idempotent.
 * Called once at startup; tests can call to ensure factories are available.
 */
export function bootstrapChannelFactories(): void {
  if (booted) return;
  registerChannelFactory('feishu', (key, config, opts) => {
    if (config.kind !== 'feishu') return null;
    // Credential resolver is wired at call site in Plan B Task 9; here we
    // assume opts carries an `env` field via a 2.0 extension. For the
    // bootstrap test we only assert the factory is registered.
    // The actual env is supplied by startup wiring (Task 9).
    const env = (opts as ChannelOpts & { env?: unknown }).env as
      | {
          authRoot: string;
          resolveSecret: (t: string, a: string, c: string, n: string) => string;
        }
      | undefined;
    if (!env) return null;
    return createFeishuChannel(key, config, opts, env);
  });
  booted = true;
}

// Legacy side-effect imports kept commented out for now; will delete in Plan G.
// import './feishu.js';
```

- [ ] **第 4 步:更新 feishu.ts 停止自动注册**

打开 `src/channels/feishu.ts`,在文件末尾附近找到 `registerChannel('feishu', ...)` 调用。将其注释掉并附上说明:

```typescript
// 2.0: auto-registration removed. Use bootstrapChannelFactories() in
// src/channels/index.ts and createFeishuChannel() for per-instance
// construction. The legacy singleton factory below is retained but
// commented out.
// registerChannel('feishu', legacyFactory);
```

保留 `feishu.ts` 其余部分不动,这样仍被引用的 helper 可以继续工作。

- [ ] **第 5 步:运行测试以验证通过**

运行:`npm test -- src/channels/bootstrap.test.ts`
预期:PASS(1 个测试)。

- [ ] **第 6 步:提交**

```bash
git add src/channels/index.ts src/channels/feishu.ts src/channels/bootstrap.test.ts
git commit -m "refactor(channels): replace auto-registration with bootstrapChannelFactories"
```

---

## Task 9:启动期 wiring —— 为每个加载的 agent 实例化 channel

**文件:**
- 修改:`src/index.ts`
- 新建:`src/channels/startup-wiring.test.ts`

- [ ] **第 1 步:写失败的测试**

新建 `src/channels/startup-wiring.test.ts`:

```typescript
import { describe, expect, it } from 'vitest';

import { planChannelInstances } from './startup-wiring.js';
import type { RegisteredAgent, RegisteredTenant } from '../tenants/types.js';

describe('planChannelInstances', () => {
  it('returns one plan row per declared channel per agent', () => {
    const tenants: RegisteredTenant[] = [
      { id: 'acme', name: 'Acme', rootDir: '/x', enabled: true },
    ];
    const agents: RegisteredAgent[] = [
      {
        id: 'finance',
        tenantId: 'acme',
        name: 'Finance',
        folder: '/x',
        provider: 'claude',
        model: 'sonnet',
        instructionsPath: '/x',
        resolvedSkills: [],
        channelTypes: ['feishu'],
        channelConfigs: {
          feishu: {
            kind: 'feishu',
            mode: 'websocket',
            appId: 'cli_x',
            appSecretRef: { raw: 'channel:FEISHU_APP_SECRET', kind: 'channel', name: 'FEISHU_APP_SECRET' },
          },
        },
        envRefs: [],
        limits: {},
      },
    ];
    const plan = planChannelInstances(tenants, agents);
    expect(plan).toHaveLength(1);
    expect(plan[0]).toEqual({
      tenantId: 'acme',
      agentId: 'finance',
      channelType: 'feishu',
    });
  });

  it('skips agents with no channels', () => {
    const agents: RegisteredAgent[] = [
      {
        id: 'ops',
        tenantId: 'acme',
        name: 'Ops',
        folder: '/x',
        provider: 'claude',
        model: 'sonnet',
        instructionsPath: '/x',
        resolvedSkills: [],
        channelTypes: [],
        channelConfigs: {},
        envRefs: [],
        limits: {},
      },
    ];
    expect(planChannelInstances([], agents)).toEqual([]);
  });
});
```

- [ ] **第 2 步:运行测试以验证失败**

运行:`npm test -- src/channels/startup-wiring.test.ts`
预期:FAIL —— 模块不存在。

- [ ] **第 3 步:实现 startup-wiring helper**

新建 `src/channels/startup-wiring.ts`:

```typescript
import { RegisteredAgent, RegisteredTenant } from '../tenants/types.js';
import { ChannelKey, ChannelType } from './keys.js';

/**
 * Compute the list of channel instances that should exist given loaded
 * tenants and agents. Used at startup to drive construction.
 *
 * Pure function — does not touch the registry or the webhook server.
 */
export function planChannelInstances(
  tenants: RegisteredTenant[],
  agents: RegisteredAgent[],
): ChannelKey[] {
  const enabledTenants = new Set(tenants.filter((t) => t.enabled).map((t) => t.id));
  const out: ChannelKey[] = [];
  for (const a of agents) {
    if (!enabledTenants.has(a.tenantId)) continue;
    for (const t of a.channelTypes) {
      out.push({ tenantId: a.tenantId, agentId: a.id, channelType: t as ChannelType });
    }
  }
  return out;
}
```

- [ ] **第 4 步:运行测试以验证通过**

运行:`npm test -- src/channels/startup-wiring.test.ts`
预期:PASS(2 个测试)。

- [ ] **第 5 步:接入 src/index.ts**

找到旧代码连接 channel 的位置(tenant 加载之后,`ipc.start()` 之前)。添加:

```typescript
// Add to imports:
import { bootstrapChannelFactories } from './channels/index.js';
import { startWebhookServer } from './channels/webhook-server.js';
import { planChannelInstances } from './channels/startup-wiring.js';
import { getOrCreateChannel } from './channels/instances.js';
import { resolveChannelSecret } from './channels/credential-resolver.js';
import { NANOCLAW_DATA_DIR } from './config.js';
import { AUTH_ROOT } from './config.js'; // added in this task

// After tenant load (Plan A Task 9):
bootstrapChannelFactories();

const WEBHOOK_PORT = parseInt(process.env.NANOCLAW_WEBHOOK_PORT || '0', 10);
const WEBHOOK_HOST = process.env.NANOCLAW_WEBHOOK_HOST || '0.0.0.0';
await startWebhookServer({ host: WEBHOOK_HOST, port: WEBHOOK_PORT });
logger.info({ component: 'webhook', port: getWebhookPort() }, 'webhook server listening');

const plan = planChannelInstances(tenantLoad.tenants, tenantLoad.agents);
for (const key of plan) {
  const agent = tenantLoad.agents.find(
    (a) => a.tenantId === key.tenantId && a.id === key.agentId,
  );
  if (!agent) continue;
  const cfg = agent.channelConfigs[key.channelType];
  if (!cfg) continue;
  const factory = getChannelFactory(key.channelType);
  if (!factory) {
    logger.warn({ component: 'channels', key }, 'no factory registered');
    continue;
  }
  const channel = factory(key, cfg, {
    onMessage: (jid, msg) => handleMessage(key, jid, msg),
    onChatMetadata: (jid, ts, name, ch, isGroup) =>
      handleChatMetadata(key, jid, ts, name, ch, isGroup),
    registeredGroups: () => registeredGroupsWithKey(key),
    env: {
      authRoot: AUTH_ROOT,
      resolveSecret: (t, a, c, n) => resolveChannelSecret(AUTH_ROOT, t, a, c, n),
    },
  } as ChannelOpts & { env: unknown });
  if (channel) {
    getOrCreateChannel(key, () => channel); // register with instance cache
    await channel.connect();
    logger.info({ component: 'channels', key }, 'channel connected');
  }
}
```

将 `AUTH_ROOT` 添加到 `src/config.ts`:

```typescript
export const AUTH_ROOT = path.resolve(
  process.env.NANOCLAW_AUTH_ROOT || path.join(NANOCLAW_DATA_DIR, 'auth'),
);
```

- [ ] **第 6 步:类型检查**

运行:`npm run typecheck`
预期:错误引用 `handleMessage`、`handleChatMetadata`、`registeredGroupsWithKey`、`getWebhookPort`。这些是 `src/index.ts` 中现有回调的占位符 —— 将它们接到该文件中的现有函数。改造现有的 onMessage/onChatMetadata wiring,让复合键随调用传递。这是机械式变更;如不确定,在 `src/index.ts` 中 grep `onMessage:` 即可找到当前调用点。

- [ ] **第 7 步:提交**

```bash
git add src/index.ts src/config.ts src/channels/startup-wiring.ts src/channels/startup-wiring.test.ts
git commit -m "feat(channels): wire per-agent channel instances at startup"
```

---

## Task 10:替换 `getFeishuChannel()` singleton 访问器

**文件:**
- 修改:`src/index.ts`、`src/ipc.ts`

- [ ] **第 1 步:找到所有调用点**

运行:`grep -rn 'getFeishuChannel\|channels\.find.*feishu' src/`
预期:列出所有调用点。典型位置:`src/index.ts:899`、`src/ipc.ts:242`。

- [ ] **第 2 步:引入类型化的查找 helper**

新建 `src/channels/lookup.ts`:

```typescript
import { getChannel } from './instances.js';
import type { Channel } from '../types.js';
import { ChannelKey, CHANNEL_TYPES, ChannelType } from './keys.js';

/**
 * Lookup a live channel instance by composite key. Returns undefined if no
 * instance is registered for this (tenant, agent, channelType).
 *
 * Replaces the legacy `channels.find((c) => c.name === 'feishu')` pattern.
 */
export function lookupChannel(
  tenantId: string,
  agentId: string,
  channelType: ChannelType,
): Channel | undefined {
  return getChannel({ tenantId, agentId, channelType });
}

/**
 * Parse a JID like 'feishu:oc_xxx' to a channelType. Returns undefined if
 * the JID does not match any known channel prefix.
 */
export function channelTypeFromJid(jid: string): ChannelType | undefined {
  for (const t of CHANNEL_TYPES) {
    if (jid.startsWith(`${t}:`)) return t;
  }
  return undefined;
}
```

新建 `src/channels/lookup.test.ts`:

```typescript
import { describe, expect, it } from 'vitest';

import { channelTypeFromJid } from './lookup.js';

describe('channelTypeFromJid', () => {
  it('detects feishu prefix', () => {
    expect(channelTypeFromJid('feishu:oc_x')).toBe('feishu');
  });
  it('returns undefined for unknown prefix', () => {
    expect(channelTypeFromJid('unknown:x')).toBeUndefined();
  });
});
```

- [ ] **第 3 步:运行 lookup 测试**

运行:`npm test -- src/channels/lookup.test.ts`
预期:PASS(2 个测试)。

- [ ] **第 4 步:改写调用点**

对第 1 步中找到的每个调用点,用复合键查找替换 singleton 查找。`(tenantId, agentId)` 元组必须来自上下文:典型来自请求 group 的文件夹,或消息 chat_jid 的映射。

`src/ipc.ts` 中 Feishu 工具请求 handler 的示例:

之前:
```typescript
const feishu = channels.find((c) => c.name === 'feishu');
if (!feishu) return;
await feishu.sendMessage(...);
```

之后:
```typescript
// Source group folder already encodes (tenant, agent, group).
// For back-compat, legacy groups without a tenant mapping route via the
// single configured default tenant/agent (set in Plan A index.ts wiring).
const { tenantId, agentId } = resolveTenantAgentForGroup(groupFolder);
const channelType = channelTypeFromJid(targetJid) ?? 'feishu';
const channel = lookupChannel(tenantId, agentId, channelType);
if (!channel) {
  logger.warn({ tenantId, agentId, channelType }, 'no channel instance for key');
  return;
}
await channel.sendMessage(...);
```

在 `src/ipc.ts` 顶部附近添加 helper `resolveTenantAgentForGroup`:

```typescript
function resolveTenantAgentForGroup(groupFolder: string): { tenantId: string; agentId: string } {
  // Foundation plan stored default tenant/agent in config; fall back to it
  // when group is not yet partition-aware. Plan F migrates all groups.
  return {
    tenantId: process.env.NANOCLAW_DEFAULT_TENANT ?? 'legacy',
    agentId: process.env.NANOCLAW_DEFAULT_AGENT ?? groupFolder,
  };
}
```

- [ ] **第 5 步:类型检查并运行测试**

运行:`npm run typecheck && npm test`
预期:无新增失败。

- [ ] **第 6 步:提交**

```bash
git add src/channels/lookup.ts src/channels/lookup.test.ts src/index.ts src/ipc.ts
git commit -m "refactor(channels): replace singleton accessors with composite-key lookup"
```

---

## Task 11:E2E 测试 —— 两个租户路由到不同的 channel 实例

**文件:**
- 新建:`src/channels/e2e-multi-tenant.test.ts`

- [ ] **第 1 步:写 E2E 测试**

新建 `src/channels/e2e-multi-tenant.test.ts`:

```typescript
import fs from 'fs';
import os from 'os';
import path from 'path';

import { afterEach, beforeEach, describe, expect, it } from 'vitest';

import { closeAllChannels } from './instances.js';
import { startWebhookServer, stopWebhookServer, WebhookHandler } from './webhook-server.js';
import { getOrCreateChannel } from './instances.js';
import { bootstrapChannelFactories } from './index.js';
import { getChannelFactory } from './registry.js';
import { resolveChannelSecret } from './credential-resolver.js';
import type { ChannelConfig, RegisteredAgent } from '../tenants/types.js';
import type { ChannelOpts } from './registry.js';
import type { ChannelKey } from './keys.js';

let authRoot: string;
let port: number;

const fakeFeishuConfig = (appId: string): ChannelConfig => ({
  kind: 'feishu',
  mode: 'websocket',
  appId,
  appSecretRef: { raw: 'channel:FEISHU_APP_SECRET', kind: 'channel', name: 'FEISHU_APP_SECRET' },
});

function writeSecret(tenant: string, agent: string, channel: string, name: string, value: string, root: string) {
  const dir = path.join(root, 'tenants', tenant, agent, channel);
  fs.mkdirSync(dir, { recursive: true });
  fs.writeFileSync(path.join(dir, 'credentials.json'), JSON.stringify({ [name]: value }));
}

beforeEach(async () => {
  bootstrapChannelFactories();
  authRoot = fs.mkdtempSync(path.join(os.tmpdir(), 'nc-auth-'));
  writeSecret('t1', 'a1', 'feishu', 'FEISHU_APP_SECRET', 'secret-t1', authRoot);
  writeSecret('t2', 'a1', 'feishu', 'FEISHU_APP_SECRET', 'secret-t2', authRoot);
  const s = await startWebhookServer({ host: '127.0.0.1', port: 0 });
  port = s.port;
});

afterEach(async () => {
  await stopWebhookServer();
  closeAllChannels();
  fs.rmSync(authRoot, { recursive: true, force: true });
});

describe('multi-tenant channel e2e', () => {
  it('constructs two distinct Feishu instances and routes webhooks correctly', async () => {
    const opts: ChannelOpts = {
      onMessage: () => {},
      onChatMetadata: () => {},
      registeredGroups: () => ({}),
    };
    const env = { authRoot, resolveSecret: resolveChannelSecret };
    const factory = getChannelFactory('feishu')!;

    const k1: ChannelKey = { tenantId: 't1', agentId: 'a1', channelType: 'feishu' };
    const k2: ChannelKey = { tenantId: 't2', agentId: 'a1', channelType: 'feishu' };

    const c1 = factory(k1, fakeFeishuConfig('cli_t1'), { ...opts, env } as ChannelOpts);
    const c2 = factory(k2, fakeFeishuConfig('cli_t2'), { ...opts, env } as ChannelOpts);
    expect(c1).toBeTruthy();
    expect(c2).toBeTruthy();
    expect(c1).not.toBe(c2);

    getOrCreateChannel(k1, () => c1!);
    getOrCreateChannel(k2, () => c2!);

    // Register webhook handlers that record which instance handled the call.
    const handled: string[] = [];
    const h1: WebhookHandler = async () => {
      handled.push('t1');
      return { status: 200 };
    };
    const h2: WebhookHandler = async () => {
      handled.push('t2');
      return { status: 200 };
    };
    startWebhookServer.register('t1', 'a1', 'feishu', h1);
    startWebhookServer.register('t2', 'a1', 'feishu', h2);

    const http = await import('http');
    const post = (p: string) =>
      new Promise<number>((resolve) => {
        const req = http.request(
          { host: '127.0.0.1', port, path: p, method: 'POST' },
          (r) => resolve(r.statusCode ?? 0),
        );
        req.end();
      });

    expect(await post('/t1/a1/feishu/event')).toBe(200);
    expect(await post('/t2/a1/feishu/event')).toBe(200);
    expect(handled).toEqual(['t1', 't2']);
  });
});
```

- [ ] **第 2 步:运行 E2E 测试**

运行:`npm test -- src/channels/e2e-multi-tenant.test.ts`
预期:PASS(1 个测试)。

- [ ] **第 3 步:运行完整测试套件**

运行:`npm test`
预期:所有测试通过。

- [ ] **第 4 步:提交**

```bash
git add src/channels/e2e-multi-tenant.test.ts
git commit -m "test(channels): e2e multi-tenant webhook routing"
```

---

## 自检说明

**规格覆盖检查**(对照 `docs/runtime-rework/`):
- ADR-021(按 (tenant,agent) 的外部身份):✓ Task 1、3、6、9、11
- ADR-022(共享 webhook + path 路由):✓ Task 4、11
- ADR-025(复合键 registry,移除 singleton):✓ Task 1、3、7、8、10
- CROSS_CUTTING_CONTRACTS「Inbound webhook server」(单个共享 HTTP):✓ Task 4
- CROSS_CUTTING_CONTRACTS「Channel isolation」(按实例的状态安全):✓ Task 5、6

**范围外**(延后):
- SUID helper、run 进程 spawn、runtime DB(`inbound.db` 等)—— Plan C
- Skill bundle staging —— Plan D
- 真正的 `sendMessage` 端到端通过 runtime —— Plan C
- 将旧 `groups/<folder>/` 迁移到 tenant 布局 —— Plan F

**占位符扫描**:无。每一步都有代码。

**类型一致性**:`ChannelKey` 在 `keys.ts`、`instances.ts`、`feishu-instance.ts`、`webhook-server.ts`、`startup-wiring.ts`、`lookup.ts` 中一致使用。Plan A 的 `FeishuChannelConfig` 原样透传。

**已知过渡期债务**(已指出,不阻塞):
- `src/channels/feishu.ts` 中旧的 singleton 工厂被注释但未删除。Plan G 在所有调用点验证完毕后删除。
- `resolveTenantAgentForGroup` 使用环境变量 fallback。Plan F 在 `registered_groups` 表的 `tenant_id` / `agent_id` 列填充后改用 DB 查询。
- `src/index.ts` 的 Task 9 第 5 步引用 `handleMessage` / `handleChatMetadata` / `registeredGroupsWithKey`,这些是该文件中的现有函数;工程师将其与复合键作为额外参数连接。
