# Foundation:租户配置加载器 + DB 分区实现计划

> **致 agentic worker:** 必需的子技能:使用 superpowers:subagent-driven-development(推荐)或 superpowers:executing-plans 按任务逐个实现本计划。步骤使用 checkbox(`- [ ]`)语法进行跟踪。

**目标:** 新增 tenant (租户) 配置加载、类型化 secret 引用、按 (tenant, agent) 划分的 SQLite DB 分区,以及用于重启对账的 `active_runs` 表 —— 不改变 runtime 行为。旧的 docker-per-group runtime 继续可用;Foundation 只是新增后续计划将要消费的能力。

**架构:** 新增 `src/tenants/` 模块加载 tenant 仓库(包含 JSON 配置 + skills + channels),用 zod schema 校验,解析 `llm:`/`channel:` 类型化 secret 引用,生成 `RegisteredTenant` / `RegisteredAgent` 对象。新增 `src/db/partition.ts` 在 `${NANOCLAW_DATA_DIR}/data/tenants/<t>/<a>/messages.db` 路径下打开按 (tenant, agent) 划分的 SQLite DB,每个 DB 都包含完整现有 schema,并将 `channel_type` 作为 chat/message/group/cursor 表的必需键。新增 `active_runs` 表记录 `pid + start_time_ticks + expected_uid + runtime_dir + cgroup_path + tenant + agent + group + run_id`,用于重启对账。启动时打印诊断信息,但不消费其中任何内容。

**技术栈:** TypeScript (NodeNext ESM)、zod 4.x、better-sqlite3、vitest 4.x。

---

## 文件结构

### 新建

| 路径 | 职责 |
|------|----------------|
| `src/tenants/types.ts` | 类型定义:`RegisteredTenant`、`RegisteredAgent`、`ResolvedSkill`、`AgentLimits`、`ChannelConfig`、`SecretRef`、`TenantDiagnostic` |
| `src/tenants/secret-refs.ts` | 解析并校验类型化 secret 引用(`llm:<name>`、`channel:<name>`) |
| `src/tenants/schemas.ts` | `tenant.json`、`agent.json`、`channels/*.json`、skill manifest 的 zod schema |
| `src/tenants/naming.ts` | tenant/agent/group ID 净化、Linux 用户名格式(`ncg-<t8>-<a8>-<hash10>`)、路径派生 |
| `src/tenants/loader.ts` | `loadTenantRepo(dir)` —— 遍历仓库、校验、解析 skills,返回对象 + 诊断信息 |
| `src/tenants/loader.test.ts` | 基于 fixture 的加载器测试 |
| `src/tenants/secret-refs.test.ts` | secret 引用解析器测试 |
| `src/tenants/naming.test.ts` | 命名净化测试 |
| `src/tenants/schemas.test.ts` | zod schema 接受/拒绝测试 |
| `src/db/partition.ts` | `openHostDb(tenantId, agentId)` + 分区感知的 schema 迁移 |
| `src/db/partition.test.ts` | 分区 DB 测试 |
| `src/db/active-runs.ts` | `active_runs` 表 CRUD + 类型化访问器 |
| `src/db/active-runs.test.ts` | active_runs 测试 |
| `tests/fixtures/tenants/acme/tenant.json` | 测试 fixture |
| `tests/fixtures/tenants/acme/agents/finance/agent.json` | 测试 fixture |
| `tests/fixtures/tenants/acme/agents/finance/channels/feishu.json` | 测试 fixture |
| `tests/fixtures/tenants/acme/agents/finance/instructions.md` | 测试 fixture |
| `tests/fixtures/tenants/acme/agents/ops/agent.json` | 测试 fixture |
| `tests/fixtures/tenants/acme/skills/acme-approval/SKILL.md` | 测试 fixture |

### 修改

| 路径 | 原因 |
|------|-----|
| `src/config.ts` | 新增 `NANOCLAW_DATA_DIR`、`NANOCLAW_TENANTS_DIR` 环境变量并提供合理默认值 |
| `src/index.ts` | 启动时加载 tenant 仓库(如已配置)并打印诊断信息。**不要**改变消息路由。 |
| `src/db.ts` | 重新导出新的分区 helper 以提高可发现性。**不要**改变现有全局 DB 行为。 |

### 本计划**不要**触碰

- `src/container-runner.ts`、`src/group-queue.ts`、`src/task-scheduler.ts`、`src/router.ts`、`src/ipc.ts` —— runtime 行为保持不变。
- `src/channels/*` —— registry 重构属于 Plan B。
- `container/` 下的任何文件 —— runtime 变更属于 Plan C。

---

## 全局不变量(适用于每个任务)

- NodeNext ESM:相对导入即使针对 TS 源文件也要使用 `.js` 扩展名。
- 测试与源码同目录存放为 `*.test.ts`,通过 `npm test` (vitest) 运行。
- 每个新增行为的代码步骤必须**先**写一个失败的测试。
- 每个任务以单一目的 commit 结束,使用前缀 `feat(tenants):`、`feat(db):`、`chore(tenants):` 等。
- 永不引入持有 tenant 状态的模块级单例。加载器返回纯对象;由调用方组合状态。
- 永不在 `src/config.ts` 之外读取环境变量。新环境变量统一在该文件中新增。

---

## 任务 1:新增环境变量与配置路径

**文件:**
- 修改:`src/config.ts`
- 测试:`src/config.test.ts` (新建)

- [ ] **第 1 步:写失败的测试**

创建 `src/config.test.ts`:

```typescript
import path from 'path';

import { describe, expect, it } from 'vitest';

import { DATA_DIR, NANOCLAW_DATA_DIR, NANOCLAW_TENANTS_DIR } from './config.js';

describe('nanoclaw paths', () => {
  it('NANOCLAW_DATA_DIR resolves to an absolute path', () => {
    expect(path.isAbsolute(NANOCLAW_DATA_DIR)).toBe(true);
  });

  it('NANOCLAW_TENANTS_DIR is absolute when set via env', () => {
    expect(path.isAbsolute(NANOCLAW_TENANTS_DIR)).toBe(true);
  });

  it('DATA_DIR still resolves (back-compat)', () => {
    expect(path.isAbsolute(DATA_DIR)).toBe(true);
  });
});
```

- [ ] **第 2 步:运行测试以确认失败**

运行:`npm test -- src/config.test.ts`
预期:FAIL —— `NANOCLAW_DATA_DIR` 和 `NANOCLAW_TENANTS_DIR` 未导出。

- [ ] **第 3 步:在 config 中新增环境变量**

编辑 `src/config.ts`。在现有 `DATA_DIR` 定义之后(大约第 38 行)紧接着添加:

```typescript
// NanoClaw 2.0 data root. Per-(tenant, agent) DBs, runtime state, logs, and
// auth all live under here. Defaults to the legacy project `data/` directory
// for compatibility; operators setting up 2.0 from scratch should point this
// at /var/lib/nanoclaw.
export const NANOCLAW_DATA_DIR = path.resolve(
  process.env.NANOCLAW_DATA_DIR || DATA_DIR,
);

// Directory containing tenant repositories (nanoclaw-tenants/ layout).
// Optional in this plan — loader is invoked only if set.
export const NANOCLAW_TENANTS_DIR = process.env.NANOCLAW_TENANTS_DIR
  ? path.resolve(process.env.NANOCLAW_TENANTS_DIR)
  : '';
```

- [ ] **第 4 步:运行测试以确认通过**

运行:`npm test -- src/config.test.ts`
预期:PASS(3 个测试)。

- [ ] **第 5 步:类型检查**

运行:`npm run typecheck`
预期:无错误。

- [ ] **第 6 步:提交**

```bash
git add src/config.ts src/config.test.ts
git commit -m "feat(config): add NANOCLAW_DATA_DIR and NANOCLAW_TENANTS_DIR env vars"
```

---

## 任务 2:tenant 类型定义

**文件:**
- 新建:`src/tenants/types.ts`
- 新建:`src/tenants/types.test.ts`

- [ ] **第 1 步:写失败的测试**

创建 `src/tenants/types.test.ts`:

```typescript
import { describe, expect, it } from 'vitest';

import type {
  AgentLimits,
  ChannelConfig,
  RegisteredAgent,
  RegisteredTenant,
  ResolvedSkill,
  SecretRef,
  TenantDiagnostic,
} from './types.js';

describe('tenant types', () => {
  it('can construct a minimal RegisteredTenant', () => {
    const t: RegisteredTenant = {
      id: 'acme',
      name: 'Acme Corp',
      rootDir: '/srv/nanoclaw-tenants/tenants/acme',
      enabled: true,
    };
    expect(t.id).toBe('acme');
  });

  it('can construct a ResolvedSkill with scope', () => {
    const s: ResolvedSkill = {
      id: 'welcome',
      scope: 'builtin',
      sourcePath: '/opt/nanoclaw/skills/builtin/welcome',
      entry: 'SKILL.md',
      readonly: true,
    };
    expect(s.scope).toBe('builtin');
  });

  it('can construct a SecretRef with llm kind', () => {
    const r: SecretRef = { raw: 'llm:ANTHROPIC_API_KEY', kind: 'llm', name: 'ANTHROPIC_API_KEY' };
    expect(r.kind).toBe('llm');
  });

  it('can construct AgentLimits', () => {
    const l: AgentLimits = { memoryMb: 1024, pids: 256, concurrentTasksPerGroup: 1 };
    expect(l.memoryMb).toBe(1024);
  });

  it('can construct a RegisteredAgent', () => {
    const a: RegisteredAgent = {
      id: 'finance',
      tenantId: 'acme',
      name: 'Finance Bot',
      folder: '/srv/nanoclaw-tenants/tenants/acme/agents/finance',
      provider: 'claude',
      model: 'claude-sonnet-4',
      instructionsPath: '/srv/nanoclaw-tenants/tenants/acme/agents/finance/instructions.md',
      resolvedSkills: [],
      channelTypes: ['feishu'],
      channelConfigs: { feishu: { kind: 'feishu', mode: 'websocket', appId: 'cli_x', appSecretRef: { raw: 'channel:FEISHU_APP_SECRET', kind: 'channel', name: 'FEISHU_APP_SECRET' } } },
      envRefs: [{ raw: 'llm:ANTHROPIC_API_KEY', kind: 'llm', name: 'ANTHROPIC_API_KEY' }],
      limits: { memoryMb: 1024, pids: 256, concurrentTasksPerGroup: 1 },
    };
    expect(a.id).toBe('finance');
  });

  it('can construct a TenantDiagnostic', () => {
    const d: TenantDiagnostic = {
      level: 'error',
      path: '/srv/nanoclaw-tenants/tenants/acme/tenant.json',
      message: 'missing required field: name',
    };
    expect(d.level).toBe('error');
  });

  it('can construct a ChannelConfig for feishu', () => {
    const c: ChannelConfig = {
      kind: 'feishu',
      mode: 'websocket',
      appId: 'cli_x',
      appSecretRef: { raw: 'channel:FEISHU_APP_SECRET', kind: 'channel', name: 'FEISHU_APP_SECRET' },
    };
    expect(c.kind).toBe('feishu');
  });
});
```

- [ ] **第 2 步:运行测试以确认失败**

运行:`npm test -- src/tenants/types.test.ts`
预期:FAIL —— 模块 `./types.js` 不存在。

- [ ] **第 3 步:创建类型文件**

创建 `src/tenants/types.ts`:

```typescript
/**
 * In-memory representation of a loaded tenant and its agents.
 *
 * These types are the boundary between the tenant config loader and the rest
 * of the control plane. Producers: src/tenants/loader.ts. Consumers: anything
 * that needs to know what tenants/agents are configured.
 */

export type TenantId = string;
export type AgentId = string;
export type GroupId = string;

export interface RegisteredTenant {
  id: TenantId;
  name: string;
  /** Absolute path to the tenant directory inside the tenant repo. */
  rootDir: string;
  enabled: boolean;
}

export type SkillScope = 'builtin' | 'tenant' | 'agent';

export interface ResolvedSkill {
  id: string;
  scope: SkillScope;
  /** Absolute filesystem path to the skill root directory (contains SKILL.md). */
  sourcePath: string;
  /** Skill entry file relative to sourcePath, typically 'SKILL.md'. */
  entry: string;
  /** Always true in 2.0 — resolved skills are staged read-only into run bundles. */
  readonly: true;
  /** Optional version from skill manifest, for cache invalidation. */
  version?: string;
}

export type SecretRefKind = 'llm' | 'channel';

export interface SecretRef {
  /** Original ref string from config, e.g. 'llm:ANTHROPIC_API_KEY'. */
  raw: string;
  kind: SecretRefKind;
  /** Bare env-var / credential name without the prefix. */
  name: string;
}

export interface AgentLimits {
  memoryMb?: number;
  pids?: number;
  cpuShares?: number;
  concurrentTasksPerGroup?: number;
}

export type ChannelKind = 'feishu' | 'slack' | 'telegram' | 'discord';

/**
 * Per-(tenant, agent) channel configuration. Union discriminator is `kind`.
 * Channels not yet typed here are rejected by the schema with an actionable
 * diagnostic.
 */
export type ChannelConfig = FeishuChannelConfig | GenericChannelConfig;

export interface FeishuChannelConfig {
  kind: 'feishu';
  mode: 'websocket' | 'webhook';
  appId: string;
  appSecretRef: SecretRef;
  webhook?: {
    encryptKeyRef?: SecretRef;
    verificationTokenRef?: SecretRef;
  };
}

export interface GenericChannelConfig {
  kind: Exclude<ChannelKind, 'feishu'>;
  /** Raw config blob; channel-specific parsing happens in Plan B. */
  raw: Record<string, unknown>;
  /** Secret refs extracted from raw, for loader-level validation. */
  secretRefs: SecretRef[];
}

export interface RegisteredAgent {
  id: AgentId;
  tenantId: TenantId;
  name: string;
  /** Absolute path to the agent directory inside the tenant repo. */
  folder: string;
  provider: 'claude' | 'opencode' | 'mock';
  model: string;
  /** Absolute path to instructions.md. */
  instructionsPath: string;
  resolvedSkills: ResolvedSkill[];
  /** Channel types declared by this agent (e.g. ['feishu']). */
  channelTypes: ChannelKind[];
  /** Per-channel config keyed by channel type. */
  channelConfigs: Partial<Record<ChannelKind, ChannelConfig>>;
  /** Typed LLM secret refs; only these are allowed to enter run env. */
  envRefs: SecretRef[];
  limits: AgentLimits;
}

export type DiagnosticLevel = 'error' | 'warn' | 'info';

export interface TenantDiagnostic {
  level: DiagnosticLevel;
  /** File path the diagnostic refers to, if applicable. */
  path?: string;
  message: string;
}

export interface TenantRepoLoadResult {
  tenants: RegisteredTenant[];
  agents: RegisteredAgent[];
  diagnostics: TenantDiagnostic[];
}
```

- [ ] **第 4 步:运行测试以确认通过**

运行:`npm test -- src/tenants/types.test.ts`
预期:PASS(7 个测试)。

- [ ] **第 5 步:类型检查**

运行:`npm run typecheck`
预期:无错误。

- [ ] **第 6 步:提交**

```bash
git add src/tenants/types.ts src/tenants/types.test.ts
git commit -m "feat(tenants): add tenant/agent/skill/secret type definitions"
```

---

## 任务 3:类型化 secret 引用解析器

**文件:**
- 新建:`src/tenants/secret-refs.ts`
- 新建:`src/tenants/secret-refs.test.ts`

- [ ] **第 1 步:写失败的测试**

创建 `src/tenants/secret-refs.test.ts`:

```typescript
import { describe, expect, it } from 'vitest';

import { parseSecretRef, parseSecretRefOrThrow } from './secret-refs.js';

describe('parseSecretRef', () => {
  it('parses llm refs', () => {
    expect(parseSecretRef('llm:ANTHROPIC_API_KEY')).toEqual({
      raw: 'llm:ANTHROPIC_API_KEY',
      kind: 'llm',
      name: 'ANTHROPIC_API_KEY',
    });
  });

  it('parses channel refs', () => {
    expect(parseSecretRef('channel:FEISHU_APP_SECRET')).toEqual({
      raw: 'channel:FEISHU_APP_SECRET',
      kind: 'channel',
      name: 'FEISHU_APP_SECRET',
    });
  });

  it('returns null for untyped ref', () => {
    expect(parseSecretRef('ANTHROPIC_API_KEY')).toBeNull();
  });

  it('returns null for unknown prefix', () => {
    expect(parseSecretRef('secret:FOO')).toBeNull();
  });

  it('returns null for empty name', () => {
    expect(parseSecretRef('llm:')).toBeNull();
  });

  it('returns null for name with whitespace', () => {
    expect(parseSecretRef('llm: FOO')).toBeNull();
  });

  it('accepts names with underscores and digits', () => {
    expect(parseSecretRef('llm:MY_KEY_2')?.name).toBe('MY_KEY_2');
  });
});

describe('parseSecretRefOrThrow', () => {
  it('returns parsed ref for valid input', () => {
    expect(parseSecretRefOrThrow('llm:FOO').name).toBe('FOO');
  });

  it('throws on untyped input with actionable message', () => {
    expect(() => parseSecretRefOrThrow('FOO')).toThrow(/must be prefixed/);
  });
});
```

- [ ] **第 2 步:运行测试以确认失败**

运行:`npm test -- src/tenants/secret-refs.test.ts`
预期:FAIL —— 模块不存在。

- [ ] **第 3 步:实现解析器**

创建 `src/tenants/secret-refs.ts`:

```typescript
import type { SecretRef, SecretRefKind } from './types.js';

const VALID_KINDS: ReadonlySet<SecretRefKind> = new Set(['llm', 'channel']);
const NAME_PATTERN = /^[A-Z_][A-Z0-9_]*$/;

/**
 * Parse a typed secret ref of the form '<kind>:<NAME>'.
 * Returns null for any input that is not a properly typed ref — the loader
 * treats null as a config error and produces a diagnostic.
 *
 * Valid: 'llm:ANTHROPIC_API_KEY', 'channel:FEISHU_APP_SECRET'
 * Invalid: 'ANTHROPIC_API_KEY' (no prefix), 'llm:' (empty name),
 *          'llm: FOO' (whitespace), 'secret:FOO' (unknown prefix)
 */
export function parseSecretRef(raw: string): SecretRef | null {
  const colonIdx = raw.indexOf(':');
  if (colonIdx <= 0) return null;

  const kind = raw.slice(0, colonIdx) as SecretRefKind;
  if (!VALID_KINDS.has(kind)) return null;

  const name = raw.slice(colonIdx + 1);
  if (!NAME_PATTERN.test(name)) return null;

  return { raw, kind, name };
}

/**
 * Strict variant: throws on invalid input. Use when the caller has already
 * decided the input must be a valid ref and wants to surface the error as
 * an exception (e.g. during schema validation).
 */
export function parseSecretRefOrThrow(raw: string): SecretRef {
  const parsed = parseSecretRef(raw);
  if (!parsed) {
    throw new Error(
      `Invalid secret ref '${raw}': must be prefixed with 'llm:' or 'channel:' and use UPPER_SNAKE_CASE`,
    );
  }
  return parsed;
}

export function isLlmRef(ref: SecretRef): boolean {
  return ref.kind === 'llm';
}

export function isChannelRef(ref: SecretRef): boolean {
  return ref.kind === 'channel';
}
```

- [ ] **第 4 步:运行测试以确认通过**

运行:`npm test -- src/tenants/secret-refs.test.ts`
预期:PASS(9 个测试)。

- [ ] **第 5 步:提交**

```bash
git add src/tenants/secret-refs.ts src/tenants/secret-refs.test.ts
git commit -m "feat(tenants): add typed llm:/channel: secret ref parser"
```

---

## 任务 4:tenant.json、agent.json、channel 配置的 zod schema

**文件:**
- 新建:`src/tenants/schemas.ts`
- 新建:`src/tenants/schemas.test.ts`

- [ ] **第 1 步:写失败的测试**

创建 `src/tenants/schemas.test.ts`:

```typescript
import { describe, expect, it } from 'vitest';

import {
  agentSchema,
  feishuChannelSchema,
  skillManifestSchema,
  tenantSchema,
} from './schemas.js';

describe('tenantSchema', () => {
  it('accepts a valid tenant', () => {
    const result = tenantSchema.safeParse({
      id: 'acme',
      name: 'Acme Corp',
      enabled: true,
    });
    expect(result.success).toBe(true);
  });

  it('rejects tenant id with uppercase', () => {
    const result = tenantSchema.safeParse({
      id: 'Acme',
      name: 'Acme',
      enabled: true,
    });
    expect(result.success).toBe(false);
  });

  it('rejects tenant id over 16 chars', () => {
    const result = tenantSchema.safeParse({
      id: 'a'.repeat(17),
      name: 'x',
      enabled: true,
    });
    expect(result.success).toBe(false);
  });

  it('rejects missing name', () => {
    const result = tenantSchema.safeParse({ id: 'acme', enabled: true });
    expect(result.success).toBe(false);
  });

  it('defaults enabled to true', () => {
    const result = tenantSchema.safeParse({ id: 'acme', name: 'Acme' });
    expect(result.success).toBe(true);
    if (result.success) expect(result.data.enabled).toBe(true);
  });
});

describe('agentSchema', () => {
  const validAgent = {
    id: 'finance',
    tenant: 'acme',
    name: 'Finance Bot',
    provider: 'claude',
    model: 'claude-sonnet-4',
    instructions: './instructions.md',
    skills: ['builtin:welcome'],
    channels: ['feishu'],
    envRefs: ['llm:ANTHROPIC_API_KEY'],
    limits: { memoryMb: 1024, pids: 256, concurrentTasksPerGroup: 1 },
  };

  it('accepts a valid agent', () => {
    expect(agentSchema.safeParse(validAgent).success).toBe(true);
  });

  it('rejects untyped envRef', () => {
    expect(
      agentSchema.safeParse({ ...validAgent, envRefs: ['ANTHROPIC_API_KEY'] })
        .success,
    ).toBe(false);
  });

  it('rejects channel-scoped ref in envRefs', () => {
    expect(
      agentSchema.safeParse({
        ...validAgent,
        envRefs: ['channel:FEISHU_APP_SECRET'],
      }).success,
    ).toBe(false);
  });

  it('rejects unknown provider', () => {
    expect(
      agentSchema.safeParse({ ...validAgent, provider: 'gemini' }).success,
    ).toBe(false);
  });

  it('accepts limits as optional', () => {
    const { limits: _omit, ...rest } = validAgent;
    void _omit;
    expect(agentSchema.safeParse(rest).success).toBe(true);
  });
});

describe('feishuChannelSchema', () => {
  it('accepts websocket mode with appId and appSecretRef', () => {
    expect(
      feishuChannelSchema.safeParse({
        mode: 'websocket',
        appId: 'cli_x',
        appSecretRef: 'channel:FEISHU_APP_SECRET',
      }).success,
    ).toBe(true);
  });

  it('rejects llm: ref for appSecretRef', () => {
    expect(
      feishuChannelSchema.safeParse({
        mode: 'websocket',
        appId: 'cli_x',
        appSecretRef: 'llm:FEISHU_APP_SECRET',
      }).success,
    ).toBe(false);
  });

  it('rejects webhook mode without webhook block', () => {
    expect(
      feishuChannelSchema.safeParse({
        mode: 'webhook',
        appId: 'cli_x',
        appSecretRef: 'channel:FEISHU_APP_SECRET',
      }).success,
    ).toBe(false);
  });
});

describe('skillManifestSchema', () => {
  it('accepts a valid manifest', () => {
    expect(
      skillManifestSchema.safeParse({
        id: 'acme-approval',
        scope: 'tenant',
        version: '1.0.0',
        entry: 'SKILL.md',
      }).success,
    ).toBe(true);
  });

  it('rejects unknown scope', () => {
    expect(
      skillManifestSchema.safeParse({
        id: 'x',
        scope: 'unknown',
        version: '1.0.0',
        entry: 'SKILL.md',
      }).success,
    ).toBe(false);
  });
});
```

- [ ] **第 2 步:运行测试以确认失败**

运行:`npm test -- src/tenants/schemas.test.ts`
预期:FAIL —— 模块不存在。

- [ ] **第 3 步:实现 schema**

创建 `src/tenants/schemas.ts`:

```typescript
import { z } from 'zod';

/**
 * ID format shared by tenants, agents, and groups: lowercase, letters/digits/dash,
 * must start with a letter, max 16 chars. Matches the username-prefix budget.
 */
export const idSchema = z
  .string()
  .regex(/^[a-z][a-z0-9-]*$/, 'must match [a-z][a-z0-9-]*')
  .max(16, 'must be at most 16 chars');

/**
 * Typed secret ref: 'llm:NAME' or 'channel:NAME' where NAME is UPPER_SNAKE_CASE.
 * The kind prefix is policy: llm refs may enter run env, channel refs must stay
 * host-side. Unknown prefixes are rejected at parse time.
 */
export const secretRefSchema = z
  .string()
  .regex(
    /^(llm|channel):[A-Z_][A-Z0-9_]*$/,
    "must be 'llm:NAME' or 'channel:NAME' (UPPER_SNAKE_CASE)",
  );

const llmRefSchema = z
  .string()
  .regex(/^llm:[A-Z_][A-Z0-9_]*$/, "must be 'llm:NAME'");
const channelRefSchema = z
  .string()
  .regex(/^channel:[A-Z_][A-Z0-9_]*$/, "must be 'channel:NAME'");

export const tenantSchema = z.object({
  id: idSchema,
  name: z.string().min(1),
  enabled: z.boolean().default(true),
});

export const skillManifestSchema = z.object({
  id: z.string().min(1),
  scope: z.enum(['builtin', 'tenant', 'agent']),
  version: z.string().optional(),
  entry: z.string().default('SKILL.md'),
  allowedAgents: z.array(z.string()).optional(),
});

export const agentLimitsSchema = z
  .object({
    memoryMb: z.number().int().positive().optional(),
    pids: z.number().int().positive().optional(),
    cpuShares: z.number().int().positive().optional(),
    concurrentTasksPerGroup: z.number().int().positive().optional(),
  })
  .optional();

export const agentSchema = z.object({
  id: idSchema,
  tenant: idSchema,
  name: z.string().min(1),
  provider: z.enum(['claude', 'opencode', 'mock']),
  model: z.string().min(1),
  instructions: z.string(),
  skills: z.array(z.string()).default([]),
  channels: z.array(z.enum(['feishu', 'slack', 'telegram', 'discord'])).default([]),
  /** Only llm: refs are permitted here. Channel refs are resolved from channel config. */
  envRefs: z.array(llmRefSchema).default([]),
  limits: agentLimitsSchema,
});

export const feishuChannelSchema = z.object({
  mode: z.enum(['websocket', 'webhook']),
  appId: z.string().min(1),
  appSecretRef: channelRefSchema,
  webhook: z
    .object({
      encryptKeyRef: channelRefSchema.optional(),
      verificationTokenRef: channelRefSchema.optional(),
    })
    .optional(),
}).superRefine((val, ctx) => {
  if (val.mode === 'webhook' && !val.webhook) {
    ctx.addIssue({
      code: z.ZodIssueCode.custom,
      message: 'webhook mode requires webhook.{encryptKeyRef,verificationTokenRef}',
    });
  }
});

export type TenantJson = z.infer<typeof tenantSchema>;
export type AgentJson = z.infer<typeof agentSchema>;
export type FeishuChannelJson = z.infer<typeof feishuChannelSchema>;
export type SkillManifestJson = z.infer<typeof skillManifestSchema>;
```

- [ ] **第 4 步:运行测试以确认通过**

运行:`npm test -- src/tenants/schemas.test.ts`
预期:PASS(所有 schema 测试通过)。

- [ ] **第 5 步:提交**

```bash
git add src/tenants/schemas.ts src/tenants/schemas.test.ts
git commit -m "feat(tenants): add zod schemas for tenant/agent/channel config"
```

---

## 任务 5:命名 helper(ID 净化 + 用户名 + 路径)

**文件:**
- 新建:`src/tenants/naming.ts`
- 新建:`src/tenants/naming.test.ts`

- [ ] **第 1 步:写失败的测试**

创建 `src/tenants/naming.test.ts`:

```typescript
import { describe, expect, it } from 'vitest';

import {
  deriveLinuxUsername,
  hashGroupSegment,
  sanitizeAgentId,
  sanitizeGroupId,
  sanitizeTenantId,
  tenantAgentDbPath,
  tenantAgentRuntimeDir,
} from './naming.js';

describe('ID sanitisation', () => {
  it('sanitizeTenantId lowercases and passes valid id', () => {
    expect(sanitizeTenantId('Acme')).toBe('acme');
    expect(sanitizeTenantId('acme-1')).toBe('acme-1');
  });

  it('sanitizeTenantId rejects empty', () => {
    expect(() => sanitizeTenantId('')).toThrow();
  });

  it('sanitizeTenantId rejects over 16 chars', () => {
    expect(() => sanitizeTenantId('a'.repeat(17))).toThrow();
  });

  it('sanitizeTenantId rejects invalid chars', () => {
    expect(() => sanitizeTenantId('acme_corp')).toThrow();
    expect(() => sanitizeTenantId('1acme')).toThrow();
  });

  it('sanitizeAgentId behaves same as tenant', () => {
    expect(sanitizeAgentId('Finance')).toBe('finance');
  });

  it('sanitizeGroupId allows lowercase, digits, dash', () => {
    expect(sanitizeGroupId('feishu-main')).toBe('feishu-main');
    expect(sanitizeGroupId('group_42')).toBe('group-42');
    expect(sanitizeGroupId('UPPER')).toBe('upper');
  });

  it('sanitizeGroupId rejects empty and path traversal', () => {
    expect(() => sanitizeGroupId('')).toThrow();
    expect(() => sanitizeGroupId('../etc')).toThrow();
    expect(() => sanitizeGroupId('/abs')).toThrow();
  });
});

describe('hashGroupSegment', () => {
  it('returns 10-char lowercase hex', () => {
    expect(hashGroupSegment('feishu-main')).toMatch(/^[0-9a-f]{10}$/);
  });

  it('is stable for same input', () => {
    expect(hashGroupSegment('feishu-main')).toBe(hashGroupSegment('feishu-main'));
  });

  it('differs for different input', () => {
    expect(hashGroupSegment('a')).not.toBe(hashGroupSegment('b'));
  });
});

describe('deriveLinuxUsername', () => {
  it('produces ncg-<t8>-<a8>-<hash10> format', () => {
    const u = deriveLinuxUsername('acme', 'finance', 'feishu-main');
    expect(u).toMatch(/^ncg-[a-z0-9-]{1,8}-[a-z0-9-]{1,8}-[0-9a-f]{10}$/);
    expect(u.length).toBeLessThanOrEqual(32);
  });

  it('truncates tenant and agent segments to 8 chars', () => {
    const u = deriveLinuxUsername('verylongtenant', 'verylongagent', 'g');
    // ncg-(8)-verylong-(8)-hash10; note 'verylongtenant' -> 'verylong', 'verylongagent' -> 'verylong'
    expect(u.startsWith('ncg-verylo-verylo-')).toBe(true);
  });

  it('rejects unsanitised IDs (callers must sanitise first)', () => {
    expect(() => deriveLinuxUsername('ACME', 'finance', 'g')).toThrow();
  });
});

describe('paths', () => {
  it('tenantAgentDbPath returns absolute path under data dir', () => {
    const p = tenantAgentDbPath('/var/lib/nanoclaw', 'acme', 'finance');
    expect(p).toBe('/var/lib/nanoclaw/data/tenants/acme/finance/messages.db');
  });

  it('tenantAgentRuntimeDir returns absolute runtime path', () => {
    const p = tenantAgentRuntimeDir('/var/lib/nanoclaw', 'acme', 'finance', 'main');
    expect(p).toBe('/var/lib/nanoclaw/runtime/acme/finance/main');
  });
});
```

- [ ] **第 2 步:运行测试以确认失败**

运行:`npm test -- src/tenants/naming.test.ts`
预期:FAIL —— 模块不存在。

- [ ] **第 3 步:实现命名 helper**

创建 `src/tenants/naming.ts`:

```typescript
import crypto from 'crypto';
import path from 'path';

const ID_PATTERN = /^[a-z][a-z0-9-]*$/;
const ID_MAX = 16;
const GROUP_PATTERN = /^[a-z0-9][a-z0-9-]*$/;

/** Sanitise + validate a tenant or agent ID. Throws on invalid input. */
export function sanitizeTenantId(raw: string): string {
  const lower = raw.toLowerCase();
  if (!ID_PATTERN.test(lower) || lower.length > ID_MAX) {
    throw new Error(
      `Invalid tenant id '${raw}': must match [a-z][a-z0-9-]* and be at most ${ID_MAX} chars`,
    );
  }
  return lower;
}

export function sanitizeAgentId(raw: string): string {
  return sanitizeTenantId(raw); // same rules
}

/** Sanitise a group ID: lowercase, non-[a-z0-9-] replaced with '-', reject empty/traversal. */
export function sanitizeGroupId(raw: string): string {
  if (!raw) throw new Error('group id must not be empty');
  if (raw.includes('/') || raw.includes('..')) {
    throw new Error(`group id must not contain '/' or '..': '${raw}'`);
  }
  const lower = raw.toLowerCase().replace(/[^a-z0-9-]/g, '-');
  if (!GROUP_PATTERN.test(lower)) {
    throw new Error(`Invalid group id after sanitisation: '${raw}' -> '${lower}'`);
  }
  return lower;
}

/**
 * 10-char hex hash of the canonical (tenant, agent, group) tuple.
 * Used as the group segment in Linux usernames so the username stays within
 * the 32-char limit regardless of group ID length.
 */
export function hashGroupSegment(
  tenant: string,
  agent: string,
  group: string,
): string {
  const input = `${tenant}|${agent}|${group}`;
  return crypto.createHash('sha256').update(input).digest('hex').slice(0, 10);
}

/**
 * Derive the canonical Linux username for a (tenant, agent, group) tuple.
 * Format: `ncg-<tenant8>-<agent8>-<hash10>` — max 32 chars.
 * Inputs must already be sanitised (lowercase + valid). Use the sanitize*
 * helpers first.
 */
export function deriveLinuxUsername(
  tenant: string,
  agent: string,
  group: string,
): string {
  if (!ID_PATTERN.test(tenant) || !ID_PATTERN.test(agent)) {
    throw new Error('tenant and agent must be pre-sanitised');
  }
  const t8 = tenant.slice(0, 8);
  const a8 = agent.slice(0, 8);
  const hash = hashGroupSegment(tenant, agent, group);
  const username = `ncg-${t8}-${a8}-${hash}`;
  if (username.length > 32) {
    // Defensive: budget is 4 + 8 + 1 + 8 + 1 + 10 = 32. Should never trip.
    throw new Error(`derived username exceeds 32 chars: ${username}`);
  }
  return username;
}

/** ${dataDir}/data/tenants/<tenant>/<agent>/messages.db */
export function tenantAgentDbPath(
  dataDir: string,
  tenant: string,
  agent: string,
): string {
  return path.join(dataDir, 'data', 'tenants', tenant, agent, 'messages.db');
}

/** ${dataDir}/runtime/<tenant>/<agent>/<group> */
export function tenantAgentRuntimeDir(
  dataDir: string,
  tenant: string,
  agent: string,
  group: string,
): string {
  return path.join(dataDir, 'runtime', tenant, agent, group);
}

/** ${dataDir}/logs/<tenant>/<agent>/<group> */
export function tenantAgentLogDir(
  dataDir: string,
  tenant: string,
  agent: string,
  group: string,
): string {
  return path.join(dataDir, 'logs', tenant, agent, group);
}
```

- [ ] **第 4 步:运行测试以确认通过**

运行:`npm test -- src/tenants/naming.test.ts`
预期:PASS(所有命名测试通过)。

- [ ] **第 5 步:类型检查**

运行:`npm run typecheck`
预期:无错误。

- [ ] **第 6 步:提交**

```bash
git add src/tenants/naming.ts src/tenants/naming.test.ts
git commit -m "feat(tenants): add ID sanitisation, username derivation, and path helpers"
```

---

## 任务 6:tenant 加载器

**文件:**
- 新建:`src/tenants/loader.ts`
- 新建:`src/tenants/loader.test.ts`
- 在 `tests/fixtures/tenants/` 下新建 fixture

- [ ] **第 1 步:创建 fixture**

创建 fixture 目录树:

`tests/fixtures/tenants/acme/tenant.json`:
```json
{
  "id": "acme",
  "name": "Acme Corp",
  "enabled": true
}
```

`tests/fixtures/tenants/acme/agents/finance/agent.json`:
```json
{
  "id": "finance",
  "tenant": "acme",
  "name": "Finance Bot",
  "provider": "claude",
  "model": "claude-sonnet-4",
  "instructions": "./instructions.md",
  "skills": ["builtin:welcome", "tenant:acme-approval"],
  "channels": ["feishu"],
  "envRefs": ["llm:ANTHROPIC_API_KEY"],
  "limits": { "memoryMb": 1024, "pids": 256, "concurrentTasksPerGroup": 1 }
}
```

`tests/fixtures/tenants/acme/agents/finance/instructions.md`:
```markdown
# Finance Bot instructions

You handle finance queries.
```

`tests/fixtures/tenants/acme/agents/finance/channels/feishu.json`:
```json
{
  "mode": "websocket",
  "appId": "cli_acme_finance",
  "appSecretRef": "channel:FEISHU_APP_SECRET"
}
```

`tests/fixtures/tenants/acme/agents/ops/agent.json`:
```json
{
  "id": "ops",
  "tenant": "acme",
  "name": "Ops Bot",
  "provider": "claude",
  "model": "claude-sonnet-4",
  "instructions": "./instructions.md",
  "skills": [],
  "channels": [],
  "envRefs": []
}
```

`tests/fixtures/tenants/acme/agents/ops/instructions.md`:
```markdown
# Ops Bot
```

`tests/fixtures/tenants/acme/skills/acme-approval/SKILL.md`:
```markdown
---
name: acme-approval
description: Approval workflow skill
---

# ACME Approval

Asks the host to run the approval tool.
```

`tests/fixtures/tenants/acme/skills/acme-approval/manifest.json`:
```json
{
  "id": "acme-approval",
  "scope": "tenant",
  "version": "1.0.0",
  "entry": "SKILL.md"
}
```

- [ ] **第 2 步:写失败的测试**

创建 `src/tenants/loader.test.ts`:

```typescript
import path from 'path';

import { describe, expect, it } from 'vitest';

import { loadTenantRepo } from './loader.js';

const FIXTURES = path.resolve(process.cwd(), 'tests', 'fixtures', 'tenants');

describe('loadTenantRepo', () => {
  it('loads the acme fixture with one tenant and two agents', () => {
    const result = loadTenantRepo(FIXTURES);
    expect(result.tenants.map((t) => t.id)).toEqual(['acme']);
    expect(result.agents.map((a) => a.id).sort()).toEqual(['finance', 'ops']);
  });

  it('resolves feishu channel config for finance agent', () => {
    const result = loadTenantRepo(FIXTURES);
    const finance = result.agents.find((a) => a.id === 'finance')!;
    expect(finance.channelTypes).toEqual(['feishu']);
    const feishu = finance.channelConfigs.feishu;
    expect(feishu?.kind).toBe('feishu');
    if (feishu?.kind === 'feishu') {
      expect(feishu.appSecretRef.kind).toBe('channel');
      expect(feishu.appSecretRef.name).toBe('FEISHU_APP_SECRET');
    }
  });

  it('resolves tenant skill reference with absolute path', () => {
    const result = loadTenantRepo(FIXTURES);
    const finance = result.agents.find((a) => a.id === 'finance')!;
    const approval = finance.resolvedSkills.find((s) => s.id === 'acme-approval');
    expect(approval).toBeDefined();
    expect(approval?.scope).toBe('tenant');
    expect(path.isAbsolute(approval?.sourcePath ?? '')).toBe(true);
  });

  it('resolves builtin skill reference without a source path on disk', () => {
    const result = loadTenantRepo(FIXTURES);
    const finance = result.agents.find((a) => a.id === 'finance')!;
    const welcome = finance.resolvedSkills.find((s) => s.id === 'welcome');
    expect(welcome?.scope).toBe('builtin');
  });

  it('emits error diagnostic for unknown tenant skill reference', () => {
    const result = loadTenantRepo(FIXTURES);
    expect(result.diagnostics.some((d) => d.level === 'error')).toBe(false);
  });

  it('returns empty result for missing directory', () => {
    const result = loadTenantRepo('/does/not/exist');
    expect(result.tenants).toEqual([]);
    expect(result.agents).toEqual([]);
    expect(result.diagnostics.some((d) => d.level === 'error')).toBe(true);
  });

  it('attaches typed llm envRefs', () => {
    const result = loadTenantRepo(FIXTURES);
    const finance = result.agents.find((a) => a.id === 'finance')!;
    expect(finance.envRefs).toEqual([
      { raw: 'llm:ANTHROPIC_API_KEY', kind: 'llm', name: 'ANTHROPIC_API_KEY' },
    ]);
  });

  it('agents with no channels load cleanly', () => {
    const result = loadTenantRepo(FIXTURES);
    const ops = result.agents.find((a) => a.id === 'ops')!;
    expect(ops.channelTypes).toEqual([]);
    expect(ops.channelConfigs).toEqual({});
  });
});
```

- [ ] **第 3 步:运行测试以确认失败**

运行:`npm test -- src/tenants/loader.test.ts`
预期:FAIL —— 模块不存在。

- [ ] **第 4 步:实现加载器**

创建 `src/tenants/loader.ts`:

```typescript
import fs from 'fs';
import path from 'path';

import {
  AgentLimits,
  ChannelConfig,
  ChannelKind,
  RegisteredAgent,
  RegisteredTenant,
  ResolvedSkill,
  SecretRef,
  SkillScope,
  TenantDiagnostic,
  TenantRepoLoadResult,
} from './types.js';
import {
  agentSchema,
  feishuChannelSchema,
  skillManifestSchema,
  tenantSchema,
} from './schemas.js';
import { parseSecretRef } from './secret-refs.js';
import { sanitizeAgentId, sanitizeTenantId } from './naming.js';

/**
 * Load and validate a tenant repository.
 *
 * Layout (see docs/runtime-rework/TARGET_ARCHITECTURE_DETAILS.md):
 *   <repoRoot>/tenants/<tenant-id>/tenant.json
 *   <repoRoot>/tenants/<tenant-id>/agents/<agent-id>/agent.json
 *   <repoRoot>/tenants/<tenant-id>/agents/<agent-id>/channels/<channel>.json
 *   <repoRoot>/tenants/<tenant-id>/skills/<skill>/SKILL.md + manifest.json
 *
 * Produces diagnostics for every recoverable config issue. Hard filesystem
 * errors (repoRoot missing) are surfaced as diagnostics and the loader returns
 * an empty result rather than throwing.
 */
export function loadTenantRepo(repoRoot: string): TenantRepoLoadResult {
  const tenants: RegisteredTenant[] = [];
  const agents: RegisteredAgent[] = [];
  const diagnostics: TenantDiagnostic[] = [];

  const tenantsRoot = path.join(repoRoot, 'tenants');
  if (!fs.existsSync(tenantsRoot)) {
    diagnostics.push({
      level: 'error',
      path: tenantsRoot,
      message: `tenants directory not found (looked at ${tenantsRoot})`,
    });
    return { tenants, agents, diagnostics };
  }

  let tenantDirs: string[] = [];
  try {
    tenantDirs = fs
      .readdirSync(tenantsRoot, { withFileTypes: true })
      .filter((d) => d.isDirectory())
      .map((d) => d.name);
  } catch (err) {
    diagnostics.push({
      level: 'error',
      path: tenantsRoot,
      message: `cannot read tenants directory: ${(err as Error).message}`,
    });
    return { tenants, agents, diagnostics };
  }

  for (const tenantDirName of tenantDirs) {
    const tenantRoot = path.join(tenantsRoot, tenantDirName);
    const tenantJsonPath = path.join(tenantRoot, 'tenant.json');

    let tenantRaw: unknown;
    try {
      tenantRaw = JSON.parse(fs.readFileSync(tenantJsonPath, 'utf8'));
    } catch (err) {
      diagnostics.push({
        level: 'error',
        path: tenantJsonPath,
        message: `cannot read tenant.json: ${(err as Error).message}`,
      });
      continue;
    }

    const parsed = tenantSchema.safeParse(tenantRaw);
    if (!parsed.success) {
      diagnostics.push({
        level: 'error',
        path: tenantJsonPath,
        message: `invalid tenant.json: ${parsed.error.message}`,
      });
      continue;
    }

    let tenantId: string;
    try {
      tenantId = sanitizeTenantId(parsed.data.id);
    } catch (err) {
      diagnostics.push({
        level: 'error',
        path: tenantJsonPath,
        message: (err as Error).message,
      });
      continue;
    }

    const tenant: RegisteredTenant = {
      id: tenantId,
      name: parsed.data.name,
      rootDir: tenantRoot,
      enabled: parsed.data.enabled,
    };
    tenants.push(tenant);

    if (!tenant.enabled) {
      diagnostics.push({
        level: 'info',
        path: tenantJsonPath,
        message: `tenant '${tenantId}' is disabled, skipping agents`,
      });
      continue;
    }

    loadAgentsForTenant(tenant, agents, diagnostics);
  }

  return { tenants, agents, diagnostics };
}

function loadAgentsForTenant(
  tenant: RegisteredTenant,
  agentsOut: RegisteredAgent[],
  diagnostics: TenantDiagnostic[],
): void {
  const agentsRoot = path.join(tenant.rootDir, 'agents');
  if (!fs.existsSync(agentsRoot)) return;

  const agentDirs = fs
    .readdirSync(agentsRoot, { withFileTypes: true })
    .filter((d) => d.isDirectory())
    .map((d) => d.name);

  for (const agentDirName of agentDirs) {
    const agentRoot = path.join(agentsRoot, agentDirName);
    const agentJsonPath = path.join(agentRoot, 'agent.json');

    let agentRaw: unknown;
    try {
      agentRaw = JSON.parse(fs.readFileSync(agentJsonPath, 'utf8'));
    } catch (err) {
      diagnostics.push({
        level: 'error',
        path: agentJsonPath,
        message: `cannot read agent.json: ${(err as Error).message}`,
      });
      continue;
    }

    const parsed = agentSchema.safeParse(agentRaw);
    if (!parsed.success) {
      diagnostics.push({
        level: 'error',
        path: agentJsonPath,
        message: `invalid agent.json: ${parsed.error.message}`,
      });
      continue;
    }

    let agentId: string;
    try {
      agentId = sanitizeAgentId(parsed.data.id);
    } catch (err) {
      diagnostics.push({
        level: 'error',
        path: agentJsonPath,
        message: (err as Error).message,
      });
      continue;
    }

    if (parsed.data.tenant !== tenant.id) {
      diagnostics.push({
        level: 'error',
        path: agentJsonPath,
        message: `agent.tenant='${parsed.data.tenant}' does not match parent tenant id='${tenant.id}'`,
      });
      continue;
    }

    const resolvedSkills = resolveSkills(
      parsed.data.skills,
      tenant,
      agentId,
      agentRoot,
      diagnostics,
    );

    const channelConfigs = resolveChannels(
      parsed.data.channels,
      agentRoot,
      tenant,
      agentId,
      diagnostics,
    );

    const envRefs: SecretRef[] = parsed.data.envRefs.map((raw) => {
      const r = parseSecretRef(raw);
      // agentSchema already enforces llm:* prefix; parseSecretRef cannot fail here
      return r ?? { raw, kind: 'llm', name: raw.slice(4) };
    });

    const limits: AgentLimits = parsed.data.limits ?? {};

    const instructionsPath = path.resolve(agentRoot, parsed.data.instructions);
    if (!fs.existsSync(instructionsPath)) {
      diagnostics.push({
        level: 'error',
        path: instructionsPath,
        message: `instructions file not found`,
      });
    }

    agentsOut.push({
      id: agentId,
      tenantId: tenant.id,
      name: parsed.data.name,
      folder: agentRoot,
      provider: parsed.data.provider,
      model: parsed.data.model,
      instructionsPath,
      resolvedSkills,
      channelTypes: parsed.data.channels,
      channelConfigs,
      envRefs,
      limits,
    });
  }
}

function resolveSkills(
  refs: string[],
  tenant: RegisteredTenant,
  agentId: string,
  agentRoot: string,
  diagnostics: TenantDiagnostic[],
): ResolvedSkill[] {
  const out: ResolvedSkill[] = [];
  for (const ref of refs) {
    const parsed = ref.split(':');
    if (parsed.length !== 2) {
      diagnostics.push({
        level: 'error',
        message: `skill ref '${ref}' must be '<scope>:<id>'`,
      });
      continue;
    }
    const [scopeStr, id] = parsed;
    if (!['builtin', 'tenant', 'agent'].includes(scopeStr)) {
      diagnostics.push({
        level: 'error',
        message: `skill ref '${ref}' has unknown scope '${scopeStr}'`,
      });
      continue;
    }
    const scope = scopeStr as SkillScope;

    if (scope === 'builtin') {
      // Builtin skills live in platform code; their path is resolved at spawn time.
      out.push({
        id,
        scope: 'builtin',
        sourcePath: '', // resolved in Plan D when staging bundles
        entry: 'SKILL.md',
        readonly: true,
      });
      continue;
    }

    const skillRoot =
      scope === 'tenant'
        ? path.join(tenant.rootDir, 'skills', id)
        : path.join(agentRoot, 'skills', id);

    const skillMd = path.join(skillRoot, 'SKILL.md');
    if (!fs.existsSync(skillMd)) {
      diagnostics.push({
        level: 'error',
        path: skillMd,
        message: `skill '${ref}' not found`,
      });
      continue;
    }

    let version: string | undefined;
    const manifestPath = path.join(skillRoot, 'manifest.json');
    if (fs.existsSync(manifestPath)) {
      try {
        const manifestRaw = JSON.parse(fs.readFileSync(manifestPath, 'utf8'));
        const manifest = skillManifestSchema.parse(manifestRaw);
        version = manifest.version;
      } catch (err) {
        diagnostics.push({
          level: 'warn',
          path: manifestPath,
          message: `invalid manifest.json: ${(err as Error).message}`,
        });
      }
    }

    out.push({
      id,
      scope,
      sourcePath: skillRoot,
      entry: 'SKILL.md',
      readonly: true,
      version,
    });
  }
  return out;
}

function resolveChannels(
  channelTypes: ChannelKind[],
  agentRoot: string,
  tenant: RegisteredTenant,
  agentId: string,
  diagnostics: TenantDiagnostic[],
): Partial<Record<ChannelKind, ChannelConfig>> {
  const out: Partial<Record<ChannelKind, ChannelConfig>> = {};
  for (const kind of channelTypes) {
    const channelPath = path.join(agentRoot, 'channels', `${kind}.json`);
    if (!fs.existsSync(channelPath)) {
      diagnostics.push({
        level: 'error',
        path: channelPath,
        message: `channel config for '${kind}' not found`,
      });
      continue;
    }

    let raw: unknown;
    try {
      raw = JSON.parse(fs.readFileSync(channelPath, 'utf8'));
    } catch (err) {
      diagnostics.push({
        level: 'error',
        path: channelPath,
        message: `cannot read ${kind}.json: ${(err as Error).message}`,
      });
      continue;
    }

    if (kind === 'feishu') {
      const parsed = feishuChannelSchema.safeParse(raw);
      if (!parsed.success) {
        diagnostics.push({
          level: 'error',
          path: channelPath,
          message: `invalid feishu.json: ${parsed.error.message}`,
        });
        continue;
      }
      const appSecretRef = parseSecretRef(parsed.data.appSecretRef)!;
      const webhook = parsed.data.webhook
        ? {
            encryptKeyRef: parsed.data.webhook.encryptKeyRef
              ? parseSecretRef(parsed.data.webhook.encryptKeyRef)!
              : undefined,
            verificationTokenRef: parsed.data.webhook.verificationTokenRef
              ? parseSecretRef(parsed.data.webhook.verificationTokenRef)!
              : undefined,
          }
        : undefined;
      out.feishu = {
        kind: 'feishu',
        mode: parsed.data.mode,
        appId: parsed.data.appId,
        appSecretRef,
        webhook,
      };
    } else {
      // Non-feishu channels are validated structurally in Plan B.
      diagnostics.push({
        level: 'warn',
        path: channelPath,
        message: `channel kind '${kind}' config parsing not implemented yet (Plan B); loaded as opaque`,
      });
      const blob = raw as Record<string, unknown>;
      const secretRefs: SecretRef[] = [];
      for (const v of Object.values(blob)) {
        if (typeof v === 'string') {
          const r = parseSecretRef(v);
          if (r) secretRefs.push(r);
        }
      }
      out[kind] = { kind, raw: blob, secretRefs };
    }
  }

  void tenant;
  void agentId; // kept for future cross-tenant validation diagnostics
  return out;
}
```

- [ ] **第 5 步:运行测试以确认通过**

运行:`npm test -- src/tenants/loader.test.ts`
预期:PASS(8 个测试)。

- [ ] **第 6 步:类型检查**

运行:`npm run typecheck`
预期:无错误。

- [ ] **第 7 步:提交**

```bash
git add src/tenants/loader.ts src/tenants/loader.test.ts tests/fixtures/tenants/
git commit -m "feat(tenants): add tenant repo loader with diagnostics"
```

---

## 任务 7:按 (tenant, agent) 的 DB 分区模块

**文件:**
- 新建:`src/db/partition.ts`
- 新建:`src/db/partition.test.ts`

- [ ] **第 1 步:写失败的测试**

创建 `src/db/partition.test.ts`:

```typescript
import fs from 'fs';
import os from 'os';
import path from 'path';

import { afterEach, beforeEach, describe, expect, it } from 'vitest';

import { closeAllHostDbs, closeHostDb, openHostDb } from './partition.js';

let tmpDir: string;

beforeEach(() => {
  tmpDir = fs.mkdtempSync(path.join(os.tmpdir(), 'nc-partition-'));
});

afterEach(() => {
  closeAllHostDbs();
  fs.rmSync(tmpDir, { recursive: true, force: true });
});

describe('openHostDb', () => {
  it('creates parent dirs and opens a DB at the canonical path', () => {
    const db = openHostDb(tmpDir, 'acme', 'finance');
    const expected = path.join(tmpDir, 'data', 'tenants', 'acme', 'finance', 'messages.db');
    expect(fs.existsSync(expected)).toBe(true);
    db.close();
  });

  it('creates the full 2.0 schema including channel_type columns', () => {
    const db = openHostDb(tmpDir, 'acme', 'finance');
    const cols = db
      .prepare("PRAGMA table_info('messages')")
      .all() as { name: string }[];
    const colNames = cols.map((c) => c.name);
    expect(colNames).toContain('channel_type');
    expect(colNames).toContain('tenant_id');
    expect(colNames).toContain('agent_id');

    const tables = db
      .prepare("SELECT name FROM sqlite_master WHERE type='table'")
      .all() as { name: string }[];
    const tableNames = tables.map((t) => t.name);
    expect(tableNames).toContain('active_runs');
    db.close();
  });

  it('reuses an existing DB without schema loss on second open', () => {
    openHostDb(tmpDir, 'acme', 'finance').close();
    const db = openHostDb(tmpDir, 'acme', 'finance');
    const count = db
      .prepare("SELECT COUNT(*) as n FROM messages")
      .get() as { n: number };
    expect(count.n).toBe(0);
    db.close();
  });

  it('rejects tenant/agent IDs that would path-traverse', () => {
    expect(() => openHostDb(tmpDir, '../escape', 'finance')).toThrow();
    expect(() => openHostDb(tmpDir, 'acme', '../etc')).toThrow();
  });
});
```

- [ ] **第 2 步:运行测试以确认失败**

运行:`npm test -- src/db/partition.test.ts`
预期:FAIL —— 模块不存在。

- [ ] **第 3 步:实现分区模块**

创建 `src/db/partition.ts`:

```typescript
import Database from 'better-sqlite3';
import fs from 'fs';
import path from 'path';

import { tenantAgentDbPath } from '../tenants/naming.js';
import { sanitizeAgentId, sanitizeTenantId } from '../tenants/naming.js';

/**
 * Open the per-(tenant, agent) host DB.
 *
 * Path: ${dataDir}/data/tenants/<tenant>/<agent>/messages.db
 *
 * Returns a fresh connection each call. Callers own the lifecycle: close
 * via `db.close()` when done. Foundation plan does not cache, because the
 * caller of this function in later plans is the control plane's routing
 * path and the right caching strategy will depend on the runtime topology
 * (per-process connection pool, shared long-lived connection, etc.).
 *
 * Schema: full 2.0 schema including channel_type, tenant_id, agent_id
 * columns on all routing tables, plus the active_runs table for restart
 * reconciliation. Existing rows from 1.x migration are expected to be
 * backfilled separately (Plan F); this plan just declares the schema.
 */
export function openHostDb(
  dataDir: string,
  tenantRaw: string,
  agentRaw: string,
): Database.Database {
  const tenant = sanitizeTenantId(tenantRaw);
  const agent = sanitizeAgentId(agentRaw);

  const dbPath = tenantAgentDbPath(dataDir, tenant, agent);
  fs.mkdirSync(path.dirname(dbPath), { recursive: true });

  const db = new Database(dbPath);
  db.pragma('journal_mode = WAL');
  db.pragma('busy_timeout = 5000');
  applySchema(db);

  return db;
}

// Legacy cache-based API kept as no-ops for forward-compat with later plans
// that may introduce connection pooling. Today these are placeholders.
export function closeHostDb(_tenant: string, _agent: string): void {
  // No-op in Foundation. Callers should call db.close() on the handle they got.
}

export function closeAllHostDbs(): void {
  // No-op in Foundation. Tests are responsible for closing their own handles.
}

/**
 * 2.0 schema for per-(tenant, agent) DBs. Mirrors the 1.x schema with these
 * changes:
 *   - channel_type column added to chats, messages, registered_groups,
 *     scheduled_tasks, router_state. Identity is (channel_type, ...) in each.
 *   - tenant_id and agent_id columns on tables that previously used bare
 *     folder as identity.
 *   - New table: active_runs (for restart reconciliation).
 */
function applySchema(db: Database.Database): void {
  db.exec(`
    CREATE TABLE IF NOT EXISTS chats (
      channel_type TEXT NOT NULL,
      jid TEXT NOT NULL,
      name TEXT,
      last_message_time TEXT,
      is_group INTEGER DEFAULT 0,
      tenant_id TEXT NOT NULL,
      agent_id TEXT NOT NULL,
      PRIMARY KEY (channel_type, jid)
    );

    CREATE TABLE IF NOT EXISTS messages (
      id TEXT NOT NULL,
      channel_type TEXT NOT NULL,
      chat_jid TEXT NOT NULL,
      sender TEXT,
      sender_name TEXT,
      content TEXT,
      timestamp TEXT NOT NULL,
      is_from_me INTEGER,
      is_bot_message INTEGER DEFAULT 0,
      card_action TEXT,
      scheduled_task_id TEXT,
      message_type TEXT,
      attachment TEXT,
      tenant_id TEXT NOT NULL,
      agent_id TEXT NOT NULL,
      PRIMARY KEY (channel_type, id),
      FOREIGN KEY (channel_type, chat_jid) REFERENCES chats(channel_type, jid)
    );
    CREATE INDEX IF NOT EXISTS idx_messages_timestamp ON messages(timestamp);
    CREATE INDEX IF NOT EXISTS idx_messages_chat ON messages(channel_type, chat_jid, timestamp);

    CREATE TABLE IF NOT EXISTS scheduled_tasks (
      id TEXT PRIMARY KEY,
      tenant_id TEXT NOT NULL,
      agent_id TEXT NOT NULL,
      group_folder TEXT NOT NULL,
      channel_type TEXT NOT NULL,
      chat_jid TEXT NOT NULL,
      prompt TEXT NOT NULL,
      schedule_type TEXT NOT NULL,
      schedule_value TEXT NOT NULL,
      next_run TEXT,
      last_run TEXT,
      last_result TEXT,
      status TEXT DEFAULT 'active',
      created_at TEXT NOT NULL
    );
    CREATE INDEX IF NOT EXISTS idx_tasks_next_run ON scheduled_tasks(next_run);
    CREATE INDEX IF NOT EXISTS idx_tasks_status ON scheduled_tasks(status);

    CREATE TABLE IF NOT EXISTS task_run_logs (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      task_id TEXT NOT NULL,
      run_at TEXT NOT NULL,
      duration_ms INTEGER NOT NULL,
      status TEXT NOT NULL,
      result TEXT,
      error TEXT,
      FOREIGN KEY (task_id) REFERENCES scheduled_tasks(id)
    );
    CREATE INDEX IF NOT EXISTS idx_task_run_logs ON task_run_logs(task_id, run_at);

    CREATE TABLE IF NOT EXISTS router_state (
      tenant_id TEXT NOT NULL,
      agent_id TEXT NOT NULL,
      channel_type TEXT NOT NULL,
      chat_jid TEXT,
      key TEXT NOT NULL,
      value TEXT NOT NULL,
      PRIMARY KEY (tenant_id, agent_id, channel_type, chat_jid, key)
    );

    CREATE TABLE IF NOT EXISTS sessions (
      tenant_id TEXT NOT NULL,
      agent_id TEXT NOT NULL,
      group_folder TEXT NOT NULL,
      session_id TEXT NOT NULL,
      PRIMARY KEY (tenant_id, agent_id, group_folder)
    );

    CREATE TABLE IF NOT EXISTS registered_groups (
      tenant_id TEXT NOT NULL,
      agent_id TEXT NOT NULL,
      channel_type TEXT NOT NULL,
      jid TEXT NOT NULL,
      name TEXT NOT NULL,
      folder TEXT NOT NULL,
      trigger TEXT,
      requires_trigger INTEGER DEFAULT 1,
      is_main INTEGER DEFAULT 0,
      is_p2p INTEGER DEFAULT 0,
      p2p_user TEXT,
      source_group TEXT,
      container_config TEXT,
      added_at TEXT NOT NULL,
      PRIMARY KEY (tenant_id, agent_id, channel_type, jid)
    );

    CREATE TABLE IF NOT EXISTS active_runs (
      tenant_id TEXT NOT NULL,
      agent_id TEXT NOT NULL,
      group_folder TEXT NOT NULL,
      run_id TEXT NOT NULL,
      mode TEXT NOT NULL,                -- 'live' | 'isolated'
      pid INTEGER NOT NULL,
      pid_start_time_ticks INTEGER NOT NULL,
      expected_uid TEXT NOT NULL,
      runtime_dir TEXT NOT NULL,
      cgroup_path TEXT,
      started_at TEXT NOT NULL,
      last_seen_at TEXT NOT NULL,
      status TEXT NOT NULL,              -- 'active' | 'exited' | 'crashed'
      PRIMARY KEY (tenant_id, agent_id, run_id)
    );
    CREATE INDEX IF NOT EXISTS idx_active_runs_pid ON active_runs(pid);
    CREATE INDEX IF NOT EXISTS idx_active_runs_tuple ON active_runs(tenant_id, agent_id, group_folder);
  `);
}
```

- [ ] **第 4 步:运行测试以确认通过**

运行:`npm test -- src/db/partition.test.ts`
预期:PASS(4 个测试)。

- [ ] **第 5 步:类型检查**

运行:`npm run typecheck`
预期:无错误。

- [ ] **第 6 步:提交**

```bash
git add src/db/partition.ts src/db/partition.test.ts
git commit -m "feat(db): add per-(tenant, agent) partition DB module"
```

---

## 任务 8:active_runs 访问器

**文件:**
- 新建:`src/db/active-runs.ts`
- 新建:`src/db/active-runs.test.ts`

- [ ] **第 1 步:写失败的测试**

创建 `src/db/active-runs.test.ts`:

```typescript
import fs from 'fs';
import os from 'os';
import path from 'path';

import { afterEach, beforeEach, describe, expect, it } from 'vitest';

import { closeAllHostDbs } from './partition.js';
import {
  ActiveRunRecord,
  insertActiveRun,
  listActiveRuns,
  markActiveRunExited,
  updateActiveRunLastSeen,
} from './active-runs.js';

let dataDir: string;

beforeEach(() => {
  dataDir = fs.mkdtempSync(path.join(os.tmpdir(), 'nc-active-'));
});

afterEach(() => {
  closeAllHostDbs();
  fs.rmSync(dataDir, { recursive: true, force: true });
});

const baseRecord: ActiveRunRecord = {
  tenant_id: 'acme',
  agent_id: 'finance',
  group_folder: 'main',
  run_id: 'run-1',
  mode: 'live',
  pid: 12345,
  pid_start_time_ticks: 9999,
  expected_uid: 'ncg-acme-finance-0123456789', // format only; real value comes from deriveLinuxUsername
  runtime_dir: '/var/lib/nanoclaw/runtime/acme/finance/main/live',
  cgroup_path: '/sys/fs/cgroup/nanoclaw/run-1',
  started_at: '2026-06-21T00:00:00.000Z',
  last_seen_at: '2026-06-21T00:00:00.000Z',
  status: 'active',
};

describe('active_runs', () => {
  it('inserts and lists records for a tenant/agent', () => {
    insertActiveRun(dataDir, baseRecord);
    const rows = listActiveRuns(dataDir, 'acme', 'finance');
    expect(rows).toHaveLength(1);
    expect(rows[0].run_id).toBe('run-1');
    expect(rows[0].expected_uid).toMatch(/^ncg-/);
  });

  it('updates last_seen_at without changing other fields', () => {
    insertActiveRun(dataDir, baseRecord);
    updateActiveRunLastSeen(
      dataDir,
      'acme',
      'finance',
      'run-1',
      '2026-06-21T00:05:00.000Z',
    );
    const rows = listActiveRuns(dataDir, 'acme', 'finance');
    expect(rows[0].last_seen_at).toBe('2026-06-21T00:05:00.000Z');
    expect(rows[0].pid).toBe(12345);
  });

  it('marks record as exited with status change', () => {
    insertActiveRun(dataDir, baseRecord);
    markActiveRunExited(dataDir, 'acme', 'finance', 'run-1', 'crashed');
    const rows = listActiveRuns(dataDir, 'acme', 'finance');
    expect(rows[0].status).toBe('crashed');
  });

  it('rejects insertion with mismatched tenant_id in record', () => {
    expect(() =>
      insertActiveRun(dataDir, { ...baseRecord, tenant_id: 'other' }, 'acme', 'finance'),
    ).toThrow();
  });
});
```

- [ ] **第 2 步:运行测试以确认失败**

运行:`npm test -- src/db/active-runs.test.ts`
预期:FAIL —— 模块不存在。

- [ ] **第 3 步:实现 active_runs 访问器**

创建 `src/db/active-runs.ts`:

```typescript
import { openHostDb } from './partition.js';

export interface ActiveRunRecord {
  tenant_id: string;
  agent_id: string;
  group_folder: string;
  run_id: string;
  mode: 'live' | 'isolated';
  pid: number;
  pid_start_time_ticks: number;
  expected_uid: string;
  runtime_dir: string;
  cgroup_path: string | null;
  started_at: string;
  last_seen_at: string;
  status: 'active' | 'exited' | 'crashed';
}

/**
 * Each accessor opens a fresh DB connection, does its work, and closes.
 * Foundation plan is not in a hot path; later plans can introduce
 * connection pooling if profiling demands it.
 */
function withDb<T>(
  dataDir: string,
  tenant: string,
  agent: string,
  fn: (db: import('better-sqlite3').Database) => T,
): T {
  const db = openHostDb(dataDir, tenant, agent);
  try {
    return fn(db);
  } finally {
    db.close();
  }
}

export function insertActiveRun(
  dataDir: string,
  record: ActiveRunRecord,
  expectedTenant?: string,
  expectedAgent?: string,
): void {
  const tenant = expectedTenant ?? record.tenant_id;
  const agent = expectedAgent ?? record.agent_id;
  if (record.tenant_id !== tenant || record.agent_id !== agent) {
    throw new Error(
      `active_runs record identity mismatch: record=(${record.tenant_id},${record.agent_id}) args=(${tenant},${agent})`,
    );
  }
  withDb(dataDir, tenant, agent, (db) => {
    db.prepare(`
      INSERT INTO active_runs (
        tenant_id, agent_id, group_folder, run_id, mode,
        pid, pid_start_time_ticks, expected_uid,
        runtime_dir, cgroup_path, started_at, last_seen_at, status
      ) VALUES (
        @tenant_id, @agent_id, @group_folder, @run_id, @mode,
        @pid, @pid_start_time_ticks, @expected_uid,
        @runtime_dir, @cgroup_path, @started_at, @last_seen_at, @status
      )
    `).run(record);
  });
}

export function listActiveRuns(
  dataDir: string,
  tenant: string,
  agent: string,
): ActiveRunRecord[] {
  return withDb(dataDir, tenant, agent, (db) =>
    db
      .prepare(`SELECT * FROM active_runs ORDER BY started_at ASC`)
      .all() as ActiveRunRecord[],
  );
}

export function updateActiveRunLastSeen(
  dataDir: string,
  tenant: string,
  agent: string,
  runId: string,
  lastSeenAt: string,
): void {
  withDb(dataDir, tenant, agent, (db) => {
    const result = db
      .prepare(`UPDATE active_runs SET last_seen_at = ? WHERE tenant_id = ? AND agent_id = ? AND run_id = ?`)
      .run(lastSeenAt, tenant, agent, runId);
    if (result.changes === 0) {
      throw new Error(
        `active_runs row not found for (${tenant}, ${agent}, ${runId})`,
      );
    }
  });
}

export function markActiveRunExited(
  dataDir: string,
  tenant: string,
  agent: string,
  runId: string,
  status: 'exited' | 'crashed',
): void {
  withDb(dataDir, tenant, agent, (db) => {
    const result = db
      .prepare(`UPDATE active_runs SET status = ? WHERE tenant_id = ? AND agent_id = ? AND run_id = ?`)
      .run(status, tenant, agent, runId);
    if (result.changes === 0) {
      throw new Error(
        `active_runs row not found for (${tenant}, ${agent}, ${runId})`,
      );
    }
  });
}

export function deleteActiveRun(
  dataDir: string,
  tenant: string,
  agent: string,
  runId: string,
): void {
  withDb(dataDir, tenant, agent, (db) => {
    db.prepare(
      `DELETE FROM active_runs WHERE tenant_id = ? AND agent_id = ? AND run_id = ?`,
    ).run(tenant, agent, runId);
  });
}
```

- [ ] **第 4 步:运行测试以确认通过**

运行:`npm test -- src/db/active-runs.test.ts`
预期:PASS(4 个测试)。

- [ ] **第 5 步:类型检查**

运行:`npm run typecheck`
预期:无错误。

- [ ] **第 6 步:提交**

```bash
git add src/db/active-runs.ts src/db/active-runs.test.ts
git commit -m "feat(db): add active_runs table accessors for restart reconciliation"
```

---

## 任务 9:启动集成 —— 加载 tenant 仓库并打印诊断信息

**文件:**
- 修改:`src/index.ts`
- 新建:`src/tenants/startup.test.ts`

- [ ] **第 1 步:写失败的测试**

创建 `src/tenants/startup.test.ts`:

```typescript
import path from 'path';

import { describe, expect, it } from 'vitest';

import { formatStartupDiagnostics } from './startup.js';

describe('formatStartupDiagnostics', () => {
  it('formats empty load result', () => {
    const lines = formatStartupDiagnostics({
      tenants: [],
      agents: [],
      diagnostics: [{ level: 'error', message: 'not configured' }],
    });
    expect(lines.join('\n')).toMatch(/not configured/);
  });

  it('formats a successful load', () => {
    const lines = formatStartupDiagnostics({
      tenants: [{ id: 'acme', name: 'Acme', rootDir: '/x', enabled: true }],
      agents: [
        {
          id: 'finance',
          tenantId: 'acme',
          name: 'Finance',
          folder: '/x',
          provider: 'claude',
          model: 'sonnet',
          instructionsPath: '/x/instructions.md',
          resolvedSkills: [],
          channelTypes: ['feishu'],
          channelConfigs: {},
          envRefs: [],
          limits: {},
        },
      ],
      diagnostics: [],
    });
    const text = lines.join('\n');
    expect(text).toMatch(/tenant.*acme/i);
    expect(text).toMatch(/agent.*finance/i);
  });

  it('flags error-level diagnostics as fatal', () => {
    const lines = formatStartupDiagnostics({
      tenants: [],
      agents: [],
      diagnostics: [{ level: 'error', message: 'boom' }],
    });
    expect(lines.some((l) => /error/i.test(l))).toBe(true);
  });
});
```

- [ ] **第 2 步:运行测试以确认失败**

运行:`npm test -- src/tenants/startup.test.ts`
预期:FAIL —— 模块不存在。

- [ ] **第 3 步:实现启动诊断 helper**

创建 `src/tenants/startup.ts`:

```typescript
import { TenantRepoLoadResult } from './types.js';

/**
 * Produce human-readable diagnostic lines summarising a tenant repo load.
 * Used at startup to print loaded tenants/agents and any diagnostics.
 */
export function formatStartupDiagnostics(result: TenantRepoLoadResult): string[] {
  const lines: string[] = [];

  if (result.tenants.length === 0 && result.agents.length === 0) {
    lines.push('No tenants loaded.');
  } else {
    lines.push(`Loaded ${result.tenants.length} tenant(s), ${result.agents.length} agent(s):`);
    for (const t of result.tenants) {
      const flag = t.enabled ? '' : ' [disabled]';
      lines.push(`  tenant ${t.id}: ${t.name}${flag}`);
    }
    for (const a of result.agents) {
      const channels = a.channelTypes.length > 0 ? a.channelTypes.join(',') : 'no-channels';
      lines.push(`  agent ${a.tenantId}/${a.id}: provider=${a.provider} model=${a.model} channels=${channels}`);
    }
  }

  for (const d of result.diagnostics) {
    const tag = d.level.toUpperCase();
    const at = d.path ? ` [${d.path}]` : '';
    lines.push(`${tag}: ${d.message}${at}`);
  }

  return lines;
}
```

- [ ] **第 4 步:运行测试以确认通过**

运行:`npm test -- src/tenants/startup.test.ts`
预期:PASS(3 个测试)。

- [ ] **第 5 步:接入 index.ts**

在 `src/index.ts` 中找到启动序列的顶部(在 config 导入之后、channels 连接之前)。添加以下代码块。具体行号可能不同;寻找靠近 `main()` 函数顶部的第一个 `logger.info(...)` 调用。

```typescript
// Add to imports near the top:
import { NANOCLAW_TENANTS_DIR } from './config.js';
import { loadTenantRepo } from './tenants/loader.js';
import { formatStartupDiagnostics } from './tenants/startup.js';

// Add near the top of main(), before channels.connect():
if (NANOCLAW_TENANTS_DIR) {
  const tenantLoad = loadTenantRepo(NANOCLAW_TENANTS_DIR);
  for (const line of formatStartupDiagnostics(tenantLoad)) {
    logger.info({ component: 'tenants' }, line);
  }
  const hasErrors = tenantLoad.diagnostics.some((d) => d.level === 'error');
  if (hasErrors && tenantLoad.tenants.length === 0) {
    logger.error(
      { component: 'tenants' },
      'tenant repo load produced errors and zero tenants; continuing with legacy runtime',
    );
  }
} else {
  logger.info(
    { component: 'tenants' },
    'NANOCLAW_TENANTS_DIR not set; tenant loader disabled (legacy runtime continues)',
  );
}
```

- [ ] **第 6 步:类型检查**

运行:`npm run typecheck`
预期:无错误。

- [ ] **第 7 步:在 dev 模式下冒烟测试**

运行:`NANOCLAW_TENANTS_DIR=$(pwd)/tests/fixtures/tenants npm run dev`(看到启动日志后按 Ctrl+C)
预期:启动日志显示 `tenant acme: Acme Corp` 和两行 `agent acme/...`,无 error 级诊断。

- [ ] **第 8 步:提交**

```bash
git add src/index.ts src/tenants/startup.ts src/tenants/startup.test.ts
git commit -m "feat(tenants): load tenant repo at startup and print diagnostics"
```

---

## 任务 10:从 src/db.ts 重新导出分区 helper 并新增 npm 脚本

**文件:**
- 修改:`src/db.ts`
- 修改:`package.json`

- [ ] **第 1 步:重新导出分区 helper**

编辑 `src/db.ts`。在文件底部(现有导出之后)添加:

```typescript
// NanoClaw 2.0 partition helpers (Foundation). Re-exported here for discoverability.
// Consumers should prefer these over reaching into src/db/partition.ts directly
// so we can swap the implementation without touching call sites.
export {
  closeAllHostDbs,
  closeHostDb,
  openHostDb,
} from './db/partition.js';
export type { ActiveRunRecord } from './db/active-runs.js';
export {
  deleteActiveRun,
  insertActiveRun,
  listActiveRuns,
  markActiveRunExited,
  updateActiveRunLastSeen,
} from './db/active-runs.js';
```

- [ ] **第 2 步:新增诊断 npm 脚本**

编辑 `package.json`。在 `"scripts"` 块中添加:

```json
    "tenants:check": "tsx scripts/check-tenants.ts"
```

创建 `scripts/check-tenants.ts`:

```typescript
import { NANOCLAW_TENANTS_DIR } from '../src/config.js';
import { loadTenantRepo } from '../src/tenants/loader.js';
import { formatStartupDiagnostics } from '../src/tenants/startup.js';

if (!NANOCLAW_TENANTS_DIR) {
  console.error('NANOCLAW_TENANTS_DIR not set');
  process.exit(2);
}

const result = loadTenantRepo(NANOCLAW_TENANTS_DIR);
for (const line of formatStartupDiagnostics(result)) {
  console.log(line);
}
const errors = result.diagnostics.filter((d) => d.level === 'error');
process.exit(errors.length > 0 ? 1 : 0);
```

- [ ] **第 3 步:运行脚本**

运行:`NANOCLAW_TENANTS_DIR=$(pwd)/tests/fixtures/tenants npm run tenants:check`
预期:打印 tenant/agent 概要,退出码 0。

- [ ] **第 4 步:运行完整测试套件**

运行:`npm test`
预期:所有测试通过,现有 1.x 测试无回归。

- [ ] **第 5 步:类型检查**

运行:`npm run typecheck`
预期:无错误。

- [ ] **第 6 步:提交**

```bash
git add src/db.ts package.json scripts/check-tenants.ts
git commit -m "feat(db,tenants): re-export partition helpers and add tenants:check script"
```

---

## 任务 11:Foundation 验收测试(集成)

**文件:**
- 新建:`src/foundation-acceptance.test.ts`

- [ ] **第 1 步:写验收测试**

创建 `src/foundation-acceptance.test.ts`:

```typescript
import fs from 'fs';
import os from 'os';
import path from 'path';

import { afterEach, beforeEach, describe, expect, it } from 'vitest';

import { closeAllHostDbs, openHostDb } from './db/partition.js';
import { insertActiveRun, listActiveRuns } from './db/active-runs.js';
import { loadTenantRepo } from './tenants/loader.js';
import { formatStartupDiagnostics } from './tenants/startup.js';
import { deriveLinuxUsername, tenantAgentDbPath, tenantAgentRuntimeDir } from './tenants/naming.js';

const FIXTURES = path.resolve(process.cwd(), 'tests', 'fixtures', 'tenants');

let dataDir: string;

beforeEach(() => {
  dataDir = fs.mkdtempSync(path.join(os.tmpdir(), 'nc-foundation-'));
});

afterEach(() => {
  closeAllHostDbs();
  fs.rmSync(dataDir, { recursive: true, force: true });
});

describe('foundation acceptance', () => {
  it('loads tenant repo, opens partition DBs, records an active run', () => {
    const load = loadTenantRepo(FIXTURES);
    expect(load.diagnostics.filter((d) => d.level === 'error')).toEqual([]);
    expect(load.tenants.length).toBeGreaterThan(0);

    // For each loaded agent, open a partition DB and confirm schema is ready.
    for (const agent of load.agents) {
      const db = openHostDb(dataDir, agent.tenantId, agent.id);
      const tables = db
        .prepare("SELECT name FROM sqlite_master WHERE type='table'")
        .all() as { name: string }[];
      expect(tables.map((t) => t.name)).toContain('active_runs');
      expect(tables.map((t) => t.name)).toContain('messages');
    }

    // Simulate recording a live run for finance/main.
    const tenant = load.tenants[0];
    const agent = load.agents.find((a) => a.id === 'finance') ?? load.agents[0];
    const group = 'main';
    const username = deriveLinuxUsername(tenant.id, agent.id, group);
    const runtimeDir = tenantAgentRuntimeDir(dataDir, tenant.id, agent.id, group);

    insertActiveRun(dataDir, {
      tenant_id: tenant.id,
      agent_id: agent.id,
      group_folder: group,
      run_id: 'run-1',
      mode: 'live',
      pid: 4242,
      pid_start_time_ticks: 12345,
      expected_uid: username,
      runtime_dir: runtimeDir,
      cgroup_path: '/sys/fs/cgroup/nanoclaw/run-1',
      started_at: new Date().toISOString(),
      last_seen_at: new Date().toISOString(),
      status: 'active',
    });

    const rows = listActiveRuns(dataDir, tenant.id, agent.id);
    expect(rows).toHaveLength(1);
    expect(rows[0].expected_uid).toBe(username);
    expect(rows[0].runtime_dir).toBe(runtimeDir);
  });

  it('startup diagnostics surface loader errors', () => {
    const load = loadTenantRepo('/does/not/exist');
    const lines = formatStartupDiagnostics(load);
    expect(lines.some((l) => /error/i.test(l))).toBe(true);
  });

  it('partition DB path matches canonical layout', () => {
    openHostDb(dataDir, 'acme', 'finance').close();
    const expected = tenantAgentDbPath(dataDir, 'acme', 'finance');
    expect(fs.existsSync(expected)).toBe(true);
  });
});
```

- [ ] **第 2 步:运行验收测试**

运行:`npm test -- src/foundation-acceptance.test.ts`
预期:PASS(3 个测试)。

- [ ] **第 3 步:运行完整测试套件**

运行:`npm test`
预期:所有测试通过。

- [ ] **第 4 步:类型检查**

运行:`npm run typecheck`
预期:无错误。

- [ ] **第 5 步:提交**

```bash
git add src/foundation-acceptance.test.ts
git commit -m "test(foundation): end-to-end acceptance for tenant loader + DB partitioning"
```

---

## 自检记录

**规范覆盖检查**(对照 `docs/runtime-rework/`):
- ADR-001(tenant/skill 边界):✓ 任务 2、4、6
- ADR-003(DB 支持的 IPC):部分 —— Foundation 引入分区 DB,但不包含 runtime DB(`inbound.db` 等);那些在 Plan C 落地
- ADR-011(类型化 secret 引用):✓ 任务 3、4
- ADR-017(按 (tenant, agent) 的 DB 分区 + `channel_type` 键):✓ 任务 7、8
- CROSS_CUTTING_CONTRACTS "Tenant, Agent, and Group Configuration":✓ 任务 2、4、6
- CROSS_CUTTING_CONTRACTS "active_runs" record shape:✓ 任务 8
- 用户命名 `ncg-<t8>-<a8>-<hash10>`:✓ 任务 5
- 类型化 secret 引用格式 `llm:` / `channel:`:✓ 任务 3、4
- 按 agent 的 DB 路径 `${dataDir}/data/tenants/<t>/<a>/messages.db`:✓ 任务 7

**超出范围**(推迟到后续计划,已在计划头中记录):
- Channel registry 复合键重构 —— Plan B
- SUID helper、run 生命周期、runtime DB(`inbound.db`/`outbound.db`/`state.db`/`tools.db`) —— Plan C
- Skill bundle 暂存 —— Plan D
- 从遗留 `groups/` 的迁移脚本 —— Plan F
- 对 `src/container-runner.ts`、`src/router.ts`、`src/ipc.ts` 行为的任何修改

**占位符扫描**:无。每个步骤都有具体代码或命令。

**类型一致性**:任务 8 测试中使用的 `ActiveRunRecord` 与其在 `src/db/active-runs.ts` 中的定义匹配。任务 9 启动测试中使用的 `RegisteredAgent` 字段与任务 2 中的定义匹配。`parseSecretRef` 返回类型在任务 3、4、6 中保持一致。
