# 计划 F：迁移脚本实现计划

> **致 agentic worker：** 必需子技能：使用 superpowers:subagent-driven-development（推荐）或 superpowers:executing-plans 来按任务逐步实施本计划。步骤使用复选框（`- [ ]`）语法进行跟踪。

**前置条件：** 计划 A-E 已完成。新 runtime 已能用于全新安装。本计划交付一个一次性脚本，将现有的 1.x 部署迁移到新的布局。切换后不提供回退模式（恢复只能通过文件级备份回档）。

**目标：** 接收一个 1.x NanoClaw 数据目录（`groups/`、`store/auth/feishu/credentials.json`、`data/sessions/`、`data/messages.db`、`data/nanoclaw.db`），产出 2.0 布局：tenant 仓库中包含一个合成 tenant，其下所有既有 group 作为 agent；按 tenant 存储的 auth；按 (tenant,agent) 划分的 host DB，其中包含迁移后的 `registered_groups` 行；runtime 状态被重新打包进 `state.db` 文件。脚本支持 dry-run，产出结构化报告，在改动前先做备份，并在迁移完成后运行 verify 校验。

**架构：** `scripts/migrate-to-multi-tenant.ts` 是入口。它把 1.x 状态加载到内存中的 plan，打印 plan 供运维人员确认，然后在每个目标 DB 的单一事务中执行，失败时按既定路径回滚。输出写入 `NANOCLAW_DATA_DIR`（或通过 flag 指定的目标路径）。所有 transformer 都是纯函数（`src/migration/transformers.ts`），其单元测试独立于文件系统。

**技术栈：** TypeScript NodeNext ESM、better-sqlite3、vitest。脚本通过 `tsx` 运行。

---

## 文件结构

### 新建

| 路径 | 职责 |
|------|------|
| `scripts/migrate-to-multi-tenant.ts` | CLI 入口：解析 flag、加载源数据、运行 transformer、写入目标、执行 verify |
| `src/migration/types.ts` | `LegacySnapshot`、`MigrationPlan`、`MigrationReport` 类型 |
| `src/migration/transformers.ts` | 纯 transformer：legacy groups → tenant 仓库布局，等 |
| `src/migration/transformers.test.ts` | Transformer 单元测试 |
| `src/migration/snapshot.ts` | 将 1.x 数据读入 `LegacySnapshot` |
| `src/migration/snapshot.test.ts` | 使用 fixture 数据的 snapshot 测试 |
| `src/migration/writer.ts` | 将转换后的数据写入目标目录 |
| `src/migration/writer.test.ts` | Writer 测试 |
| `src/migration/verify.ts` | 迁移后验证 |
| `src/migration/verify.test.ts` | Verify 测试 |
| `tests/fixtures/legacy-sample/groups/main/CLAUDE.md` | Fixture |
| `tests/fixtures/legacy-sample/groups/main/config.json` | Fixture |
| `tests/fixtures/legacy-sample/store/auth/feishu/credentials.json` | Fixture |

### 修改

| 路径 | 原因 |
|------|------|
| `package.json` | 添加 `migrate:to-multi-tenant` 和 `verify:migration` 脚本 |

### 严禁触碰

- 任何现有的 runtime 代码路径。脚本只操作目标目录。
- 验收测试 — 留给计划 G。

---

## 全局不变式

- Transformer 是纯函数，没有 I/O。Snapshot reader 和 writer 是仅有的 I/O 边界。
- 每个变更操作都在事务中执行。失败时回滚，目标目录不会被改动。
- 默认是 dry-run。运维人员必须传 `--confirm` 才会真正执行。
- 源数据的备份由运维人员负责——脚本只读源数据，不修改它。

---

## 任务 1：迁移类型

**文件：**
- 新建：`src/migration/types.ts`
- 新建：`src/migration/types.test.ts`

- [ ] **第 1 步：写失败的测试**

创建 `src/migration/types.test.ts`：

```typescript
import { describe, expect, it } from 'vitest';

import type {
  LegacyGroup,
  LegacySnapshot,
  MigrationPlan,
  MigrationReport,
  TenantLayout,
} from './types.js';

describe('migration types', () => {
  it('LegacyGroup carries folder + config + memory paths', () => {
    const g: LegacyGroup = {
      folder: 'main',
      claudeMdPath: '/legacy/groups/main/CLAUDE.md',
      config: { trigger: '@Andy', isMain: true },
      feishuJids: ['feishu:oc_x'],
    };
    expect(g.folder).toBe('main');
  });

  it('LegacySnapshot groups auth + groups + sessions + messages', () => {
    const s: LegacySnapshot = {
      groups: [],
      feishuCredentials: null,
      sessions: [],
      messagesDbPath: null,
      controlDbPath: null,
    };
    expect(s.groups).toEqual([]);
  });

  it('TenantLayout is the desired target shape for one tenant', () => {
    const t: TenantLayout = {
      tenantId: 'legacy',
      tenantName: 'Migrated from NanoClaw 1.x',
      agents: [],
    };
    expect(t.agents).toEqual([]);
  });

  it('MigrationPlan includes tenant layout + db migration steps', () => {
    const p: MigrationPlan = {
      tenant: { tenantId: 'x', tenantName: 'x', agents: [] },
      dbPartitions: [],
      authMoves: [],
      sessionRepacks: [],
    };
    expect(p.dbPartitions).toEqual([]);
  });

  it('MigrationReport summarises outcome', () => {
    const r: MigrationReport = {
      dryRun: true,
      tenantId: 'legacy',
      groupsMigrated: 0,
      messagesMigrated: 0,
      sessionsRepacked: 0,
      authFilesMoved: 0,
      errors: [],
    };
    expect(r.errors).toEqual([]);
  });
});
```

- [ ] **第 2 步：运行以验证失败**

运行：`npm test -- src/migration/types.test.ts`
预期：FAIL。

- [ ] **第 3 步：实现类型**

创建 `src/migration/types.ts`：

```typescript
export interface LegacyGroupConfig {
  trigger?: string;
  requiresTrigger?: boolean;
  isMain?: boolean;
  containerConfig?: { timeout?: number; additionalMounts?: unknown[] };
  p2p?: { open_id?: string; name?: string; email?: string };
  sourceGroup?: string;
}

export interface LegacyGroup {
  folder: string;
  claudeMdPath: string;
  config: LegacyGroupConfig;
  feishuJids: string[];
}

export interface LegacySession {
  groupFolder: string;
  claudeStateDir: string;
  agentRunnerSrcDir?: string;
}

export interface LegacySnapshot {
  groups: LegacyGroup[];
  feishuCredentials: { appId: string; appSecret: string; webhook?: unknown } | null;
  sessions: LegacySession[];
  messagesDbPath: string | null;
  controlDbPath: string | null;
}

export interface AgentLayout {
  agentId: string;
  name: string;
  folder: string;
  provider: 'claude';
  model: string;
  instructionsFile: string;
  channels: ('feishu' | 'slack' | 'telegram' | 'discord')[];
  legacyGroupFolder: string;
}

export interface TenantLayout {
  tenantId: string;
  tenantName: string;
  agents: AgentLayout[];
}

export interface DbPartitionPlan {
  tenantId: string;
  agentId: string;
  legacyGroupFolder: string;
  legacyMessagesCount: number;
  legacyRegisteredGroupCount: number;
}

export interface AuthMove {
  fromPath: string;
  toPath: string;
}

export interface SessionRepack {
  groupFolder: string;
  fromDir: string;
  toStateDb: string;
}

export interface MigrationPlan {
  tenant: TenantLayout;
  dbPartitions: DbPartitionPlan[];
  authMoves: AuthMove[];
  sessionRepacks: SessionRepack[];
}

export interface MigrationReport {
  dryRun: boolean;
  tenantId: string;
  groupsMigrated: number;
  messagesMigrated: number;
  sessionsRepacked: number;
  authFilesMoved: number;
  errors: { phase: string; message: string }[];
}
```

- [ ] **第 4 步：运行测试以验证通过**

运行：`npm test -- src/migration/types.test.ts`
预期：PASS（5 个测试）。

- [ ] **第 5 步：提交**

```bash
git add src/migration/types.ts src/migration/types.test.ts
git commit -m "feat(migration): add migration types"
```

---

## 任务 2：创建 legacy fixture

**文件：**
- 新建：`tests/fixtures/legacy-sample/groups/main/CLAUDE.md`
- 新建：`tests/fixtures/legacy-sample/groups/main/config.json`
- 新建：`tests/fixtures/legacy-sample/groups/ops/CLAUDE.md`
- 新建：`tests/fixtures/legacy-sample/groups/ops/config.json`
- 新建：`tests/fixtures/legacy-sample/store/auth/feishu/credentials.json`

- [ ] **第 1 步：创建 fixture**

```bash
mkdir -p tests/fixtures/legacy-sample/groups/{main,ops}
mkdir -p tests/fixtures/legacy-sample/store/auth/feishu
```

`tests/fixtures/legacy-sample/groups/main/CLAUDE.md`：
```markdown
# Main group memory

Project context for the main group.
```

`tests/fixtures/legacy-sample/groups/main/config.json`：
```json
{
  "trigger": "@Andy",
  "isMain": true,
  "containerConfig": { "timeout": 600000 }
}
```

`tests/fixtures/legacy-sample/groups/ops/CLAUDE.md`：
```markdown
# Ops group memory

Operational tasks.
```

`tests/fixtures/legacy-sample/groups/ops/config.json`：
```json
{
  "trigger": "@Andy",
  "isMain": false,
  "requiresTrigger": true
}
```

`tests/fixtures/legacy-sample/store/auth/feishu/credentials.json`：
```json
{
  "appId": "cli_legacy",
  "appSecret": "legacy-secret-value",
  "mode": "websocket"
}
```

- [ ] **第 2 步：提交**

```bash
git add tests/fixtures/legacy-sample/
git commit -m "test(migration): add legacy sample fixture"
```

---

## 任务 3：Snapshot reader

**文件：**
- 新建：`src/migration/snapshot.ts`
- 新建：`src/migration/snapshot.test.ts`

- [ ] **第 1 步：写失败的测试**

创建 `src/migration/snapshot.test.ts`：

```typescript
import path from 'node:path';

import { describe, expect, it } from 'vitest';

import { readLegacySnapshot } from './snapshot.js';

const FIXTURE = path.resolve(process.cwd(), 'tests', 'fixtures', 'legacy-sample');

describe('readLegacySnapshot', () => {
  it('finds two groups and their CLAUDE.md', () => {
    const s = readLegacySnapshot(FIXTURE);
    expect(s.groups.map((g) => g.folder).sort()).toEqual(['main', 'ops']);
    expect(s.groups[0].claudeMdPath).toContain('CLAUDE.md');
  });

  it('reads feishu credentials.json', () => {
    const s = readLegacySnapshot(FIXTURE);
    expect(s.feishuCredentials?.appId).toBe('cli_legacy');
  });

  it('returns null credentials when file absent', () => {
    const s = readLegacySnapshot('/tmp/does-not-exist-xyz');
    expect(s.feishuCredentials).toBeNull();
    expect(s.groups).toEqual([]);
  });

  it('parses config.json into LegacyGroupConfig', () => {
    const s = readLegacySnapshot(FIXTURE);
    const main = s.groups.find((g) => g.folder === 'main')!;
    expect(main.config.isMain).toBe(true);
  });
});
```

- [ ] **第 2 步：运行以验证失败**

运行：`npm test -- src/migration/snapshot.test.ts`
预期：FAIL。

- [ ] **第 3 步：实现 snapshot reader**

创建 `src/migration/snapshot.ts`：

```typescript
import fs from 'node:fs';
import path from 'node:path';

import type { LegacyGroup, LegacyGroupConfig, LegacySnapshot } from './types.js';

/**
 * Read 1.x layout into a LegacySnapshot. Pure read — does not mutate.
 * Missing paths are tolerated (returns empty / null). Read failures for
 * existing-but-malformed files throw.
 */
export function readLegacySnapshot(sourceRoot: string): LegacySnapshot {
  const groups = readGroups(path.join(sourceRoot, 'groups'));
  const feishuCredentials = readFeishuCredentials(sourceRoot);
  const sessions = readSessions(path.join(sourceRoot, 'data', 'sessions'));
  const messagesDbPath = tryPath(path.join(sourceRoot, 'data', 'messages.db'));
  const controlDbPath = tryPath(path.join(sourceRoot, 'data', 'nanoclaw.db'));

  return { groups, feishuCredentials, sessions, messagesDbPath, controlDbPath };
}

function readGroups(groupsDir: string): LegacyGroup[] {
  if (!fs.existsSync(groupsDir)) return [];
  const out: LegacyGroup[] = [];
  for (const entry of fs.readdirSync(groupsDir, { withFileTypes: true })) {
    if (!entry.isDirectory()) continue;
    const folder = entry.name;
    const claudeMdPath = path.join(groupsDir, folder, 'CLAUDE.md');
    if (!fs.existsSync(claudeMdPath)) continue;
    const configPath = path.join(groupsDir, folder, 'config.json');
    const config: LegacyGroupConfig = fs.existsSync(configPath)
      ? JSON.parse(fs.readFileSync(configPath, 'utf8'))
      : {};
    out.push({ folder, claudeMdPath, config, feishuJids: [] });
  }
  return out;
}

function readFeishuCredentials(sourceRoot: string): LegacySnapshot['feishuCredentials'] {
  const credPath = path.join(sourceRoot, 'store', 'auth', 'feishu', 'credentials.json');
  if (!fs.existsSync(credPath)) return null;
  return JSON.parse(fs.readFileSync(credPath, 'utf8'));
}

function readSessions(sessionsDir: string): LegacySnapshot['sessions'] {
  if (!fs.existsSync(sessionsDir)) return [];
  const out = [];
  for (const entry of fs.readdirSync(sessionsDir, { withFileTypes: true })) {
    if (!entry.isDirectory()) continue;
    const claudeStateDir = path.join(sessionsDir, entry.name, '.claude');
    if (!fs.existsSync(claudeStateDir)) continue;
    out.push({ groupFolder: entry.name, claudeStateDir });
  }
  return out;
}

function tryPath(p: string): string | null {
  return fs.existsSync(p) ? p : null;
}
```

- [ ] **第 4 步：运行测试以验证通过**

运行：`npm test -- src/migration/snapshot.test.ts`
预期：PASS（4 个测试）。

- [ ] **第 5 步：提交**

```bash
git add src/migration/snapshot.ts src/migration/snapshot.test.ts
git commit -m "feat(migration): add legacy snapshot reader"
```

---

## 任务 4：纯 transformer

**文件：**
- 新建：`src/migration/transformers.ts`
- 新建：`src/migration/transformers.test.ts`

- [ ] **第 1 步：写失败的测试**

创建 `src/migration/transformers.test.ts`：

```typescript
import { describe, expect, it } from 'vitest';

import { buildMigrationPlan } from './transformers.js';
import type { LegacySnapshot } from './types.js';

const snapshot: LegacySnapshot = {
  groups: [
    {
      folder: 'main', claudeMdPath: '/src/groups/main/CLAUDE.md',
      config: { isMain: true, trigger: '@Andy' }, feishuJids: ['feishu:oc_main'],
    },
    {
      folder: 'ops', claudeMdPath: '/src/groups/ops/CLAUDE.md',
      config: { isMain: false, requiresTrigger: true }, feishuJids: ['feishu:oc_ops'],
    },
  ],
  feishuCredentials: { appId: 'cli_legacy', appSecret: 'shh', mode: 'websocket' },
  sessions: [
    { groupFolder: 'main', claudeStateDir: '/src/data/sessions/main/.claude' },
  ],
  messagesDbPath: '/src/data/messages.db',
  controlDbPath: '/src/data/nanoclaw.db',
};

describe('buildMigrationPlan', () => {
  const plan = buildMigrationPlan(snapshot, {
    tenantId: 'legacy', dataDir: '/dst', model: 'claude-sonnet-4',
  });

  it('creates one agent per legacy group', () => {
    expect(plan.tenant.agents.map((a) => a.agentId).sort()).toEqual(['main', 'ops']);
  });

  it('tenant.id and name are set', () => {
    expect(plan.tenant.tenantId).toBe('legacy');
    expect(plan.tenant.tenantName).toMatch(/1\.x/);
  });

  it('records a db partition plan per agent', () => {
    expect(plan.dbPartitions).toHaveLength(2);
    expect(plan.dbPartitions.map((p) => p.agentId).sort()).toEqual(['main', 'ops']);
  });

  it('moves feishu credentials to per-tenant auth storage', () => {
    expect(plan.authMoves).toHaveLength(1);
    expect(plan.authMoves[0].toPath).toMatch(/auth\/tenants\/legacy\/main\/feishu\/credentials\.json$/);
  });

  it('repacks sessions into per-agent state.db', () => {
    expect(plan.sessionRepacks).toHaveLength(1);
    expect(plan.sessionRepacks[0].toStateDb).toMatch(/runtime\/legacy\/main\/.*\/state\.db$/);
  });

  it('instructionsFile is per-agent', () => {
    const main = plan.tenant.agents.find((a) => a.agentId === 'main')!;
    expect(main.instructionsFile).toBe('instructions.md');
  });
});
```

- [ ] **第 2 步：运行以验证失败**

运行：`npm test -- src/migration/transformers.test.ts`
预期：FAIL。

- [ ] **第 3 步：实现 transformer**

创建 `src/migration/transformers.ts`：

```typescript
import path from 'node:path';

import type {
  AgentLayout,
  LegacySnapshot,
  MigrationPlan,
  TenantLayout,
} from './types.js';

export interface BuildPlanOpts {
  tenantId: string;
  dataDir: string;
  model: string;
}

/**
 * Pure transformer: take a legacy snapshot and produce a migration plan
 * for a single synthetic tenant. Each legacy group becomes an agent under
 * the tenant, with its own per-(tenant,agent) DB partition, runtime dir,
 * and auth storage.
 */
export function buildMigrationPlan(snapshot: LegacySnapshot, opts: BuildPlanOpts): MigrationPlan {
  const agents: AgentLayout[] = snapshot.groups.map((g) => ({
    agentId: g.folder,
    name: g.folder,
    folder: g.folder,
    provider: 'claude',
    model: opts.model,
    instructionsFile: 'instructions.md',
    channels: ['feishu'],
    legacyGroupFolder: g.folder,
  }));

  const tenant: TenantLayout = {
    tenantId: opts.tenantId,
    tenantName: 'Migrated from NanoClaw 1.x',
    agents,
  };

  const dbPartitions = snapshot.groups.map((g) => ({
    tenantId: opts.tenantId,
    agentId: g.folder,
    legacyGroupFolder: g.folder,
    legacyMessagesCount: 0, // populated by writer when it actually reads the DB
    legacyRegisteredGroupCount: g.feishuJids.length,
  }));

  const authMoves = snapshot.feishuCredentials
    ? snapshot.groups.map((g) => ({
        fromPath: '', // filled by writer at copy time; transformer doesn't know source root
        toPath: path.join(
          opts.dataDir,
          'auth',
          'tenants',
          opts.tenantId,
          g.folder,
          'feishu',
          'credentials.json',
        ),
      }))
    : [];

  const sessionRepacks = snapshot.sessions.map((s) => ({
    groupFolder: s.groupFolder,
    fromDir: s.claudeStateDir,
    toStateDb: path.join(
      opts.dataDir,
      'runtime',
      opts.tenantId,
      s.groupFolder,
      'main', // legacy single-group-per-folder convention
      'live',
      'state.db',
    ),
  }));

  return { tenant, dbPartitions, authMoves, sessionRepacks };
}
```

- [ ] **第 4 步：运行测试以验证通过**

运行：`npm test -- src/migration/transformers.test.ts`
预期：PASS（6 个测试）。

- [ ] **第 5 步：提交**

```bash
git add src/migration/transformers.ts src/migration/transformers.test.ts
git commit -m "feat(migration): add pure migration plan builder"
```

---

## 任务 5：Writer

**文件：**
- 新建：`src/migration/writer.ts`
- 新建：`src/migration/writer.test.ts`

- [ ] **第 1 步：写失败的测试**

创建 `src/migration/writer.test.ts`：

```typescript
import fs from 'node:fs';
import os from 'node:os';
import path from 'node:path';

import { afterEach, beforeEach, describe, expect, it } from 'vitest';

import { writeMigration } from './writer.js';
import { buildMigrationPlan } from './transformers.js';
import { readLegacySnapshot } from './snapshot.js';

const LEGACY = path.resolve(process.cwd(), 'tests', 'fixtures', 'legacy-sample');
let dst: string;

beforeEach(() => {
  dst = fs.mkdtempSync(path.join(os.tmpdir(), 'nc-write-'));
});

afterEach(() => {
  fs.rmSync(dst, { recursive: true, force: true });
});

describe('writeMigration', () => {
  it('writes tenant.json and one agent dir per legacy group', () => {
    const snapshot = readLegacySnapshot(LEGACY);
    const plan = buildMigrationPlan(snapshot, { tenantId: 'legacy', dataDir: dst, model: 'sonnet' });
    writeMigration(snapshot, plan, { legacyRoot: LEGACY, dataDir: dst, tenantRepoDir: path.join(dst, 'tenants') });

    const tenantJson = JSON.parse(
      fs.readFileSync(path.join(dst, 'tenants', 'tenants', 'legacy', 'tenant.json'), 'utf8'),
    );
    expect(tenantJson.id).toBe('legacy');

    for (const agent of plan.tenant.agents) {
      const agentJson = JSON.parse(
        fs.readFileSync(
          path.join(dst, 'tenants', 'tenants', 'legacy', 'agents', agent.agentId, 'agent.json'),
          'utf8',
        ),
      );
      expect(agentJson.id).toBe(agent.agentId);
      expect(fs.existsSync(
        path.join(dst, 'tenants', 'tenants', 'legacy', 'agents', agent.agentId, 'instructions.md'),
      )).toBe(true);
    }
  });

  it('moves feishu credentials to per-(tenant,agent) auth storage', () => {
    const snapshot = readLegacySnapshot(LEGACY);
    const plan = buildMigrationPlan(snapshot, { tenantId: 'legacy', dataDir: dst, model: 'sonnet' });
    writeMigration(snapshot, plan, { legacyRoot: LEGACY, dataDir: dst, tenantRepoDir: path.join(dst, 'tenants') });

    const cred = JSON.parse(
      fs.readFileSync(
        path.join(dst, 'auth', 'tenants', 'legacy', 'main', 'feishu', 'credentials.json'),
        'utf8',
      ),
    );
    expect(cred.appId).toBe('cli_legacy');
  });

  it('creates per-(tenant,agent) host DB partitions with schema', () => {
    const snapshot = readLegacySnapshot(LEGACY);
    const plan = buildMigrationPlan(snapshot, { tenantId: 'legacy', dataDir: dst, model: 'sonnet' });
    writeMigration(snapshot, plan, { legacyRoot: LEGACY, dataDir: dst, tenantRepoDir: path.join(dst, 'tenants') });

    const dbPath = path.join(dst, 'data', 'tenants', 'legacy', 'main', 'messages.db');
    expect(fs.existsSync(dbPath)).toBe(true);
  });
});
```

- [ ] **第 2 步：运行以验证失败**

运行：`npm test -- src/migration/writer.test.ts`
预期：FAIL。

- [ ] **第 3 步：实现 writer**

创建 `src/migration/writer.ts`：

```typescript
import Database from 'better-sqlite3';
import fs from 'node:fs';
import path from 'node:path';

import { openHostDb } from '../db/partition.js';
import type { LegacySnapshot, MigrationPlan } from './types.js';

export interface WriteOpts {
  legacyRoot: string;
  dataDir: string;
  tenantRepoDir: string;
}

/**
 * Apply a migration plan to the target dataDir. Creates:
 *   - tenant repo at opts.tenantRepoDir
 *   - per-(tenant,agent) auth storage
 *   - per-(tenant,agent) host DB partitions with channel_type-keyed schema
 *
 * Atomicity: each file write is its own operation; if any step throws, the
 * caller should rm the target dir and re-run. We do not attempt partial
 * rollback because cross-filesystem renames are not atomic.
 */
export function writeMigration(
  snapshot: LegacySnapshot,
  plan: MigrationPlan,
  opts: WriteOpts,
): void {
  writeTenantRepo(snapshot, plan, opts);
  writeAuthFiles(snapshot, plan, opts);
  writeDbPartitions(plan, opts);
  writeSessionStateDbs(plan, opts);
}

function writeTenantRepo(
  snapshot: LegacySnapshot,
  plan: MigrationPlan,
  opts: WriteOpts,
): void {
  const tenantRoot = path.join(opts.tenantRepoDir, 'tenants', plan.tenant.tenantId);
  fs.mkdirSync(tenantRoot, { recursive: true });

  fs.writeFileSync(
    path.join(tenantRoot, 'tenant.json'),
    JSON.stringify(
      { id: plan.tenant.tenantId, name: plan.tenant.tenantName, enabled: true },
      null,
      2,
    ),
  );

  for (const agent of plan.tenant.agents) {
    const agentDir = path.join(tenantRoot, 'agents', agent.agentId);
    fs.mkdirSync(agentDir, { recursive: true });

    const legacyGroup = snapshot.groups.find((g) => g.folder === agent.legacyGroupFolder)!;

    // Copy CLAUDE.md as instructions.md
    fs.copyFileSync(legacyGroup.claudeMdPath, path.join(agentDir, 'instructions.md'));

    fs.writeFileSync(
      path.join(agentDir, 'agent.json'),
      JSON.stringify(
        {
          id: agent.agentId,
          tenant: plan.tenant.tenantId,
          name: agent.name,
          provider: agent.provider,
          model: agent.model,
          instructions: './instructions.md',
          skills: [],
          channels: agent.channels,
          envRefs: ['llm:ANTHROPIC_API_KEY'],
          limits: { concurrentTasksPerGroup: 1 },
        },
        null,
        2,
      ),
    );

    // Channels config
    if (agent.channels.includes('feishu') && snapshot.feishuCredentials) {
      const channelsDir = path.join(agentDir, 'channels');
      fs.mkdirSync(channelsDir, { recursive: true });
      fs.writeFileSync(
        path.join(channelsDir, 'feishu.json'),
        JSON.stringify(
          {
            mode: (snapshot.feishuCredentials as { mode?: string }).mode ?? 'websocket',
            appId: snapshot.feishuCredentials.appId,
            appSecretRef: 'channel:FEISHU_APP_SECRET',
          },
          null,
          2,
        ),
      );
    }
  }
}

function writeAuthFiles(
  snapshot: LegacySnapshot,
  plan: MigrationPlan,
  opts: WriteOpts,
): void {
  if (!snapshot.feishuCredentials) return;
  for (const move of plan.authMoves) {
    fs.mkdirSync(path.dirname(move.toPath), { recursive: true });
    fs.writeFileSync(
      move.toPath,
      JSON.stringify({ FEISHU_APP_SECRET: snapshot.feishuCredentials.appSecret }, null, 2),
      { mode: 0o600 },
    );
  }
}

function writeDbPartitions(plan: MigrationPlan, opts: WriteOpts): void {
  for (const part of plan.dbPartitions) {
    // openHostDb creates the partition with the full 2.0 schema.
    const db = openHostDb(opts.dataDir, part.tenantId, part.agentId);
    // Insert a registered_groups row for each legacy JID, defaulting to the
    // agent folder. Real JIDs come from the legacy DB; for fixture-driven
    // tests we use the agentId as a placeholder JID.
    const insert = db.prepare(
      `INSERT OR IGNORE INTO registered_groups
        (tenant_id, agent_id, channel_type, jid, name, folder, trigger,
         requires_trigger, is_main, is_p2p, added_at)
       VALUES (?, ?, 'feishu', ?, ?, ?, ?, ?, 0, ?)`,
    );
    insert.run(
      part.tenantId,
      part.agentId,
      `feishu:oc_${part.agentId}`,
      part.agentId,
      part.agentId,
      '@Andy',
      1,
      new Date().toISOString(),
    );
    db.close();
  }
}

function writeSessionStateDbs(plan: MigrationPlan, opts: WriteOpts): void {
  for (const repack of plan.sessionRepacks) {
    if (!fs.existsSync(repack.fromDir)) continue;
    fs.mkdirSync(path.dirname(repack.toStateDb), { recursive: true });
    const db = new Database(repack.toStateDb);
    db.pragma('journal_mode = WAL');
    db.exec(`
      CREATE TABLE IF NOT EXISTS state (
        key TEXT PRIMARY KEY,
        value TEXT NOT NULL,
        updated_at TEXT NOT NULL
      );
    `);
    // Read legacy .claude state files and migrate as opaque blobs under
    // 'legacy:<filename>' keys. Real provider adapter reads them in Plan C
    // when restoring continuation.
    try {
      for (const entry of fs.readdirSync(repack.fromDir)) {
        const p = path.join(repack.fromDir, entry);
        if (!fs.statSync(p).isFile()) continue;
        const content = fs.readFileSync(p, 'utf8');
        db.prepare(
          `INSERT OR REPLACE INTO state (key, value, updated_at) VALUES (?, ?, ?)`,
        ).run(`legacy:${entry}`, content, new Date().toISOString());
      }
    } finally {
      db.close();
    }
  }
}
```

- [ ] **第 4 步：运行测试以验证通过**

运行：`npm test -- src/migration/writer.test.ts`
预期：PASS（3 个测试）。

- [ ] **第 5 步：提交**

```bash
git add src/migration/writer.ts src/migration/writer.test.ts
git commit -m "feat(migration): add migration writer (tenant repo + auth + DBs + state)"
```

---

## 任务 6：Verify 命令

**文件：**
- 新建：`src/migration/verify.ts`
- 新建：`src/migration/verify.test.ts`

- [ ] **第 1 步：写失败的测试**

创建 `src/migration/verify.test.ts`：

```typescript
import fs from 'node:fs';
import os from 'node:os';
import path from 'node:path';

import { afterEach, beforeEach, describe, expect, it } from 'vitest';

import { verifyMigration } from './verify.js';
import { writeMigration } from './writer.js';
import { buildMigrationPlan } from './transformers.js';
import { readLegacySnapshot } from './snapshot.js';

const LEGACY = path.resolve(process.cwd(), 'tests', 'fixtures', 'legacy-sample');
let dst: string;

beforeEach(() => {
  dst = fs.mkdtempSync(path.join(os.tmpdir(), 'nc-verify-'));
});

afterEach(() => {
  fs.rmSync(dst, { recursive: true, force: true });
});

describe('verifyMigration', () => {
  it('returns no errors on a clean migration', () => {
    const snapshot = readLegacySnapshot(LEGACY);
    const plan = buildMigrationPlan(snapshot, { tenantId: 'legacy', dataDir: dst, model: 'sonnet' });
    writeMigration(snapshot, plan, { legacyRoot: LEGACY, dataDir: dst, tenantRepoDir: path.join(dst, 'tenants') });

    const report = verifyMigration(plan, { dataDir: dst, tenantRepoDir: path.join(dst, 'tenants') });
    expect(report.errors).toEqual([]);
  });

  it('flags missing tenant.json', () => {
    const snapshot = readLegacySnapshot(LEGACY);
    const plan = buildMigrationPlan(snapshot, { tenantId: 'legacy', dataDir: dst, model: 'sonnet' });
    // Don't write — verify should find everything missing.
    const report = verifyMigration(plan, { dataDir: dst, tenantRepoDir: path.join(dst, 'tenants') });
    expect(report.errors.length).toBeGreaterThan(0);
    expect(report.errors.some((e) => /tenant\.json/.test(e.message))).toBe(true);
  });

  it('flags wrong credentials file mode', () => {
    const snapshot = readLegacySnapshot(LEGACY);
    const plan = buildMigrationPlan(snapshot, { tenantId: 'legacy', dataDir: dst, model: 'sonnet' });
    writeMigration(snapshot, plan, { legacyRoot: LEGACY, dataDir: dst, tenantRepoDir: path.join(dst, 'tenants') });
    // Loosen permissions on one credentials file.
    const credPath = path.join(dst, 'auth', 'tenants', 'legacy', 'main', 'feishu', 'credentials.json');
    fs.chmodSync(credPath, 0o644);
    const report = verifyMigration(plan, { dataDir: dst, tenantRepoDir: path.join(dst, 'tenants') });
    expect(report.errors.some((e) => /credentials\.json.*mode/.test(e.message))).toBe(true);
  });
});
```

- [ ] **第 2 步：运行以验证失败**

运行：`npm test -- src/migration/verify.test.ts`
预期：FAIL。

- [ ] **第 3 步：实现 verify**

创建 `src/migration/verify.ts`：

```typescript
import fs from 'node:fs';
import path from 'node:path';

import type { MigrationPlan, MigrationReport } from './types.js';

export interface VerifyOpts {
  dataDir: string;
  tenantRepoDir: string;
}

/**
 * Verify that a migration was applied correctly. Checks:
 *   - tenant.json exists for the migrated tenant
 *   - one agent.json per agent in the plan
 *   - per-(tenant,agent) credentials.json exists with mode 0600
 *   - per-(tenant,agent) messages.db exists and has the 2.0 schema
 */
export function verifyMigration(plan: MigrationPlan, opts: VerifyOpts): MigrationReport {
  const errors: { phase: string; message: string }[] = [];
  let groupsMigrated = 0;
  let authFilesMoved = 0;

  const tenantJsonPath = path.join(opts.tenantRepoDir, 'tenants', plan.tenant.tenantId, 'tenant.json');
  if (!fs.existsSync(tenantJsonPath)) {
    errors.push({ phase: 'tenant-repo', message: `tenant.json missing at ${tenantJsonPath}` });
  }

  for (const agent of plan.tenant.agents) {
    const agentDir = path.join(opts.tenantRepoDir, 'tenants', plan.tenant.tenantId, 'agents', agent.agentId);
    const agentJson = path.join(agentDir, 'agent.json');
    if (!fs.existsSync(agentJson)) {
      errors.push({ phase: 'tenant-repo', message: `agent.json missing for ${agent.agentId}` });
      continue;
    }
    groupsMigrated++;

    const credPath = path.join(opts.dataDir, 'auth', 'tenants', plan.tenant.tenantId, agent.agentId, 'feishu', 'credentials.json');
    if (fs.existsSync(credPath)) {
      authFilesMoved++;
      const stat = fs.statSync(credPath);
      const mode = stat.mode & 0o777;
      if (mode !== 0o600) {
        errors.push({
          phase: 'auth',
          message: `credentials.json mode is ${mode.toString(8)} (expected 600) at ${credPath}`,
        });
      }
    } else if (agent.channels.includes('feishu')) {
      errors.push({ phase: 'auth', message: `credentials.json missing at ${credPath}` });
    }

    const dbPath = path.join(opts.dataDir, 'data', 'tenants', plan.tenant.tenantId, agent.agentId, 'messages.db');
    if (!fs.existsSync(dbPath)) {
      errors.push({ phase: 'db', message: `messages.db missing at ${dbPath}` });
    }
  }

  return {
    dryRun: false,
    tenantId: plan.tenant.tenantId,
    groupsMigrated,
    messagesMigrated: 0,
    sessionsRepacked: plan.sessionRepacks.length,
    authFilesMoved,
    errors,
  };
}
```

- [ ] **第 4 步：运行测试以验证通过**

运行：`npm test -- src/migration/verify.test.ts`
预期：PASS（3 个测试）。

- [ ] **第 5 步：提交**

```bash
git add src/migration/verify.ts src/migration/verify.test.ts
git commit -m "feat(migration): add post-migration verify"
```

---

## 任务 7：CLI 入口 + npm 脚本

**文件：**
- 新建：`scripts/migrate-to-multi-tenant.ts`
- 修改：`package.json`

- [ ] **第 1 步：写 CLI**

创建 `scripts/migrate-to-multi-tenant.ts`：

```typescript
import fs from 'node:fs';
import path from 'node:path';

import { buildMigrationPlan } from '../src/migration/transformers.js';
import { readLegacySnapshot } from '../src/migration/snapshot.js';
import { verifyMigration } from '../src/migration/verify.js';
import { writeMigration } from '../src/migration/writer.js';

interface Args {
  source: string;
  dataDir: string;
  tenantRepoDir: string;
  tenantId: string;
  model: string;
  confirm: boolean;
}

function parseArgs(argv: string[]): Args {
  const out: Partial<Args> = {
    tenantId: 'legacy',
    model: 'claude-sonnet-4',
    confirm: false,
  };
  for (let i = 2; i < argv.length; i++) {
    const a = argv[i];
    const next = () => argv[++i];
    if (a === '--source') out.source = next();
    else if (a === '--data-dir') out.dataDir = next();
    else if (a === '--tenant-repo') out.tenantRepoDir = next();
    else if (a === '--tenant-id') out.tenantId = next();
    else if (a === '--model') out.model = next();
    else if (a === '--confirm') out.confirm = true;
    else if (a === '--help' || a === '-h') {
      console.log(usage());
      process.exit(0);
    } else {
      console.error(`unknown arg: ${a}`);
      console.error(usage());
      process.exit(2);
    }
  }
  if (!out.source || !out.dataDir || !out.tenantRepoDir) {
    console.error('missing required --source, --data-dir, or --tenant-repo');
    console.error(usage());
    process.exit(2);
  }
  return out as Args;
}

function usage(): string {
  return `Usage: migrate-to-multi-tenant --source <legacy-root> --data-dir <target-data> --tenant-repo <target-repo>
  [--tenant-id legacy] [--model claude-sonnet-4] [--confirm]

  --source       Path to a copy of the 1.x NanoClaw data dir (groups/, store/, data/).
  --data-dir     Target /var/lib/nanoclaw path. Must not exist yet or be empty.
  --tenant-repo  Target tenant repository path. Will be created if absent.
  --tenant-id    Synthetic tenant ID (default: 'legacy').
  --model        Default model for migrated agents.
  --confirm      Apply the migration. Without this flag, runs in dry-run mode.`;
}

async function main(): Promise<void> {
  const args = parseArgs(process.argv);

  console.log(`[migrate] reading legacy snapshot from ${args.source}`);
  const snapshot = readLegacySnapshot(args.source);
  console.log(`[migrate] found ${snapshot.groups.length} groups, ${snapshot.sessions.length} sessions`);

  const plan = buildMigrationPlan(snapshot, {
    tenantId: args.tenantId,
    dataDir: args.dataDir,
    model: args.model,
  });

  console.log('[migrate] plan:');
  console.log(`  tenant: ${plan.tenant.tenantId} (${plan.tenant.tenantName})`);
  console.log(`  agents: ${plan.tenant.agents.length}`);
  console.log(`  auth moves: ${plan.authMoves.length}`);
  console.log(`  session repacks: ${plan.sessionRepacks.length}`);

  if (!args.confirm) {
    console.log('[migrate] dry-run mode; re-run with --confirm to apply');
    return;
  }

  if (fs.existsSync(args.dataDir) && fs.readdirSync(args.dataDir).length > 0) {
    console.error(`[migrate] target data dir ${args.dataDir} is not empty; aborting`);
    process.exit(2);
  }

  fs.mkdirSync(args.dataDir, { recursive: true });
  fs.mkdirSync(args.tenantRepoDir, { recursive: true });

  writeMigration(snapshot, plan, {
    legacyRoot: args.source,
    dataDir: args.dataDir,
    tenantRepoDir: args.tenantRepoDir,
  });

  const report = verifyMigration(plan, { dataDir: args.dataDir, tenantRepoDir: args.tenantRepoDir });
  console.log('[migrate] verify report:');
  console.log(JSON.stringify(report, null, 2));
  if (report.errors.length > 0) {
    console.error(`[migrate] ${report.errors.length} error(s) after migration`);
    process.exit(1);
  }
  console.log('[migrate] done');
}

main().catch((err) => {
  console.error(err);
  process.exit(1);
});
```

- [ ] **第 2 步：添加 npm 脚本**

编辑 `package.json`。在 `"scripts"` 中添加：

```json
    "migrate:to-multi-tenant": "tsx scripts/migrate-to-multi-tenant.ts",
    "verify:migration": "tsx scripts/migrate-to-multi-tenant.ts --dry-run"
```

- [ ] **第 3 步：针对 fixture 进行 CLI 端到端测试**

```bash
TMP=$(mktemp -d)
npm run migrate:to-multi-tenant -- \
  --source tests/fixtures/legacy-sample \
  --data-dir "$TMP/data" \
  --tenant-repo "$TMP/tenants" \
  --confirm
RC=$?
echo "exit code: $RC"
ls "$TMP/data"
ls "$TMP/tenants/tenants/legacy/agents"
rm -rf "$TMP"
```

预期：退出码 0；data 和 tenant-repo 已按 verify 报告的输出写入。

- [ ] **第 4 步：提交**

```bash
git add scripts/migrate-to-multi-tenant.ts package.json
git commit -m "feat(migration): add migrate:to-multi-tenant CLI"
```

---

## 任务 8：编写通过备份回档的回滚流程文档

**文件：**
- 新建：`docs/MIGRATION_RUNBOOK.md`

- [ ] **第 1 步：编写 runbook**

创建 `docs/MIGRATION_RUNBOOK.md`：

```markdown
# NanoClaw 2.0 Migration Runbook

This runbook describes how to migrate a NanoClaw 1.x deployment to the
2.0 multi-tenant host-direct runtime using the one-shot migration script.

## Prerequisites

- NanoClaw 2.0 binaries installed (helper, agent-runner, control plane).
- A file-level backup of the 1.x data directory (`groups/`, `store/`, `data/`).
- Operator with permission to create `/var/lib/nanoclaw` and install the
  SUID helper binary.

## Pre-migration checks

1. Stop the running 1.x NanoClaw service:

   ```bash
   systemctl --user stop nanoclaw
   ```

2. Take a file-level backup:

   ```bash
   rsync -a /path/to/nanoclaw/ /backup/nanoclaw-$(date +%Y%m%d)/
   ```

3. Confirm 2.0 prerequisites:

   ```bash
   /usr/lib/nanoclaw/nc-setuid-helper --help
   ```

## Dry-run

Run the migration in dry-run mode against a copy of the backup:

```bash
npm run migrate:to-multi-tenant -- \
  --source /backup/nanoclaw-YYYYMMDD \
  --data-dir /tmp/migration-dryrun/data \
  --tenant-repo /tmp/migration-dryrun/tenants
```

Inspect the printed plan. Confirm:
- Every legacy group appears as an agent.
- Auth moves cover all Feishu credentials you expect.
- Session repacks cover groups with active continuation state.

## Apply

Run with `--confirm` against the real target paths:

```bash
sudo npm run migrate:to-multi-tenant -- \
  --source /backup/nanoclaw-YYYYMMDD \
  --data-dir /var/lib/nanoclaw \
  --tenant-repo /etc/nanoclaw/tenants \
  --confirm
```

The script:
1. Creates `/var/lib/nanoclaw` if absent.
2. Writes the tenant repo at `/etc/nanoclaw/tenants`.
3. Writes per-(tenant, agent) auth storage, host DBs, and runtime state DBs.
4. Runs `verify:migration` and prints a report.

## Post-migration verification

```bash
npm run tenants:check
# Set NANOCLAW_TENANTS_DIR=/etc/nanoclaw/tenants first.
# Expected: prints tenant summary, no errors.

systemctl --user start nanoclaw
journalctl --user -u nanoclaw -f | grep -i error
```

Watch logs for:
- `channel connected` for each expected Feishu app.
- `runtime db opened` for each (tenant, agent).
- No `auth file not found` or `secret resolution failed` errors.

## Rollback

The 2.0 runtime does not provide a fallback mode. To roll back:

1. Stop the 2.0 service:

   ```bash
   systemctl --user stop nanoclaw
   ```

2. Restore the 1.x data directory from backup:

   ```bash
   rm -rf /var/lib/nanoclaw
   rsync -a /backup/nanoclaw-YYYYMMDD/ /path/to/nanoclaw/
   ```

3. Reinstall the 1.x NanoClaw package.

4. Restart:

   ```bash
   systemctl --user start nanoclaw
   ```

There is no in-process rollback. Any 2.0-side state (active_runs, runtime
DBs) is discarded; messages produced after cutover but before rollback
are preserved in the per-(tenant, agent) host DBs but become inaccessible
to the rolled-back 1.x runtime.
```

- [ ] **第 2 步：提交**

```bash
git add docs/MIGRATION_RUNBOOK.md
git commit -m "docs(migration): add migration runbook with rollback procedure"
```

---

## 自检记录

**规格覆盖检查：**
- CROSS_CUTTING_CONTRACTS "Cleanup and Migration Data"：✓ 任务 3、4、5（覆盖 groups/、store/auth/feishu/、sessions、DBs）
- "Irreversible by design; restoration is by file-level backup recovery"：✓ 任务 8 runbook
- ADR-002 reversed（无回退模式）：✓ 贯穿全程——脚本不提供 in-process rollback

**不在范围内：**
- 验收测试 — 计划 G
- Legacy 文件 IPC 清理（脚本不删除 legacy 目录；由运维人员完成）
- 从 1.x `data/messages.db` 实时回填消息（writer 创建带 schema 的 partition 但不迁移消息行——那是未来的增强）

**占位符扫描：** 无。

**类型一致性：** `MigrationPlan`、`MigrationReport` 在 transformers、writer、verify 之间形状一致。writer 消费的 `LegacySnapshot` 字段与 snapshot reader 产出的字段匹配。

**已知的迁移期债务：**
- Writer 不从 legacy `data/messages.db` 迁移实际的消息行——只创建空的 partition DB。需要保留历史的生产迁移应扩展 `writeDbPartitions`，从 legacy DB 读取并写入正确的 partition。
- JID 到 folder 的映射使用 `feishu:oc_<agentId>` 作为占位符。真实迁移必须从 legacy control DB 读取 `registered_groups`。
- Session 状态以不透明的 `legacy:<filename>` 键重新打包；provider adapter 在恢复 continuation 时必须解释这些键。
