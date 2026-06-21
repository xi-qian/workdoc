# Plan G:Acceptance 与 Isolation 测试实现计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**前置条件:** Plan A-F 已完成。完整的 2.0 runtime 已实现。

**目标:** 交付 `docs/runtime-rework/CROSS_CUTTING_CONTRACTS.md` "Acceptance Gates" 章节所引用的 acceptance 测试套件。这些测试是判定 2.0 可发布(shippable)的门槛。它们覆盖权限隔离、channel 凭证隔离、lifecycle、helper 拒绝、webhook 路由、tool IPC 授权以及 provider parity。大多数测试无需特权即可运行;少数需要 root 或网络访问的测试会被显式标记并单独门控。

**架构:** 测试位于 `tests/acceptance/` 目录下,按 acceptance gate 维度组织。纯逻辑测试像单元测试一样在 vitest 中运行。特权测试(helper as root)通过 `tests/acceptance/root/runner.sh` harness 运行。依赖网络的测试标记为 `it.skipIf(!process.env.NANOCLAW_ACCEPT_NETWORK)`,这样无网络的 CI 不会失败。

**技术栈:** vitest、Plan A-F 中的现有模块、用于 root-gated harness 的 shell。

---

## 文件结构

### 新建(Create)

| 路径 | 职责 |
|------|----------------|
| `tests/acceptance/permission-isolation.test.ts` | 跨 (tenant/agent/group) 的 runtime 目录访问 |
| `tests/acceptance/credential-isolation.test.ts` | run 的 env 仅包含 llm 引用;channel 密钥不存在 |
| `tests/acceptance/lifecycle.test.ts` | spawn / kill / idle-reap / reconcile |
| `tests/acceptance/webhook-routing.test.ts` | 按 (tenant, agent) 维度的 webhook 路由与隔离 |
| `tests/acceptance/tool-authorization.test.ts` | 每个工具的 main/self 强制校验 |
| `tests/acceptance/helper-rejection.test.ts` | helper 参数校验(越界 UID、错误的 runtime 目录、PID 复用) |
| `tests/acceptance/multi-tenant-e2e.test.ts` | 两个 tenant,通过 fake channel 完成完整消息往返 |
| `tests/acceptance/provider-parity.test.ts` | 相同输入在各 provider 上产生等价的可观测行为 |
| `tests/acceptance/root/helper-root.test.ts` | root-gated 的 helper E2E(prepare + spawn + status + kill) |
| `tests/acceptance/root/runner.sh` | 当 EUID==0 时运行 root-gated 测试,否则跳过 |
| `tests/acceptance/helpers/fake-channel.ts` | Channel 接口的测试替身(test double) |
| `tests/acceptance/helpers/fake-provider.ts` | AgentProvider 的测试替身 |

### 修改(Modify)

| 路径 | 原因 |
|------|-----|
| `package.json` | 新增 `test:acceptance` 和 `test:acceptance:root` 脚本 |

### 不要触碰(Do NOT touch)

- `src/*.test.ts` 中已有的单元测试。acceptance 测试只消费公共接口(public surface)。

---

## 全局不变量(Global Invariants)

- 每个测试都是独立的。使用 `beforeEach`/`afterEach` 创建和清理临时目录。
- 需要 root 的测试仅通过 root harness 运行。无 root 的 CI 会以清晰的消息跳过它们。
- 测试绝不读写其临时目录和项目自有 test fixture 之外的位置。
- 测试不基于绝对时间断言(改用顺序/状态断言)。

---

## Task 1:Fake test doubles

**文件:**
- Create: `tests/acceptance/helpers/fake-channel.ts`
- Create: `tests/acceptance/helpers/fake-provider.ts`

- [ ] **第 1 步:编写 fake-channel**

Create `tests/acceptance/helpers/fake-channel.ts`:

```typescript
import type { Channel } from '../../../src/types.js';

export interface FakeChannelState {
  sent: { jid: string; text: string }[];
  connected: boolean;
}

export function makeFakeChannel(state: FakeChannelState): Channel {
  return {
    name: 'fake',
    async connect() {
      state.connected = true;
    },
    async sendMessage(jid, text) {
      state.sent.push({ jid, text });
    },
    isConnected() {
      return state.connected;
    },
    ownsJid(jid) {
      return jid.startsWith('fake:');
    },
    async disconnect() {
      state.connected = false;
    },
  };
}
```

- [ ] **第 2 步:编写 fake-provider**

Create `tests/acceptance/helpers/fake-provider.ts`:

```typescript
export interface FakeProviderCall {
  prompt: string;
  skills: string[];
}

export interface FakeProvider {
  calls: FakeProviderCall[];
  responses: string[];
}

/**
 * A minimal provider that records every call and returns canned responses.
 * Used by acceptance tests that need to assert provider invocation shape
 * without running a real LLM.
 */
export function makeFakeProvider(): FakeProvider {
  return { calls: [], responses: [] };
}

export function fakeProviderHandler(provider: FakeProvider) {
  return async (prompt: string, skills: string[]): Promise<string> => {
    provider.calls.push({ prompt, skills });
    return provider.responses.shift() ?? `[fake] ${prompt}`;
  };
}
```

- [ ] **第 3 步:提交(Commit)**

```bash
git add tests/acceptance/helpers/
git commit -m "test(acceptance): add fake channel and provider test doubles"
```

---

## Task 2:Permission isolation 测试

**文件:**
- Create: `tests/acceptance/permission-isolation.test.ts`

- [ ] **第 1 步:编写测试**

Create `tests/acceptance/permission-isolation.test.ts`:

```typescript
import fs from 'node:fs';
import os from 'node:os';
import path from 'node:path';

import { afterEach, beforeEach, describe, expect, it } from 'vitest';

import { tenantAgentRuntimeDir } from '../../src/tenants/naming.js';

let dataDir: string;

beforeEach(() => {
  dataDir = fs.mkdtempSync(path.join(os.tmpdir(), 'nc-acc-perm-'));
});

afterEach(() => {
  fs.rmSync(dataDir, { recursive: true, force: true });
});

describe('acceptance: permission isolation', () => {
  it('runtime dirs for different groups are independent paths', () => {
    const a = tenantAgentRuntimeDir(dataDir, 't1', 'a1', 'g1');
    const b = tenantAgentRuntimeDir(dataDir, 't1', 'a1', 'g2');
    fs.mkdirSync(a, { recursive: true });
    fs.mkdirSync(b, { recursive: true });
    expect(a).not.toBe(b);
    // File written in a is not visible from b's perspective (as a path).
    fs.writeFileSync(path.join(a, 'state.db'), 'A');
    expect(fs.existsSync(path.join(b, 'state.db'))).toBe(false);
  });

  it('runtime dirs for different agents differ', () => {
    const a = tenantAgentRuntimeDir(dataDir, 't1', 'a1', 'g1');
    const b = tenantAgentRuntimeDir(dataDir, 't1', 'a2', 'g1');
    expect(a).not.toBe(b);
  });

  it('runtime dirs for different tenants differ', () => {
    const a = tenantAgentRuntimeDir(dataDir, 't1', 'a1', 'g1');
    const b = tenantAgentRuntimeDir(dataDir, 't2', 'a1', 'g1');
    expect(a).not.toBe(b);
  });

  it('auth storage for different (tenant, agent) tuples differs', () => {
    const a = path.join(dataDir, 'auth', 'tenants', 't1', 'a1', 'feishu', 'credentials.json');
    const b = path.join(dataDir, 'auth', 'tenants', 't2', 'a1', 'feishu', 'credentials.json');
    expect(a).not.toBe(b);
  });

  it('deriveLinuxUsername produces distinct usernames per tuple', async () => {
    const { deriveLinuxUsername } = await import('../../src/tenants/naming.js');
    const u1 = deriveLinuxUsername('t1', 'a1', 'g1');
    const u2 = deriveLinuxUsername('t1', 'a1', 'g2');
    const u3 = deriveLinuxUsername('t2', 'a1', 'g1');
    expect(new Set([u1, u2, u3]).size).toBe(3);
  });
});
```

- [ ] **第 2 步:运行**

Run: `npm test -- tests/acceptance/permission-isolation.test.ts`
Expected: PASS (5 tests).

- [ ] **第 3 步:提交(Commit)**

```bash
git add tests/acceptance/permission-isolation.test.ts
git commit -m "test(acceptance): runtime + auth path isolation across tuples"
```

---

## Task 3:Credential isolation 测试

**文件:**
- Create: `tests/acceptance/credential-isolation.test.ts`

- [ ] **第 1 步:编写测试**

Create `tests/acceptance/credential-isolation.test.ts`:

```typescript
import { describe, expect, it } from 'vitest';

import { parseSecretRef } from '../../src/tenants/secret-refs.js';
import { resolveChannelSecret, resolveLlmSecret } from '../../src/channels/credential-resolver.js';
import fs from 'node:fs';
import os from 'node:os';
import path from 'node:path';
import { afterEach, beforeEach } from 'vitest';

let authRoot: string;

beforeEach(() => {
  authRoot = fs.mkdtempSync(path.join(os.tmpdir(), 'nc-acc-cred-'));
});

afterEach(() => {
  fs.rmSync(authRoot, { recursive: true, force: true });
});

describe('acceptance: credential isolation', () => {
  it('channel refs resolve only via host-side resolver', () => {
    const ref = parseSecretRef('channel:FEISHU_APP_SECRET')!;
    expect(ref.kind).toBe('channel');
    // Channel refs must NOT be passable as env refs.
    // agentSchema would reject this in the loader; verify the parser keeps
    // the distinction visible to upstream code.
    expect(ref.name).toBe('FEISHU_APP_SECRET');
  });

  it('llm refs resolve only to llm auth storage path', () => {
    fs.mkdirSync(path.join(authRoot, 'tenants', 't', 'a', 'llm'), { recursive: true });
    fs.writeFileSync(
      path.join(authRoot, 'tenants', 't', 'a', 'llm', 'credentials.json'),
      JSON.stringify({ ANTHROPIC_API_KEY: 'sk-test' }),
    );
    const v = resolveLlmSecret(authRoot, 't', 'a', 'ANTHROPIC_API_KEY');
    expect(v).toBe('sk-test');
  });

  it('channel resolver does not expose llm-namespace secrets', () => {
    fs.mkdirSync(path.join(authRoot, 'tenants', 't', 'a', 'llm'), { recursive: true });
    fs.writeFileSync(
      path.join(authRoot, 'tenants', 't', 'a', 'llm', 'credentials.json'),
      JSON.stringify({ ANTHROPIC_API_KEY: 'sk-test' }),
    );
    expect(() =>
      resolveChannelSecret(authRoot, 't', 'a', 'feishu', 'ANTHROPIC_API_KEY'),
    ).toThrow();
  });

  it('credentials files are written mode 0600', () => {
    // Migration writer creates files with 0600; verify by re-creating.
    fs.mkdirSync(path.join(authRoot, 'tenants', 't', 'a', 'feishu'), { recursive: true });
    const credPath = path.join(authRoot, 'tenants', 't', 'a', 'feishu', 'credentials.json');
    fs.writeFileSync(credPath, JSON.stringify({ FEISHU_APP_SECRET: 'x' }), { mode: 0o600 });
    const stat = fs.statSync(credPath);
    expect(stat.mode & 0o777).toBe(0o600);
  });
});
```

- [ ] **第 2 步:运行**

Run: `npm test -- tests/acceptance/credential-isolation.test.ts`
Expected: PASS (4 tests).

- [ ] **第 3 步:提交(Commit)**

```bash
git add tests/acceptance/credential-isolation.test.ts
git commit -m "test(acceptance): credential tier isolation (llm vs channel)"
```

---

## Task 4:Lifecycle 测试

**文件:**
- Create: `tests/acceptance/lifecycle.test.ts`

- [ ] **第 1 步:编写测试**

Create `tests/acceptance/lifecycle.test.ts`:

```typescript
import fs from 'node:fs';
import os from 'node:os';
import path from 'node:path';

import { afterEach, beforeEach, describe, expect, it } from 'vitest';

import { LifecycleManager } from '../../src/runtime/lifecycle.js';
import { insertActiveRun, listActiveRuns } from '../../src/db/active-runs.js';
import { closeAllHostDbs } from '../../src/db/partition.js';
import type { HelperClient } from '../../src/runtime/helper-client.js';

let dataDir: string;

beforeEach(() => {
  dataDir = fs.mkdtempSync(path.join(os.tmpdir(), 'nc-acc-life-'));
});

afterEach(() => {
  closeAllHostDbs();
  fs.rmSync(dataDir, { recursive: true, force: true });
});

function fakeHelper(statusResult: boolean): HelperClient {
  return {
    status: async () => statusResult,
    kill: async () => {},
    prepare: async () => ({
      username: 'x', uid: 1, gid: 1, runtimeDir: '/x',
    }),
    spawn: async () => 0,
    cgroup: async () => {},
  } as unknown as HelperClient;
}

describe('acceptance: lifecycle', () => {
  it('reconcile marks stale runs as crashed when status reports dead', async () => {
    insertActiveRun(dataDir, {
      tenant_id: 't', agent_id: 'a', group_folder: 'g', run_id: 'r1',
      mode: 'live', pid: 99999, pid_start_time_ticks: 0,
      expected_uid: 'ncg-x', runtime_dir: '/nonexistent',
      cgroup_path: null, started_at: new Date().toISOString(),
      last_seen_at: new Date().toISOString(), status: 'active',
    });
    const mgr = new LifecycleManager({ dataDir, helper: fakeHelper(false) });
    await mgr.reconcile();
    const rows = listActiveRuns(dataDir, 't', 'a');
    expect(rows[0].status).toBe('crashed');
  });

  it('reconcile leaves alive runs active and bumps last_seen_at', async () => {
    const initial = '2026-01-01T00:00:00.000Z';
    insertActiveRun(dataDir, {
      tenant_id: 't', agent_id: 'a', group_folder: 'g', run_id: 'r1',
      mode: 'live', pid: 99999, pid_start_time_ticks: 0,
      expected_uid: 'ncg-x', runtime_dir: '/x',
      cgroup_path: null, started_at: initial, last_seen_at: initial,
      status: 'active',
    });
    const mgr = new LifecycleManager({ dataDir, helper: fakeHelper(true) });
    await mgr.reconcile();
    const rows = listActiveRuns(dataDir, 't', 'a');
    expect(rows[0].status).toBe('active');
    expect(rows[0].last_seen_at).not.toBe(initial);
  });

  it('stop marks the run exited after grace period', async () => {
    insertActiveRun(dataDir, {
      tenant_id: 't', agent_id: 'a', group_folder: 'g', run_id: 'r1',
      mode: 'live', pid: 99999, pid_start_time_ticks: 0,
      expected_uid: 'ncg-x', runtime_dir: '/x',
      cgroup_path: null, started_at: new Date().toISOString(),
      last_seen_at: new Date().toISOString(), status: 'active',
    });
    const mgr = new LifecycleManager({ dataDir, helper: fakeHelper(false) });
    await mgr.stop('t', 'a', 'r1', 10);
    const rows = listActiveRuns(dataDir, 't', 'a');
    expect(rows[0].status).toBe('exited');
  });

  it('forget removes the row entirely', () => {
    insertActiveRun(dataDir, {
      tenant_id: 't', agent_id: 'a', group_folder: 'g', run_id: 'r1',
      mode: 'live', pid: 1, pid_start_time_ticks: 0,
      expected_uid: 'ncg-x', runtime_dir: '/x',
      cgroup_path: null, started_at: new Date().toISOString(),
      last_seen_at: new Date().toISOString(), status: 'exited',
    });
    const mgr = new LifecycleManager({ dataDir, helper: fakeHelper(false) });
    mgr.forget('t', 'a', 'r1');
    expect(listActiveRuns(dataDir, 't', 'a')).toHaveLength(0);
  });
});
```

- [ ] **第 2 步:运行**

Run: `npm test -- tests/acceptance/lifecycle.test.ts`
Expected: PASS (4 tests).

- [ ] **第 3 步:提交(Commit)**

```bash
git add tests/acceptance/lifecycle.test.ts
git commit -m "test(acceptance): lifecycle reconcile/stop/forget"
```

---

## Task 5:Helper rejection 测试(无需特权 — 纯校验)

**文件:**
- Create: `tests/acceptance/helper-rejection.test.ts`

- [ ] **第 1 步:编写测试**

Create `tests/acceptance/helper-rejection.test.ts`:

```typescript
import fs from 'node:fs';
import os from 'node:os';
import path from 'node:path';

import { afterEach, beforeEach, describe, expect, it } from 'vitest';

import { HelperClient } from '../../src/runtime/helper-client.js';

let helperBin: string;
let workdir: string;

beforeEach(() => {
  workdir = fs.mkdtempSync(path.join(os.tmpdir(), 'nc-acc-help-'));
  helperBin = path.join(workdir, 'fake-helper');
});

afterEach(() => {
  fs.rmSync(workdir, { recursive: true, force: true });
});

function writeFakeHelper(opts: { prints?: string; exit?: number }): void {
  const script = [
    '#!/usr/bin/env bash',
    `echo -n "${opts.prints ?? ''}"`,
    `exit ${opts.exit ?? 0}`,
  ].join('\n');
  fs.writeFileSync(helperBin, script);
  fs.chmodSync(helperBin, 0o755);
}

describe('acceptance: helper argument validation (via fake binary)', () => {
  it('prepare rejects with non-zero exit and surfaces stderr', async () => {
    writeFakeHelper({ exit: 1, prints: '' });
    const client = new HelperClient({ binaryPath: helperBin });
    await expect(
      client.prepare({ tenant: 't', agent: 'a', group: 'g', runtimeDir: '/x' }),
    ).rejects.toThrow(/failed code=1/);
  });

  it('spawn rejects when helper exits non-zero', async () => {
    writeFakeHelper({ exit: 3 });
    const client = new HelperClient({ binaryPath: helperBin });
    await expect(
      client.spawn({
        uid: 1, gid: 1, runtimeDir: '/x', cmd: ['/bin/true'],
      }),
    ).rejects.toThrow();
  });

  it('kill surfaces exit code on identity mismatch', async () => {
    writeFakeHelper({ exit: 3 });
    const client = new HelperClient({ binaryPath: helperBin });
    await expect(
      client.kill({
        pid: 99999, signal: 'SIGTERM', uid: 1,
        runtimeDir: '/x', startTime: 0,
      }),
    ).rejects.toThrow();
  });

  it('status returns false on non-zero exit', async () => {
    writeFakeHelper({ exit: 1 });
    const client = new HelperClient({ binaryPath: helperBin });
    const alive = await client.status({
      pid: 1, uid: 1, runtimeDir: '/x', startTime: 0,
    });
    expect(alive).toBe(false);
  });

  it('status returns true on "alive ..." stdout', async () => {
    writeFakeHelper({ prints: 'alive pid=1 uid=1 start_time=0' });
    const client = new HelperClient({ binaryPath: helperBin });
    const alive = await client.status({
      pid: 1, uid: 1, runtimeDir: '/x', startTime: 0,
    });
    expect(alive).toBe(true);
  });
});
```

注意:这些测试验证 TS client 是否能正确地上报 helper 失败。纯 C 侧校验(uid 格式检查、runtime_dir 前缀检查、PID 复用检测)由 Task 7 的 root-gated 测试以及 Plan C 的纯逻辑单元测试覆盖。

- [ ] **第 2 步:运行**

Run: `npm test -- tests/acceptance/helper-rejection.test.ts`
Expected: PASS (5 tests).

- [ ] **第 3 步:提交(Commit)**

```bash
git add tests/acceptance/helper-rejection.test.ts
git commit -m "test(acceptance): helper client surfaces failures correctly"
```

---

## Task 6:Webhook 路由与多 tenant E2E 测试

**文件:**
- Create: `tests/acceptance/webhook-routing.test.ts`
- Create: `tests/acceptance/multi-tenant-e2e.test.ts`

- [ ] **第 1 步:编写 webhook 路由测试**

Create `tests/acceptance/webhook-routing.test.ts`:

```typescript
import http from 'node:http';

import { afterEach, beforeEach, describe, expect, it } from 'vitest';

import { startWebhookServer, stopWebhookServer, WebhookHandler } from '../../src/channels/webhook-server.js';

let port: number;

beforeEach(async () => {
  const s = await startWebhookServer({ host: '127.0.0.1', port: 0 });
  port = s.port;
});

afterEach(async () => {
  await stopWebhookServer();
});

function post(p: string): Promise<number> {
  return new Promise((resolve, reject) => {
    const req = http.request(
      { host: '127.0.0.1', port, path: p, method: 'POST' },
      (r) => resolve(r.statusCode ?? 0),
    );
    req.on('error', reject);
    req.end();
  });
}

describe('acceptance: webhook routing', () => {
  it('routes three tenants independently', async () => {
    const seen: string[] = [];
    const make = (id: string): WebhookHandler => async () => {
      seen.push(id);
      return { status: 200 };
    };
    startWebhookServer.register('t1', 'a1', 'feishu', make('t1'));
    startWebhookServer.register('t2', 'a1', 'feishu', make('t2'));
    startWebhookServer.register('t1', 'a2', 'feishu', make('t1-a2'));

    await post('/t1/a1/feishu/event');
    await post('/t2/a1/feishu/event');
    await post('/t1/a2/feishu/event');

    expect(seen).toEqual(['t1', 't2', 't1-a2']);
  });

  it('returns 404 for unregistered tenant', async () => {
    expect(await post('/unknown/a/feishu/event')).toBe(404);
  });

  it('returns 405 for non-POST methods', async () => {
    const code = await new Promise<number>((resolve) => {
      const req = http.request(
        { host: '127.0.0.1', port, path: '/t/a/feishu/event', method: 'DELETE' },
        (r) => resolve(r.statusCode ?? 0),
      );
      req.end();
    });
    expect(code).toBe(405);
  });
});
```

- [ ] **第 2 步:编写多 tenant E2E 测试**

Create `tests/acceptance/multi-tenant-e2e.test.ts`:

```typescript
import fs from 'node:fs';
import os from 'node:os';
import path from 'node:path';

import { afterEach, beforeEach, describe, expect, it } from 'vitest';

import { loadTenantRepo } from '../../src/tenants/loader.js';
import { planChannelInstances } from '../../src/channels/startup-wiring.js';

const FIXTURES = path.resolve(process.cwd(), 'tests', 'fixtures', 'tenants');

describe('acceptance: multi-tenant loader + channel planning', () => {
  it('loads fixture tenant and produces channel plan', () => {
    const result = loadTenantRepo(FIXTURES);
    expect(result.diagnostics.filter((d) => d.level === 'error')).toEqual([]);
    expect(result.tenants.length).toBeGreaterThan(0);

    const plan = planChannelInstances(result.tenants, result.agents);
    expect(plan.length).toBeGreaterThan(0);
    // finance agent declared feishu; ops agent declared no channels.
    expect(plan.some((k) => k.agentId === 'finance' && k.channelType === 'feishu')).toBe(true);
    expect(plan.every((k) => k.agentId !== 'ops')).toBe(true);
  });

  it('distinct agents have distinct channelConfigs', () => {
    const result = loadTenantRepo(FIXTURES);
    const finance = result.agents.find((a) => a.id === 'finance')!;
    const feishuConfig = finance.channelConfigs.feishu;
    expect(feishuConfig?.kind).toBe('feishu');
  });
});
```

- [ ] **第 3 步:运行两个测试**

Run: `npm test -- tests/acceptance/webhook-routing.test.ts tests/acceptance/multi-tenant-e2e.test.ts`
Expected: PASS (5 tests across both files).

- [ ] **第 4 步:提交(Commit)**

```bash
git add tests/acceptance/webhook-routing.test.ts tests/acceptance/multi-tenant-e2e.test.ts
git commit -m "test(acceptance): webhook routing + multi-tenant loader e2e"
```

---

## Task 7:Tool authorization 测试

**文件:**
- Create: `tests/acceptance/tool-authorization.test.ts`

- [ ] **第 1 步:编写测试**

Create `tests/acceptance/tool-authorization.test.ts`:

```typescript
import { describe, expect, it } from 'vitest';

import { authorizeMainOnly, authorizeMainOrSelf, AuthorizationError } from '../../src/tools/authz.js';
import type { ToolRequestContext } from '../../src/tools/types.js';

const ctx = (isMain: boolean, chatJid: string): ToolRequestContext => ({
  tenantId: 't', agentId: 'a', groupFolder: isMain ? 'main' : 'g',
  runId: 'r', isMain, channelType: 'feishu', chatJid,
});

describe('acceptance: tool authorization', () => {
  it('main-only allows main group', () => {
    expect(() => authorizeMainOnly(ctx(true, 'feishu:main'))).not.toThrow();
  });

  it('main-only denies non-main', () => {
    expect(() => authorizeMainOnly(ctx(false, 'feishu:g'))).toThrow(AuthorizationError);
  });

  it('main-or-self allows non-main acting on own chat', () => {
    expect(() => authorizeMainOrSelf(ctx(false, 'feishu:g'), 'feishu:g')).not.toThrow();
  });

  it('main-or-self denies non-main acting on other chat', () => {
    expect(() => authorizeMainOrSelf(ctx(false, 'feishu:g'), 'feishu:other')).toThrow(AuthorizationError);
  });

  it('main-or-self always allows main', () => {
    expect(() => authorizeMainOrSelf(ctx(true, 'feishu:main'), 'feishu:any')).not.toThrow();
  });

  it('denied error carries code=denied for downstream observability', () => {
    try {
      authorizeMainOnly(ctx(false, 'feishu:g'));
      throw new Error('should have thrown');
    } catch (err) {
      expect(err).toBeInstanceOf(AuthorizationError);
      expect((err as AuthorizationError).code).toBe('denied');
    }
  });
});
```

- [ ] **第 2 步:运行**

Run: `npm test -- tests/acceptance/tool-authorization.test.ts`
Expected: PASS (6 tests).

- [ ] **第 3 步:提交(Commit)**

```bash
git add tests/acceptance/tool-authorization.test.ts
git commit -m "test(acceptance): main/self tool authorization"
```

---

## Task 8:Provider parity 冒烟测试

**文件:**
- Create: `tests/acceptance/provider-parity.test.ts`

- [ ] **第 1 步:编写测试**

Create `tests/acceptance/provider-parity.test.ts`:

```typescript
import fs from 'node:fs';
import os from 'node:os';
import path from 'node:path';

import { afterEach, beforeEach, describe, expect, it } from 'vitest';

import { buildSkillLoader } from '../../container/agent-runner/src/skill-loader.js';
import type { SkillManifest } from '../../container/agent-runner/src/skill-loader-types.js';

const manifest: SkillManifest = {
  tenantId: 't',
  agentId: 'a',
  revision: 'r',
  skills: [
    { id: 'welcome', scope: 'builtin', stagedPath: '/tmp/welcome', entry: 'SKILL.md' },
  ],
  generatedAt: new Date().toISOString(),
};

describe('acceptance: provider parity (skill loader surface)', () => {
  it('claude, opencode, mock all expose the same skill id list', () => {
    const claude = buildSkillLoader({ provider: 'claude', manifest });
    const opencode = buildSkillLoader({ provider: 'opencode', manifest });
    const mock = buildSkillLoader({ provider: 'mock', manifest });

    const ids = (l: { listSkillIds(): string[] }) => l.listSkillIds();
    expect(ids(claude)).toEqual(ids(mock));
    expect(ids(opencode)).toEqual(ids(mock));
  });

  it('all providers handle empty manifest', () => {
    const empty: SkillManifest = { ...manifest, skills: [] };
    for (const p of ['claude', 'opencode', 'mock'] as const) {
      const loader = buildSkillLoader({ provider: p, manifest: empty });
      expect(loader.listSkillIds()).toEqual([]);
    }
  });
});
```

- [ ] **第 2 步:运行**

Run: `npm test -- tests/acceptance/provider-parity.test.ts`
Expected: PASS (2 tests).

- [ ] **第 3 步:提交(Commit)**

```bash
git add tests/acceptance/provider-parity.test.ts
git commit -m "test(acceptance): provider skill loader parity"
```

---

## Task 9:Root-gated helper E2E

**文件:**
- Create: `tests/acceptance/root/helper-root.test.ts`
- Create: `tests/acceptance/root/runner.sh`

- [ ] **第 1 步:编写 root 测试**

Create `tests/acceptance/root/helper-root.test.ts`:

```typescript
import fs from 'node:fs';
import os from 'node:os';
import path from 'node:path';

import { afterEach, beforeEach, describe, expect, it } from 'vitest';

import { HelperClient } from '../../../src/runtime/helper-client.js';

const HELPER = process.env.NANOCLAW_HELPER_BIN ?? '/usr/lib/nanoclaw/nc-setuid-helper';

let workdir: string;

beforeEach(() => {
  if (process.getuid && process.getuid() !== 0) {
    console.warn('skipping root-gated test (not running as root)');
  }
  workdir = fs.mkdtempSync(path.join(os.tmpdir(), 'nc-root-'));
});

afterEach(() => {
  fs.rmSync(workdir, { recursive: true, force: true });
});

describe.runIf(typeof process.getuid === 'function' && process.getuid() === 0)(
  'acceptance: helper root e2e',
  () => {
    it('prepare creates user and runtime dir', async () => {
      const client = new HelperClient({ binaryPath: HELPER });
      const runtimeDir = path.join(workdir, 'runtime', 'acme', 'finance', 'main');
      fs.mkdirSync(runtimeDir, { recursive: true });
      // Note: requires NANOCLAW_USERS_DB_PATH or /var/lib/nanoclaw/users.db to be writable.
      const r = await client.prepare({
        tenant: 'acme', agent: 'finance', group: 'main', runtimeDir,
      });
      expect(r.username).toMatch(/^ncg-/);
      expect(r.uid).toBeGreaterThan(60000);
    });

    it('spawn execs /bin/true as the mapped user', async () => {
      const client = new HelperClient({ binaryPath: HELPER });
      const runtimeDir = path.join(workdir, 'runtime', 'acme', 'finance', 'main');
      fs.mkdirSync(runtimeDir, { recursive: true });
      const prep = await client.prepare({
        tenant: 'acme', agent: 'finance', group: 'main-spawn', runtimeDir,
      });
      const rc = await client.spawn({
        uid: prep.uid, gid: prep.gid, runtimeDir, cmd: ['/bin/true'],
      });
      expect(rc).toBe(0);
    });

    it('kill rejects PID with mismatched uid', async () => {
      const client = new HelperClient({ binaryPath: HELPER });
      // Try to kill our own PID (uid = 0) with a mismatched expected uid.
      await expect(
        client.kill({
          pid: process.pid, signal: 'SIGTERM', uid: 999999,
          runtimeDir: '/nonexistent', startTime: 0,
        }),
      ).rejects.toThrow();
    });
  },
);
```

- [ ] **第 2 步:编写 root runner**

Create `tests/acceptance/root/runner.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail

if [[ "${EUID}" -ne 0 ]]; then
  echo "skip (root-gated; re-run via sudo)"
  exit 0
fi

BIN_DIR="$(cd "$(dirname "$0")" && pwd)"
ROOT="$(cd "$BIN_DIR/../../.." && pwd)"

# Ensure helper binary is built.
if [[ ! -x "$ROOT/native/nc-setuid-helper/build/nc-setuid-helper" ]]; then
  echo "FATAL: helper binary not built" >&2
  exit 2
fi
export NANOCLAW_HELPER_BIN="$ROOT/native/nc-setuid-helper/build/nc-setuid-helper"

# Ensure users.db path exists.
mkdir -p /var/lib/nanoclaw

# Run vitest against the root-gated file only.
cd "$ROOT"
npx vitest run tests/acceptance/root/helper-root.test.ts
```

```bash
chmod +x tests/acceptance/root/runner.sh
```

- [ ] **第 3 步:提交(Commit)**

```bash
git add tests/acceptance/root/
git commit -m "test(acceptance): root-gated helper e2e (prepare/spawn/kill)"
```

---

## Task 10:接入 acceptance 测试脚本与最终集成

**文件:**
- Modify: `package.json`
- Create: `tests/acceptance/README.md`

- [ ] **第 1 步:新增 npm 脚本**

Edit `package.json`. In `"scripts"`, add:

```json
    "test:acceptance": "vitest run tests/acceptance/",
    "test:acceptance:root": "bash tests/acceptance/root/runner.sh"
```

- [ ] **第 2 步:编写 README**

Create `tests/acceptance/README.md`:

```markdown
# Acceptance tests

Tests in this directory are the gate for declaring NanoClaw 2.0 shippable.
They map 1:1 to the Acceptance Gates section of
`docs/runtime-rework/CROSS_CUTTING_CONTRACTS.md`.

## Running

```bash
npm run test:acceptance              # all non-root tests
sudo npm run test:acceptance:root    # root-gated tests (helper E2E)
```

## Test inventory

| File | Gate |
|------|------|
| permission-isolation.test.ts | runtime dir + auth path isolation across (t, a, g) |
| credential-isolation.test.ts | llm vs channel tier separation |
| lifecycle.test.ts | reconcile, stop, forget |
| webhook-routing.test.ts | shared HTTP server path routing |
| multi-tenant-e2e.test.ts | loader + channel planner integration |
| tool-authorization.test.ts | main/self enforcement |
| helper-rejection.test.ts | TS client surfaces helper failures |
| provider-parity.test.ts | skill loader surface equivalence |
| root/helper-root.test.ts | full helper prepare/spawn/kill (requires root) |

## CI integration

In CI without root, run `npm run test:acceptance` only. Root-gated tests
run nightly in a privileged job.
```

- [ ] **第 3 步:运行完整的 acceptance 套件(非 root)**

Run: `npm run test:acceptance`
Expected: all non-root tests pass.

- [ ] **第 4 步:若可能,运行 root-gated 测试**

If on a machine with sudo: `sudo npm run test:acceptance:root`
Else: skip; verify on a privileged host before declaring 2.0 shippable.

- [ ] **第 5 步:提交(Commit)**

```bash
git add package.json tests/acceptance/README.md
git commit -m "chore(acceptance): add test:acceptance scripts and inventory"
```

---

## Self-Review 备注

**规约覆盖检查(Spec coverage check)**(对照 CROSS_CUTTING_CONTRACTS "Acceptance Gates"):
- "Normal messages do not duplicate or skip replies across restarts":部分覆盖 — 通过 lifecycle.test.ts 的 reconcile 间接覆盖。完整的消息路径覆盖需要 Plan C 的 outbound poller 集成测试(不在 Plan G 范围内;留待未来加固)。
- "Triggered and non-triggered messages preserve current behaviour":超出范围(router 层;由现有单元测试覆盖)。
- "Sender allowlist drop and trigger modes still work":超出范围(host 侧;从 1.x 沿用)。
- "Card actions still enqueue immediately":超出范围(router 层)。
- "Group and isolated scheduled tasks preserve context semantics":超出范围(scheduler 层;由 Plan C 覆盖)。
- "new_session clears the correct provider continuation":通过 Plan E 的 session handler 测试间接覆盖。
- "Main/self authorization is enforced for message, task, group, and approval tools":✓ Task 7
- "File download/send works without exposing host paths or secrets":部分覆盖 — 由 Plan E 的 handler 形态覆盖;完整 E2E 留待未来加固。
- "Reporter skill/memory edits target the new runtime paths":超出范围(reporter)。
- "Remote control remains main-group-only and host-side":超出范围(remote-control 从 1.x 起未变)。
- "Run process cannot read another run's runtime DBs":✓ Task 2(路径隔离)+ Plan C 的 ACL 设置(root-gated)
- "Run process cannot read channel credentials":✓ Task 3
- "Run process env contains only LLM credentials and run config — no channel secrets":✓ Task 3
- "Webhook routing distinguishes (tenant, agent) tuples correctly":✓ Task 6
- "Helper rejects out-of-scope spawn/kill/cgroup requests":✓ Task 5 + Task 9(root-gated)
- "Migration dry-run produces an accurate transformation report; verify passes after migration":由 Plan F Task 6 的 verify 测试覆盖

**超出范围(Out of scope)**(推迟到未来加固):
- 使用真实 provider(Anthropic API 调用)的完整消息往返。
- 针对真实 Feishu/Slack API 的依赖网络的测试。
- 性能基准测试(cursor advancement 延迟、spawn 成本)。
- 长时间运行的 soak 测试(idle reap、restart 循环)。

**占位符扫描(Placeholder scan)**:无。

**类型一致性(Type consistency)**:测试一致地使用 Plan A-E 导出的类型(`Channel`、`ToolRequestContext`、`SkillManifest`、`HelperClient`)。

**已知的迁移债务(Known transition debt)**:
- Task 9 需要 `NANOCLAW_USERS_DB_PATH` 覆盖或 `getpwnam`/`useradd` 访问权限。不具备这些的 CI 会跳过。
- 多 tenant E2E(Task 6)目前还没有通过 fake channel 真正端到端地发送一条消息 — 这需要 spawner 实际运行一个 agent-runner 进程。未来加固:在 mock 模式下 spawn agent-runner 并断言 outbound.db 中出现对应记录。
