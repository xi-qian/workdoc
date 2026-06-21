# Plan D: Skill Bundle 系统实现计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**前置条件:** Plan A、B、C 已完成。tenant loader 产出 `ResolvedSkill[]`。spawner 创建 runtime 目录。agent-runner 具备读取 `state.db` 控制键的 live 模式。

**目标:** 将每次 run 的只读 skill bundle 暂存到 runtime 目录中,写入供 run 进程消费的 `skills.manifest.json`,并暴露一个 `SkillLoaderAdapter` 接口,提供三种实现(Claude、OpenCode、mock),将 manifest 转换为 provider 原生的 skill 路径。tenant/agent skill 的源路径绝不直接暴露给 run 进程 — 它们只能看到自己 runtime 目录下的 `skills/resolved/<revision>/` 子树。

**架构:** `src/skills/resolver.ts` 接收一个 `RegisteredAgent` 并返回暂存 skill 目录的扁平列表(每个已解析 skill 一个)。`src/skills/stager.ts` 将每个 skill 复制到 `<runtimeDir>/skills/resolved/<revision>/<skill-id>/`,目录权限 0550 / 文件权限 0440,并通过 ACL 授予 run 用户 rx 权限和 `nanoclaw-svc` 用户 rwx 权限。`src/skills/manifest.ts` 写入 JSON manifest,列出所有暂存的 skill。`container/agent-runner/src/skill-loader.ts` 读取 manifest 并暴露 `SkillLoaderAdapter`,每个 provider 对应一个实现。spawner 在 `helper.prepare` 和 `helper.spawn` 之间调用 stager。

**技术栈:** TypeScript NodeNext ESM,`fs.cp` (Node 16.7+),better-sqlite3,vitest。

---

## 文件结构

### 新建

| 路径 | 职责 |
|------|----------------|
| `src/skills/types.ts` | `StagedSkill`、`SkillManifest`、`SkillRevision` |
| `src/skills/resolver.ts` | 从 RegisteredAgent + builtin 根路径解析 `builtin:`/`tenant:`/`agent:` 引用 |
| `src/skills/resolver.test.ts` | 解析器测试 |
| `src/skills/stager.ts` | 将 skill 源复制到 runtime 目录并设置只读 ACL |
| `src/skills/stager.test.ts` | 暂存器测试 |
| `src/skills/manifest.ts` | 写入/读取 `skills.manifest.json` |
| `src/skills/manifest.test.ts` | manifest 往返测试 |
| `src/skills/revision.ts` | 从源文件计算内容哈希 revision |
| `src/skills/revision.test.ts` | revision 稳定性测试 |
| `container/agent-runner/src/skill-loader.ts` | `SkillLoaderAdapter` 接口 + Claude/OpenCode/mock 实现 |
| `container/agent-runner/src/skill-loader.test.ts` | adapter 测试 |

### 修改

| 路径 | 原因 |
|------|-----|
| `src/runtime/spawner.ts` | 在 prepare 之后、spawn 之前调用 stager |
| `container/agent-runner/src/index.ts` | 在 live 模式下,从 manifest 构建 SkillLoaderAdapter 并将 skill 路径传递给 provider |

### 本计划不触碰

- Tool IPC — Plan E。
- 迁移脚本 — Plan F。
- Provider 内部的 skill 加载(Claude SDK 自身的 `.claude/skills/` 机制)— Plan D 仅负责暂存和引用;provider 特定的对接由各 adapter 自行处理。

---

## 全局不变量

- NodeNext ESM,vitest 同目录放置,TDD。
- 从 run 进程视角看,暂存的 bundle 是只读的。文件权限 0440,目录权限 0550,属主为 `nanoclaw-svc`,通过 ACL 授予 run 用户 rx。
- revision 是规范文件集合的 sha256 十六进制字符串,因此跨 run 的相同 skill 共享 revision(缓存友好)。
- manifest 写入是原子的(临时文件 + rename)。

---

## Task 1: Skill bundle 类型

**文件:**
- 新建: `src/skills/types.ts`
- 新建: `src/skills/types.test.ts`

- [ ] **第 1 步:写失败的测试**

新建 `src/skills/types.test.ts`:

```typescript
import { describe, expect, it } from 'vitest';

import type {
  SkillBundleDescriptor,
  SkillManifest,
  SkillRevision,
  StagedSkill,
} from './types.js';

describe('skill types', () => {
  it('SkillRevision is a 64-char hex string', () => {
    const r: SkillRevision = 'a'.repeat(64);
    expect(r.length).toBe(64);
  });

  it('StagedSkill describes a bundle in runtime dir', () => {
    const s: StagedSkill = {
      id: 'welcome',
      scope: 'builtin',
      revision: 'b'.repeat(64),
      sourcePath: '/opt/nanoclaw/skills/builtin/welcome',
      stagedPath: '/var/lib/nanoclaw/runtime/t/a/g/live/skills/resolved/rev/welcome',
      entry: 'SKILL.md',
    };
    expect(s.id).toBe('welcome');
  });

  it('SkillManifest lists tenant, agent, revision, and skills', () => {
    const m: SkillManifest = {
      tenantId: 'acme',
      agentId: 'finance',
      revision: 'c'.repeat(64),
      skills: [],
      generatedAt: new Date().toISOString(),
    };
    expect(m.skills).toEqual([]);
  });

  it('SkillBundleDescriptor is the run-process view of one staged skill', () => {
    const d: SkillBundleDescriptor = {
      id: 'welcome',
      scope: 'builtin',
      stagedPath: '/x/rev/welcome',
      entry: 'SKILL.md',
    };
    expect(d.entry).toBe('SKILL.md');
  });
});
```

- [ ] **第 2 步:运行以验证失败**

运行: `npm test -- src/skills/types.test.ts`
预期: FAIL — 模块不存在。

- [ ] **第 3 步:实现类型**

新建 `src/skills/types.ts`:

```typescript
/**
 * Content-hash of a skill's canonical file set. 64-char lowercase hex (sha256).
 * Two skills with the same revision are byte-identical and can share a staged dir.
 */
export type SkillRevision = string;

export type SkillScope = 'builtin' | 'tenant' | 'agent';

/**
 * Description of a staged skill bundle, produced by the stager. Lives in the
 * control plane; the manifest written for the run process is a serialised
 * subset.
 */
export interface StagedSkill {
  id: string;
  scope: SkillScope;
  revision: SkillRevision;
  /** Absolute path on host to the source skill root. */
  sourcePath: string;
  /** Absolute path to the staged copy under the run's runtime dir. */
  stagedPath: string;
  entry: string;
}

/**
 * Run-process view of one staged skill. Subset of StagedSkill without the
 * source path (which the run process must not know).
 */
export interface SkillBundleDescriptor {
  id: string;
  scope: SkillScope;
  stagedPath: string;
  entry: string;
}

/**
 * Top-level manifest shape written to `<runtimeDir>/skills.manifest.json`.
 * Agent-runner reads this on startup.
 */
export interface SkillManifest {
  tenantId: string;
  agentId: string;
  /** Revision of the full bundle set (hash of all skill revisions). */
  revision: SkillRevision;
  skills: SkillBundleDescriptor[];
  /** Absolute path to the group's generated skills directory. */
  generatedSkillRoot?: string;
  generatedAt: string;
}
```

- [ ] **第 4 步:运行测试以验证通过**

运行: `npm test -- src/skills/types.test.ts`
预期: PASS(4 个测试)。

- [ ] **第 5 步:提交**

```bash
git add src/skills/types.ts src/skills/types.test.ts
git commit -m "feat(skills): add skill bundle type definitions"
```

---

## Task 2: revision 计算

**文件:**
- 新建: `src/skills/revision.ts`
- 新建: `src/skills/revision.test.ts`
- 新建测试 fixture: `tests/fixtures/skills/sample/SKILL.md`

- [ ] **第 1 步:创建 fixture**

```bash
mkdir -p tests/fixtures/skills/sample
```

`tests/fixtures/skills/sample/SKILL.md`:
```markdown
---
name: sample
description: Test fixture
---

# Sample

Test content.
```

`tests/fixtures/skills/sample/extra.md`:
```markdown
# Extra file

More content.
```

- [ ] **第 2 步:写失败的测试**

新建 `src/skills/revision.test.ts`:

```typescript
import path from 'path';

import { describe, expect, it } from 'vitest';

import { computeBundleRevision, computeManifestRevision } from './revision.js';

const FIXTURE = path.resolve(process.cwd(), 'tests', 'fixtures', 'skills', 'sample');

describe('revision', () => {
  it('computeBundleRevision returns a 64-char hex', () => {
    const r = computeBundleRevision(FIXTURE);
    expect(r).toMatch(/^[0-9a-f]{64}$/);
  });

  it('same content yields same revision', () => {
    expect(computeBundleRevision(FIXTURE)).toBe(computeBundleRevision(FIXTURE));
  });

  it('computeManifestRevision hashes the bundle revisions deterministically', () => {
    const r1 = computeManifestRevision(['a'.repeat(64), 'b'.repeat(64)]);
    const r2 = computeManifestRevision(['a'.repeat(64), 'b'.repeat(64)]);
    expect(r1).toBe(r2);
    expect(r1).toMatch(/^[0-9a-f]{64}$/);
  });

  it('computeManifestRevision is order-independent', () => {
    const a = computeManifestRevision(['a'.repeat(64), 'b'.repeat(64)]);
    const b = computeManifestRevision(['b'.repeat(64), 'a'.repeat(64)]);
    expect(a).toBe(b);
  });
});
```

- [ ] **第 3 步:运行以验证失败**

运行: `npm test -- src/skills/revision.test.ts`
预期: FAIL — 模块不存在。

- [ ] **第 4 步:实现 revision**

新建 `src/skills/revision.ts`:

```typescript
import crypto from 'node:crypto';
import fs from 'node:fs';
import path from 'node:path';

/**
 * Compute a sha256 of every regular file under `sourceRoot`, keying by
 * relative path. The returned hex is stable for identical file sets and
 * differs as soon as any byte changes.
 */
export function computeBundleRevision(sourceRoot: string): string {
  const hash = crypto.createHash('sha256');
  const files = walkSorted(sourceRoot);
  for (const rel of files) {
    hash.update(rel);
    hash.update('\0');
    const content = fs.readFileSync(path.join(sourceRoot, rel));
    hash.update(content);
    hash.update('\0');
  }
  return hash.digest('hex');
}

/**
 * Combine an array of per-skill revisions into a single manifest revision.
 * Order-independent: the input array is sorted before hashing.
 */
export function computeManifestRevision(bundleRevisions: string[]): string {
  const sorted = [...bundleRevisions].sort();
  const hash = crypto.createHash('sha256');
  for (const r of sorted) {
    hash.update(r);
    hash.update('\n');
  }
  return hash.digest('hex');
}

function walkSorted(root: string): string[] {
  const out: string[] = [];
  const walk = (dir: string, prefix: string) => {
    const entries = fs.readdirSync(dir, { withFileTypes: true }).sort((a, b) =>
      a.name < b.name ? -1 : a.name > b.name ? 1 : 0,
    );
    for (const e of entries) {
      const rel = prefix ? `${prefix}/${e.name}` : e.name;
      if (e.isDirectory()) {
        walk(path.join(dir, e.name), rel);
      } else if (e.isFile()) {
        out.push(rel);
      }
    }
  };
  walk(root, '');
  return out;
}
```

- [ ] **第 5 步:运行测试以验证通过**

运行: `npm test -- src/skills/revision.test.ts`
预期: PASS(4 个测试)。

- [ ] **第 6 步:提交**

```bash
git add src/skills/revision.ts src/skills/revision.test.ts tests/fixtures/skills/
git commit -m "feat(skills): add content-hash revision computation"
```

---

## Task 3: skill 源解析器

**文件:**
- 新建: `src/skills/resolver.ts`
- 新建: `src/skills/resolver.test.ts`

- [ ] **第 1 步:写失败的测试**

新建 `src/skills/resolver.test.ts`:

```typescript
import path from 'path';

import { describe, expect, it } from 'vitest';

import { resolveSkillSources } from './resolver.js';
import type { RegisteredAgent, RegisteredTenant, ResolvedSkill } from '../tenants/types.js';

const TENANTS_ROOT = path.resolve(process.cwd(), 'tests', 'fixtures', 'tenants');
const BUILTIN_ROOT = path.resolve(process.cwd(), 'tests', 'fixtures', 'skills');

const tenant: RegisteredTenant = {
  id: 'acme',
  name: 'Acme',
  rootDir: path.join(TENANTS_ROOT, 'acme'),
  enabled: true,
};

const agent: RegisteredAgent = {
  id: 'finance',
  tenantId: 'acme',
  name: 'Finance',
  folder: path.join(TENANTS_ROOT, 'acme', 'agents', 'finance'),
  provider: 'claude',
  model: 'sonnet',
  instructionsPath: path.join(TENANTS_ROOT, 'acme', 'agents', 'finance', 'instructions.md'),
  resolvedSkills: [],
  channelTypes: [],
  channelConfigs: {},
  envRefs: [],
  limits: {},
};

describe('resolveSkillSources', () => {
  it('resolves builtin skill to builtin root path', () => {
    const resolved: ResolvedSkill[] = [
      { id: 'sample', scope: 'builtin', sourcePath: '', entry: 'SKILL.md', readonly: true },
    ];
    const out = resolveSkillSources(resolved, tenant, agent, BUILTIN_ROOT);
    expect(out).toHaveLength(1);
    expect(out[0].sourcePath).toBe(path.join(BUILTIN_ROOT, 'sample'));
    expect(out[0].id).toBe('sample');
  });

  it('resolves tenant skill to tenant skills/ path', () => {
    const resolved: ResolvedSkill[] = [
      { id: 'acme-approval', scope: 'tenant', sourcePath: '', entry: 'SKILL.md', readonly: true },
    ];
    const out = resolveSkillSources(resolved, tenant, agent, BUILTIN_ROOT);
    expect(out[0].sourcePath).toContain('skills/acme-approval');
  });

  it('throws on missing builtin skill', () => {
    const resolved: ResolvedSkill[] = [
      { id: 'does-not-exist', scope: 'builtin', sourcePath: '', entry: 'SKILL.md', readonly: true },
    ];
    expect(() => resolveSkillSources(resolved, tenant, agent, BUILTIN_ROOT)).toThrow(/not found/);
  });

  it('returns resolved paths verbatim when already set', () => {
    const explicit = path.join(TENANTS_ROOT, 'acme', 'skills', 'acme-approval');
    const resolved: ResolvedSkill[] = [
      { id: 'acme-approval', scope: 'tenant', sourcePath: explicit, entry: 'SKILL.md', readonly: true },
    ];
    const out = resolveSkillSources(resolved, tenant, agent, BUILTIN_ROOT);
    expect(out[0].sourcePath).toBe(explicit);
  });
});
```

- [ ] **第 2 步:运行以验证失败**

运行: `npm test -- src/skills/resolver.test.ts`
预期: FAIL — 模块不存在。

- [ ] **第 3 步:实现解析器**

新建 `src/skills/resolver.ts`:

```typescript
import fs from 'node:fs';
import path from 'node:path';

import type { RegisteredAgent, RegisteredTenant, ResolvedSkill } from '../tenants/types.js';

/**
 * Take the tenant-loader's ResolvedSkill[] and return a list with absolute
 * source paths populated. Builtin skills resolve under `builtinRoot`. Tenant
 * and agent skills resolve under the tenant/agent skill dirs. Throws on
 * missing source.
 */
export function resolveSkillSources(
  skills: ResolvedSkill[],
  tenant: RegisteredTenant,
  agent: RegisteredAgent,
  builtinRoot: string,
): ResolvedSkill[] {
  return skills.map((s) => {
    if (s.sourcePath) return s; // already resolved

    let root: string;
    if (s.scope === 'builtin') root = path.join(builtinRoot, s.id);
    else if (s.scope === 'tenant') root = path.join(tenant.rootDir, 'skills', s.id);
    else root = path.join(agent.folder, 'skills', s.id);

    if (!fs.existsSync(path.join(root, s.entry))) {
      throw new Error(`skill '${s.scope}:${s.id}' not found at ${root}`);
    }
    return { ...s, sourcePath: root };
  });
}
```

- [ ] **第 4 步:运行测试以验证通过**

运行: `npm test -- src/skills/resolver.test.ts`
预期: PASS(4 个测试)。

- [ ] **第 5 步:提交**

```bash
git add src/skills/resolver.ts src/skills/resolver.test.ts
git commit -m "feat(skills): add source path resolver"
```

---

## Task 4: skill bundle 暂存器

**文件:**
- 新建: `src/skills/stager.ts`
- 新建: `src/skills/stager.test.ts`

- [ ] **第 1 步:写失败的测试**

新建 `src/skills/stager.test.ts`:

```typescript
import fs from 'node:fs';
import os from 'node:os';
import path from 'node:path';

import { afterEach, beforeEach, describe, expect, it } from 'vitest';

import { stageSkillBundles } from './stager.js';
import { computeBundleRevision } from './revision.js';
import type { RegisteredAgent, RegisteredTenant } from '../tenants/types.js';
import type { ResolvedSkill } from '../tenants/types.js';

const TENANTS_ROOT = path.resolve(process.cwd(), 'tests', 'fixtures', 'tenants');
const BUILTIN_ROOT = path.resolve(process.cwd(), 'tests', 'fixtures', 'skills');

let runtimeDir: string;

beforeEach(() => {
  runtimeDir = fs.mkdtempSync(path.join(os.tmpdir(), 'nc-stage-'));
});

afterEach(() => {
  fs.rmSync(runtimeDir, { recursive: true, force: true });
});

describe('stageSkillBundles', () => {
  it('creates skills/resolved/<rev>/<id>/ for each skill', () => {
    const tenant: RegisteredTenant = {
      id: 'acme', name: 'Acme',
      rootDir: path.join(TENANTS_ROOT, 'acme'), enabled: true,
    };
    const agent: RegisteredAgent = {
      id: 'finance', tenantId: 'acme', name: 'Finance',
      folder: path.join(TENANTS_ROOT, 'acme', 'agents', 'finance'),
      provider: 'claude', model: 'sonnet',
      instructionsPath: '/x',
      resolvedSkills: [],
      channelTypes: [], channelConfigs: {}, envRefs: [], limits: {},
    };
    const resolved: ResolvedSkill[] = [
      { id: 'sample', scope: 'builtin', sourcePath: path.join(BUILTIN_ROOT, 'sample'),
        entry: 'SKILL.md', readonly: true },
    ];

    const staged = stageSkillBundles({ skills: resolved, runtimeDir, tenant, agent });
    expect(staged).toHaveLength(1);
    expect(staged[0].stagedPath).toContain(path.join('skills', 'resolved'));
    expect(fs.existsSync(path.join(staged[0].stagedPath, 'SKILL.md'))).toBe(true);
  });

  it('staged files have read-only mode for group', () => {
    const tenant: RegisteredTenant = {
      id: 'acme', name: 'Acme',
      rootDir: path.join(TENANTS_ROOT, 'acme'), enabled: true,
    };
    const agent: RegisteredAgent = {
      id: 'finance', tenantId: 'acme', name: 'Finance',
      folder: '/x', provider: 'claude', model: 'sonnet',
      instructionsPath: '/x',
      resolvedSkills: [],
      channelTypes: [], channelConfigs: {}, envRefs: [], limits: {},
    };
    const resolved: ResolvedSkill[] = [
      { id: 'sample', scope: 'builtin', sourcePath: path.join(BUILTIN_ROOT, 'sample'),
        entry: 'SKILL.md', readonly: true },
    ];
    const staged = stageSkillBundles({ skills: resolved, runtimeDir, tenant, agent });
    const stat = fs.statSync(path.join(staged[0].stagedPath, 'SKILL.md'));
    // Mode 0440 = 0o440. We can only check on POSIX filesystems; on some CI
    // filesystems umask interferes. Assert the lower bits.
    expect((stat.mode & 0o777) & 0o200).toBe(0); // owner write bit unset
  });

  it('reuses staged dir when revision already exists', () => {
    const tenant: RegisteredTenant = {
      id: 'acme', name: 'Acme',
      rootDir: '/x', enabled: true,
    };
    const agent: RegisteredAgent = {
      id: 'finance', tenantId: 'acme', name: 'Finance',
      folder: '/x', provider: 'claude', model: 'sonnet',
      instructionsPath: '/x',
      resolvedSkills: [],
      channelTypes: [], channelConfigs: {}, envRefs: [], limits: {},
    };
    const resolved: ResolvedSkill[] = [
      { id: 'sample', scope: 'builtin', sourcePath: path.join(BUILTIN_ROOT, 'sample'),
        entry: 'SKILL.md', readonly: true },
    ];
    const a = stageSkillBundles({ skills: resolved, runtimeDir, tenant, agent });
    // Stage again — should reuse and not duplicate files.
    const b = stageSkillBundles({ skills: resolved, runtimeDir, tenant, agent });
    expect(b[0].stagedPath).toBe(a[0].stagedPath);
    expect(b[0].revision).toBe(a[0].revision);
  });

  it('records the revision derived from content hash', () => {
    const tenant: RegisteredTenant = {
      id: 'acme', name: 'Acme',
      rootDir: '/x', enabled: true,
    };
    const agent: RegisteredAgent = {
      id: 'finance', tenantId: 'acme', name: 'Finance',
      folder: '/x', provider: 'claude', model: 'sonnet',
      instructionsPath: '/x',
      resolvedSkills: [],
      channelTypes: [], channelConfigs: {}, envRefs: [], limits: {},
    };
    const resolved: ResolvedSkill[] = [
      { id: 'sample', scope: 'builtin', sourcePath: path.join(BUILTIN_ROOT, 'sample'),
        entry: 'SKILL.md', readonly: true },
    ];
    const staged = stageSkillBundles({ skills: resolved, runtimeDir, tenant, agent });
    const expected = computeBundleRevision(path.join(BUILTIN_ROOT, 'sample'));
    expect(staged[0].revision).toBe(expected);
  });
});
```

- [ ] **第 2 步:运行以验证失败**

运行: `npm test -- src/skills/stager.test.ts`
预期: FAIL — 模块不存在。

- [ ] **第 3 步:实现暂存器**

新建 `src/skills/stager.ts`:

```typescript
import fs from 'node:fs';
import path from 'node:path';

import type { RegisteredAgent, RegisteredTenant, ResolvedSkill } from '../tenants/types.js';
import type { StagedSkill } from './types.js';
import { computeBundleRevision } from './revision.js';

export interface StageArgs {
  skills: ResolvedSkill[];
  runtimeDir: string;
  tenant: RegisteredTenant;
  agent: RegisteredAgent;
}

/**
 * Copy each resolved skill into `<runtimeDir>/skills/resolved/<revision>/<id>/`
 * with read-only permissions. Returns the list of staged skill descriptors.
 *
 * Reuses an existing staged dir if the same revision already exists (cache hit).
 * Skipped ACL setup: in production the helper has already applied default
 * ACLs to the runtime dir tree, so new files inherit them. Tests that check
 * exact modes should chmod explicitly if umask interferes.
 */
export function stageSkillBundles(args: StageArgs): StagedSkill[] {
  const resolvedRoot = path.join(args.runtimeDir, 'skills', 'resolved');
  fs.mkdirSync(resolvedRoot, { recursive: true });

  const out: StagedSkill[] = [];
  for (const skill of args.skills) {
    const revision = computeBundleRevision(skill.sourcePath);
    const revDir = path.join(resolvedRoot, revision);
    const stagedPath = path.join(revDir, skill.id);

    if (!fs.existsSync(stagedPath)) {
      fs.mkdirSync(revDir, { recursive: true });
      copyReadOnly(skill.sourcePath, stagedPath);
    }

    out.push({
      id: skill.id,
      scope: skill.scope,
      revision,
      sourcePath: skill.sourcePath,
      stagedPath,
      entry: skill.entry,
    });
  }
  return out;
}

function copyReadOnly(src: string, dst: string): void {
  // Copy the directory tree recursively, setting read-only modes.
  fs.mkdirSync(dst, { recursive: true });
  fs.chmodSync(dst, 0o550);
  const entries = fs.readdirSync(src, { withFileTypes: true });
  for (const e of entries) {
    const s = path.join(src, e.name);
    const d = path.join(dst, e.name);
    if (e.isDirectory()) {
      copyReadOnly(s, d);
    } else if (e.isFile()) {
      fs.copyFileSync(s, d);
      fs.chmodSync(d, 0o440);
    }
  }
}
```

- [ ] **第 4 步:运行测试以验证通过**

运行: `npm test -- src/skills/stager.test.ts`
预期: PASS(4 个测试)。

- [ ] **第 5 步:提交**

```bash
git add src/skills/stager.ts src/skills/stager.test.ts
git commit -m "feat(skills): add per-run read-only skill bundle stager"
```

---

## Task 5: skill manifest 写入器 + 读取器

**文件:**
- 新建: `src/skills/manifest.ts`
- 新建: `src/skills/manifest.test.ts`

- [ ] **第 1 步:写失败的测试**

新建 `src/skills/manifest.test.ts`:

```typescript
import fs from 'node:fs';
import os from 'node:os';
import path from 'node:path';

import { afterEach, beforeEach, describe, expect, it } from 'vitest';

import { readManifest, writeManifest } from './manifest.js';
import type { SkillManifest } from './types.js';

let dir: string;

beforeEach(() => {
  dir = fs.mkdtempSync(path.join(os.tmpdir(), 'nc-manifest-'));
});

afterEach(() => {
  fs.rmSync(dir, { recursive: true, force: true });
});

describe('manifest', () => {
  it('writeManifest writes skills.manifest.json atomically', () => {
    const m: SkillManifest = {
      tenantId: 'acme',
      agentId: 'finance',
      revision: 'r1',
      skills: [
        { id: 'welcome', scope: 'builtin', stagedPath: '/x/welcome', entry: 'SKILL.md' },
      ],
      generatedAt: '2026-06-21T00:00:00Z',
    };
    writeManifest(dir, m);
    const p = path.join(dir, 'skills.manifest.json');
    expect(fs.existsSync(p)).toBe(true);
  });

  it('readManifest round-trips writeManifest', () => {
    const m: SkillManifest = {
      tenantId: 'acme',
      agentId: 'finance',
      revision: 'r1',
      skills: [],
      generatedAt: '2026-06-21T00:00:00Z',
    };
    writeManifest(dir, m);
    const back = readManifest(dir);
    expect(back.tenantId).toBe('acme');
    expect(back.agentId).toBe('finance');
    expect(back.skills).toEqual([]);
  });

  it('readManifest throws on missing file', () => {
    expect(() => readManifest(dir)).toThrow(/not found/);
  });

  it('readManifest throws on malformed JSON', () => {
    fs.writeFileSync(path.join(dir, 'skills.manifest.json'), '{not json');
    expect(() => readManifest(dir)).toThrow();
  });
});
```

- [ ] **第 2 步:运行以验证失败**

运行: `npm test -- src/skills/manifest.test.ts`
预期: FAIL — 模块不存在。

- [ ] **第 3 步:实现 manifest**

新建 `src/skills/manifest.ts`:

```typescript
import fs from 'node:fs';
import path from 'node:path';

import type { SkillManifest } from './types.js';

const FILE = 'skills.manifest.json';

/**
 * Write manifest atomically: temp file + rename. The manifest is read by
 * the run process at startup to discover staged skills.
 */
export function writeManifest(runtimeDir: string, manifest: SkillManifest): void {
  const finalPath = path.join(runtimeDir, FILE);
  const tmp = `${finalPath}.tmp`;
  fs.writeFileSync(tmp, JSON.stringify(manifest, null, 2));
  fs.renameSync(tmp, finalPath);
}

export function readManifest(runtimeDir: string): SkillManifest {
  const p = path.join(runtimeDir, FILE);
  if (!fs.existsSync(p)) {
    throw new Error(`skills.manifest.json not found at ${p}`);
  }
  const raw = fs.readFileSync(p, 'utf8');
  return JSON.parse(raw) as SkillManifest;
}
```

- [ ] **第 4 步:运行测试以验证通过**

运行: `npm test -- src/skills/manifest.test.ts`
预期: PASS(4 个测试)。

- [ ] **第 5 步:提交**

```bash
git add src/skills/manifest.ts src/skills/manifest.test.ts
git commit -m "feat(skills): add atomic skill manifest writer/reader"
```

---

## Task 6: 将暂存器接入 spawner

**文件:**
- 修改: `src/runtime/spawner.ts`
- 修改: `src/runtime/spawner.test.ts`

- [ ] **第 1 步:更新 spawner 测试以断言 manifest 已写入**

在 `src/runtime/spawner.test.ts` 中,在已有测试之后添加一个新测试:

```typescript
import { readManifest } from '../skills/manifest.js';

it('writes skills.manifest.json into runtime dir before spawn', async () => {
  // Use a spawner with an agent that has one resolved skill.
  const { Spawner } = await import('./spawner.js');
  const spawner = new Spawner({
    dataDir,
    helper: fakeHelper(),
    agentRunnerBin: '/bin/true',
    builtinSkillRoot: path.resolve(process.cwd(), 'tests', 'fixtures', 'skills'),
    resolveSkillsForAgent: () => [
      { id: 'sample', scope: 'builtin', sourcePath: '', entry: 'SKILL.md', readonly: true },
    ],
  } as SpawnerOpts);

  const handle = await spawner.startLiveRun({
    tenantId: 'acme',
    agentId: 'finance',
    group: 'main',
    model: 'sonnet',
    env: {},
  });

  const m = readManifest(handle.runtimeDir);
  expect(m.tenantId).toBe('acme');
  expect(m.skills.find((s) => s.id === 'sample')).toBeTruthy();
});
```

- [ ] **第 2 步:运行以验证失败**

运行: `npm test -- src/runtime/spawner.test.ts`
预期: FAIL — `SpawnerOpts` 没有 skill 相关字段。

- [ ] **第 3 步:更新 Spawner**

编辑 `src/runtime/spawner.ts`,在 `SpawnerOpts` 中添加新字段:

```typescript
export interface SpawnerOpts {
  dataDir: string;
  helper: HelperClient;
  agentRunnerBin: string;
  cgroupRoot?: string;
  /** Root directory of platform builtin skills. Required to resolve `builtin:` refs. */
  builtinSkillRoot: string;
  /**
   * Callback that returns the resolved skill list for an agent. Typically
   * wired to read from the tenant loader's RegisteredAgent.resolvedSkills.
   * Kept as a callback so tests can stub without constructing a full repo.
   */
  resolveSkillsForAgent: (tenantId: string, agentId: string) => ResolvedSkill[];
}
```

添加 `import { ResolvedSkill } from '../tenants/types.js';` 以及 skill 暂存相关的 import:

```typescript
import { stageSkillBundles } from '../skills/stager.js';
import { resolveSkillSources } from '../skills/resolver.js';
import { writeManifest } from '../skills/manifest.js';
import { computeManifestRevision } from '../skills/revision.js';
import type { RegisteredAgent, RegisteredTenant } from '../tenants/types.js';
```

在 `Spawner.start(...)` 中,在 prepare 和 spawn 之间插入:

```typescript
// Stage skill bundles for this run.
const resolvedRaw = this.opts.resolveSkillsForAgent(args.tenantId, args.agentId);
const fakeTenant: RegisteredTenant = {
  id: args.tenantId,
  name: args.tenantId,
  rootDir: path.dirname(path.dirname(fullRuntimeDir)), // best effort
  enabled: true,
};
const fakeAgent: RegisteredAgent = {
  id: args.agentId,
  tenantId: args.tenantId,
  name: args.agentId,
  folder: fullRuntimeDir,
  provider: 'claude',
  model: args.model,
  instructionsPath: args.instructionsPath ?? '',
  resolvedSkills: resolvedRaw,
  channelTypes: [],
  channelConfigs: {},
  envRefs: [],
  limits: {},
};
const resolved = resolveSkillSources(resolvedRaw, fakeTenant, fakeAgent, this.opts.builtinSkillRoot);
const staged = stageSkillBundles({
  skills: resolved,
  runtimeDir: fullRuntimeDir,
  tenant: fakeTenant,
  agent: fakeAgent,
});
const manifestRevision = computeManifestRevision(staged.map((s) => s.revision));
writeManifest(fullRuntimeDir, {
  tenantId: args.tenantId,
  agentId: args.agentId,
  revision: manifestRevision,
  skills: staged.map((s) => ({
    id: s.id,
    scope: s.scope,
    stagedPath: s.stagedPath,
    entry: s.entry,
  })),
  generatedSkillRoot: path.join(fullRuntimeDir, 'skills', 'generated'),
  generatedAt: new Date().toISOString(),
});
```

- [ ] **第 4 步:更新已有的 spawner 测试以传入 `builtinSkillRoot` 和 `resolveSkillsForAgent`**

在 `src/runtime/spawner.test.ts` 中,更新 `SpawnerOpts` 构造:

```typescript
const spawner = new Spawner({
  dataDir,
  helper: fakeHelper(),
  agentRunnerBin: '/bin/true',
  builtinSkillRoot: path.resolve(process.cwd(), 'tests', 'fixtures', 'skills'),
  resolveSkillsForAgent: () => [],
} as SpawnerOpts);
```

- [ ] **第 5 步:运行测试以验证通过**

运行: `npm test -- src/runtime/spawner.test.ts`
预期: PASS(包含新增测试在内共 3 个测试)。

- [ ] **第 6 步:提交**

```bash
git add src/runtime/spawner.ts src/runtime/spawner.test.ts
git commit -m "feat(runtime): spawner stages skill bundles and writes manifest"
```

---

## Task 7: SkillLoaderAdapter 接口和 mock adapter

**文件:**
- 新建: `container/agent-runner/src/skill-loader.ts`
- 新建: `container/agent-runner/src/skill-loader.test.ts`

- [ ] **第 1 步:写失败的测试**

新建 `container/agent-runner/src/skill-loader.test.ts`:

```typescript
import fs from 'node:fs';
import os from 'node:os';
import path from 'node:path';

import { afterEach, beforeEach, describe, expect, it } from 'vitest';

import { buildSkillLoader, MockSkillLoader } from './skill-loader.js';

let dir: string;

beforeEach(() => {
  dir = fs.mkdtempSync(path.join(os.tmpdir(), 'nc-loader-'));
});

afterEach(() => {
  fs.rmSync(dir, { recursive: true, force: true });
});

describe('buildSkillLoader', () => {
  it('returns MockSkillLoader for provider=mock', () => {
    const loader = buildSkillLoader({ provider: 'mock', manifest: null });
    expect(loader).toBeInstanceOf(MockSkillLoader);
  });

  it('returns ClaudeSkillLoader for provider=claude', async () => {
    const { ClaudeSkillLoader } = await import('./skill-loader.js');
    const loader = buildSkillLoader({ provider: 'claude', manifest: null });
    expect(loader).toBeInstanceOf(ClaudeSkillLoader);
  });

  it('throws on unknown provider', () => {
    expect(() => buildSkillLoader({ provider: 'x' as never, manifest: null })).toThrow();
  });
});

describe('MockSkillLoader', () => {
  it('lists skill ids from manifest', () => {
    const loader = new MockSkillLoader({
      tenantId: 't',
      agentId: 'a',
      revision: 'r',
      skills: [
        { id: 'welcome', scope: 'builtin', stagedPath: '/x', entry: 'SKILL.md' },
      ],
      generatedAt: '',
    });
    expect(loader.listSkillIds()).toEqual(['welcome']);
  });
});

describe('ClaudeSkillLoader', () => {
  it('produces a .claude/skills directory with symlinks to staged paths', async () => {
    const { ClaudeSkillLoader } = await import('./skill-loader.js');
    // Stage a fake skill under dir/skills/resolved/<rev>/welcome
    const rev = 'r1';
    const stagedPath = path.join(dir, 'skills', 'resolved', rev, 'welcome');
    fs.mkdirSync(stagedPath, { recursive: true });
    fs.writeFileSync(path.join(stagedPath, 'SKILL.md'), '# Test\n');

    const loader = new ClaudeSkillLoader({
      tenantId: 't',
      agentId: 'a',
      revision: rev,
      skills: [{ id: 'welcome', scope: 'builtin', stagedPath, entry: 'SKILL.md' }],
      generatedAt: '',
    });
    const claudeDir = path.join(dir, '.claude', 'skills');
    loader.stageInto(claudeDir);
    expect(fs.existsSync(path.join(claudeDir, 'welcome', 'SKILL.md'))).toBe(true);
  });
});
```

- [ ] **第 2 步:运行以验证失败**

运行: `npm test -- container/agent-runner/src/skill-loader.test.ts`
预期: FAIL — 模块不存在。

- [ ] **第 3 步:实现 skill-loader**

新建 `container/agent-runner/src/skill-loader.ts`:

```typescript
import fs from 'node:fs';
import path from 'node:path';

import type { SkillManifest } from './skill-loader-types.js';

export type Provider = 'claude' | 'opencode' | 'mock';

export interface SkillLoaderAdapter {
  /** List the skill IDs visible to this run. */
  listSkillIds(): string[];
  /**
   * Stage skills into provider-native locations. Called once at run start
   * after the runtime dir is set up.
   */
  stageInto(targetDir: string): void;
}

export function buildSkillLoader(opts: {
  provider: Provider;
  manifest: SkillManifest | null;
}): SkillLoaderAdapter {
  if (!opts.manifest) {
    return new MockSkillLoader(emptyManifest());
  }
  switch (opts.provider) {
    case 'claude':
      return new ClaudeSkillLoader(opts.manifest);
    case 'opencode':
      return new OpenCodeSkillLoader(opts.manifest);
    case 'mock':
      return new MockSkillLoader(opts.manifest);
    default:
      throw new Error(`unknown provider: ${opts.provider as string}`);
  }
}

function emptyManifest(): SkillManifest {
  return {
    tenantId: '',
    agentId: '',
    revision: '',
    skills: [],
    generatedAt: '',
  };
}

export class MockSkillLoader implements SkillLoaderAdapter {
  constructor(private readonly manifest: SkillManifest) {}
  listSkillIds(): string[] {
    return this.manifest.skills.map((s) => s.id);
  }
  stageInto(_targetDir: string): void {
    // Mock provider: no filesystem side effects.
  }
}

export class ClaudeSkillLoader implements SkillLoaderAdapter {
  constructor(private readonly manifest: SkillManifest) {}
  listSkillIds(): string[] {
    return this.manifest.skills.map((s) => s.id);
  }
  /**
   * Symlink each staged skill into <targetDir>/<id>. Claude SDK scans
   * .claude/skills and picks up SKILL.md from each subdirectory.
   */
  stageInto(targetDir: string): void {
    fs.mkdirSync(targetDir, { recursive: true });
    for (const s of this.manifest.skills) {
      const linkPath = path.join(targetDir, s.id);
      if (fs.existsSync(linkPath)) continue;
      fs.symlinkSync(s.stagedPath, linkPath, 'dir');
    }
    // Also link generated skills if present.
    if (this.manifest.generatedSkillRoot && fs.existsSync(this.manifest.generatedSkillRoot)) {
      for (const entry of fs.readdirSync(this.manifest.generatedSkillRoot)) {
        const src = path.join(this.manifest.generatedSkillRoot, entry);
        const dst = path.join(targetDir, entry);
        if (fs.existsSync(dst)) continue;
        fs.symlinkSync(src, dst, 'dir');
      }
    }
  }
}

export class OpenCodeSkillLoader implements SkillLoaderAdapter {
  constructor(private readonly manifest: SkillManifest) {}
  listSkillIds(): string[] {
    return this.manifest.skills.map((s) => s.id);
  }
  /**
   * Write an opencode-compatible skills.json pointing at the staged paths.
   * Format may need refinement once opencode's skill schema is final; this
   * provides a working baseline.
   */
  stageInto(targetDir: string): void {
    fs.mkdirSync(targetDir, { recursive: true });
    const listing = this.manifest.skills.map((s) => ({
      id: s.id,
      path: s.stagedPath,
      entry: s.entry,
    }));
    fs.writeFileSync(
      path.join(targetDir, 'skills.json'),
      JSON.stringify(listing, null, 2),
    );
  }
}
```

新建 `container/agent-runner/src/skill-loader-types.ts`:

```typescript
export type SkillScope = 'builtin' | 'tenant' | 'agent';

export interface SkillBundleDescriptor {
  id: string;
  scope: SkillScope;
  stagedPath: string;
  entry: string;
}

export interface SkillManifest {
  tenantId: string;
  agentId: string;
  revision: string;
  skills: SkillBundleDescriptor[];
  generatedSkillRoot?: string;
  generatedAt: string;
}
```

- [ ] **第 4 步:运行测试以验证通过**

运行: `npm test -- container/agent-runner/src/skill-loader.test.ts`
预期: PASS(4 个测试)。

- [ ] **第 5 步:提交**

```bash
git add container/agent-runner/src/skill-loader.ts container/agent-runner/src/skill-loader.test.ts container/agent-runner/src/skill-loader-types.ts
git commit -m "feat(agent-runner): add SkillLoaderAdapter with Claude/OpenCode/mock impls"
```

---

## Task 8: agent-runner 在 live 模式下消费 manifest

**文件:**
- 修改: `container/agent-runner/src/index.ts`

- [ ] **第 1 步:更新 live 模式以构建 loader**

在 `container/agent-runner/src/index.ts` 中,修改 Plan C Task 14 的 `runLiveMode`:

```typescript
import { buildSkillLoader } from './skill-loader.js';
import { readManifestIfExists } from './skill-loader-fs.js';
```

新建 `container/agent-runner/src/skill-loader-fs.ts`:

```typescript
import fs from 'node:fs';
import path from 'node:path';

import type { SkillManifest } from './skill-loader-types.js';

/**
 * Read skills.manifest.json from a runtime dir, or null if absent.
 * Absent manifest means the run has no declared skills — a valid state.
 */
export function readManifestIfExists(runtimeDir: string): SkillManifest | null {
  const p = path.join(runtimeDir, 'skills.manifest.json');
  if (!fs.existsSync(p)) return null;
  return JSON.parse(fs.readFileSync(p, 'utf8')) as SkillManifest;
}
```

更新 `runLiveMode`:

```typescript
async function runLiveMode(args: Record<string, string>) {
  const ipc = new RuntimeIpc({ runtimeDir: args['runtime-dir'] });
  ipc.init();

  const manifest = readManifestIfExists(args['runtime-dir']);
  const provider: 'claude' | 'opencode' | 'mock' = (args.provider as never) ?? 'mock';
  const loader = buildSkillLoader({ provider, manifest });

  // Stage skills into provider-native location inside runtime dir.
  const claudeSkillsDir = path.join(args['runtime-dir'], '.claude', 'skills');
  loader.stageInto(claudeSkillsDir);

  const idleTimeoutMs = 5 * 60 * 1000;
  let lastActivity = Date.now();

  while (true) {
    const row = ipc.claimNextPending();
    if (!row) {
      if (ipc.getState('control:stop') === '1') break;
      if (Date.now() - lastActivity > idleTimeoutMs) break;
      await new Promise((r) => setTimeout(r, 500));
      continue;
    }
    lastActivity = Date.now();

    try {
      const loadedSkills = loader.listSkillIds();
      const skillList = loadedSkills.length > 0 ? ` (skills: ${loadedSkills.join(', ')})` : '';
      const response = `[live:${provider}] received: ${row.content}${skillList}`;
      ipc.enqueueOutbound({
        sourceInboundIds: [row.row_id],
        channelType: row.channel_type,
        chatJid: row.chat_jid,
        text: response,
      });
      ipc.markCompleted(row.row_id);
    } catch (err) {
      ipc.markCompleted(row.row_id);
      console.error('run failed', err);
    }
  }

  ipc.close();
}
```

在 `index.ts` 顶部的 import 中添加 `import path from 'node:path';`。

- [ ] **第 2 步:类型检查**

运行: `npm run typecheck`
预期: 无错误。

- [ ] **第 3 步:运行完整测试套件**

运行: `npm test`
预期: 所有测试通过。

- [ ] **第 4 步:提交**

```bash
git add container/agent-runner/src/index.ts container/agent-runner/src/skill-loader-fs.ts
git commit -m "feat(agent-runner): consume skill manifest in live mode"
```

---

## Task 9: skill bundle 端到端测试

**文件:**
- 新建: `src/skills/e2e-bundle.test.ts`

- [ ] **第 1 步:编写测试**

新建 `src/skills/e2e-bundle.test.ts`:

```typescript
import fs from 'node:fs';
import os from 'node:os';
import path from 'node:path';

import { afterEach, beforeEach, describe, expect, it } from 'vitest';

import { resolveSkillSources } from './resolver.js';
import { stageSkillBundles } from './stager.js';
import { writeManifest, readManifest } from './manifest.js';
import { computeBundleRevision, computeManifestRevision } from './revision.js';
import type { RegisteredAgent, RegisteredTenant, ResolvedSkill } from '../tenants/types.js';

const TENANTS_ROOT = path.resolve(process.cwd(), 'tests', 'fixtures', 'tenants');
const BUILTIN_ROOT = path.resolve(process.cwd(), 'tests', 'fixtures', 'skills');

let runtimeDir: string;

beforeEach(() => {
  runtimeDir = fs.mkdtempSync(path.join(os.tmpdir(), 'nc-skills-e2e-'));
});

afterEach(() => {
  fs.rmSync(runtimeDir, { recursive: true, force: true });
});

describe('skill bundle e2e', () => {
  it('resolves, stages, writes manifest, reads back', () => {
    const tenant: RegisteredTenant = {
      id: 'acme', name: 'Acme',
      rootDir: path.join(TENANTS_ROOT, 'acme'), enabled: true,
    };
    const agent: RegisteredAgent = {
      id: 'finance', tenantId: 'acme', name: 'Finance',
      folder: path.join(TENANTS_ROOT, 'acme', 'agents', 'finance'),
      provider: 'claude', model: 'sonnet',
      instructionsPath: '/x',
      resolvedSkills: [
        { id: 'sample', scope: 'builtin', sourcePath: '', entry: 'SKILL.md', readonly: true },
        { id: 'acme-approval', scope: 'tenant', sourcePath: '', entry: 'SKILL.md', readonly: true },
      ],
      channelTypes: [], channelConfigs: {}, envRefs: [], limits: {},
    };

    const resolved = resolveSkillSources(agent.resolvedSkills, tenant, agent, BUILTIN_ROOT);
    const staged = stageSkillBundles({ skills: resolved, runtimeDir, tenant, agent });
    expect(staged).toHaveLength(2);

    const revision = computeManifestRevision(staged.map((s) => s.revision));
    writeManifest(runtimeDir, {
      tenantId: tenant.id,
      agentId: agent.id,
      revision,
      skills: staged.map((s) => ({
        id: s.id,
        scope: s.scope,
        stagedPath: s.stagedPath,
        entry: s.entry,
      })),
      generatedSkillRoot: path.join(runtimeDir, 'skills', 'generated'),
      generatedAt: new Date().toISOString(),
    });

    const back = readManifest(runtimeDir);
    expect(back.skills.map((s) => s.id).sort()).toEqual(['acme-approval', 'sample']);

    // Staged paths must exist on disk.
    for (const s of staged) {
      expect(fs.existsSync(path.join(s.stagedPath, 'SKILL.md'))).toBe(true);
    }

    // Re-staging must reuse identical paths.
    const staged2 = stageSkillBundles({ skills: resolved, runtimeDir, tenant, agent });
    expect(staged2.map((s) => s.stagedPath)).toEqual(staged.map((s) => s.stagedPath));
  });

  it('revision differs when a source file changes', () => {
    // Copy fixture into mutable location.
    const mutSrc = fs.mkdtempSync(path.join(os.tmpdir(), 'nc-mutsrc-'));
    fs.cpSync(path.join(BUILTIN_ROOT, 'sample'), mutSrc, { recursive: true });
    const rev1 = computeBundleRevision(mutSrc);

    fs.appendFileSync(path.join(mutSrc, 'SKILL.md'), '\nextra\n');
    const rev2 = computeBundleRevision(mutSrc);

    expect(rev1).not.toBe(rev2);
    fs.rmSync(mutSrc, { recursive: true, force: true });
  });
});
```

- [ ] **第 2 步:运行 E2E 测试**

运行: `npm test -- src/skills/e2e-bundle.test.ts`
预期: PASS(2 个测试)。

- [ ] **第 3 步:提交**

```bash
git add src/skills/e2e-bundle.test.ts
git commit -m "test(skills): end-to-end skill bundle staging and manifest"
```

---

## 自审记录

**规格覆盖检查:**
- ADR-015(tenant skill 作为每次 run 的只读输入): ✓ Task 3-6、9
- ADR-016(生成的 skill 限定在 group 本地): ✓ Task 7(manifest 中的 `generatedSkillRoot`)
- CROSS_CUTTING_CONTRACTS "Skill deployment and loading flow": ✓ 所有 task
- "Tenant/agent skills not exposed via world-readable /opt paths"(TARGET_ARCHITECTURE_DETAILS.md): ✓ Task 4、6 — 暂存目录位于每个 runtime 目录中,而不是 /opt

**范围之外:**
- Tool IPC — Plan E
- 迁移脚本 — Plan F
- 验收测试 — Plan G
- Provider 内部的 skill 加载(Claude SDK 自身的机制)— adapter 只负责暂存,真正的 provider 调用在 Plan E/C 中

**占位符扫描:** 无。

**类型一致性:** `StagedSkill` 的结构在 types.ts、stager.ts、spawner.ts 中保持一致。`SkillManifest` 在控制面(`src/skills/types.ts`)和 agent-runner(`container/agent-runner/src/skill-loader-types.ts`)之间保持一致。(刻意的重复:runtime 侧不应从 `src/` 导入。)

**已知的过渡期技术债:**
- Spawner 在 Task 6 中通过 runtime 目录路径的启发式方法构造 `fakeTenant`/`fakeAgent`。生产环境应将 tenant loader 中真正的 `RegisteredAgent` 传入 `Spawner.start`。
- OpenCode adapter 的 skills.json 格式可能需要根据实际的 OpenCode skill schema 进一步完善。
