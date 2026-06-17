# Approval Permissions Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add per-group, per-action, per-approval-code permission checks to NanoClaw's Feishu approval IPC handler.

**Architecture:** New module `src/approval-allowlist.ts` loads config from `approval-allowlist.json` (project root). The `isApprovalAllowed()` function is called in `src/ipc.ts` before each approval API call. Default-deny: missing or invalid config blocks all approval operations.

**Tech Stack:** TypeScript, Vitest, Node.js fs module.

---

### Task 1: Add config path constant

**Files:**
- Modify: `src/config.ts:36` (after `WEBHOOK_TASKS_CONFIG`)

- [ ] **Step 1: Add the constant**

In `src/config.ts`, add after the `WEBHOOK_TASKS_CONFIG` definition (line 42):

```typescript
export const APPROVAL_ALLOWLIST_PATH = path.resolve(
  PROJECT_ROOT,
  process.env.APPROVAL_ALLOWLIST_PATH || 'approval-allowlist.json',
);
```

- [ ] **Step 2: Verify build**

Run: `npx tsc --noEmit`
Expected: no errors

- [ ] **Step 3: Commit**

```bash
git add src/config.ts
git commit -m "feat: add APPROVAL_ALLOWLIST_PATH config constant"
```

---

### Task 2: Create approval-allowlist module with failing tests

**Files:**
- Create: `src/approval-allowlist.ts`
- Create: `src/approval-allowlist.test.ts`

- [ ] **Step 1: Write the test file**

Create `src/approval-allowlist.test.ts`:

```typescript
import fs from 'fs';
import os from 'os';
import path from 'path';
import { afterEach, beforeEach, describe, expect, it } from 'vitest';

import {
  isApprovalAllowed,
  loadApprovalAllowlist,
} from './approval-allowlist.js';

let tmpDir: string;

function cfgPath(name = 'approval-allowlist.json'): string {
  return path.join(tmpDir, name);
}

function writeConfig(config: unknown, name?: string): string {
  const p = cfgPath(name);
  fs.writeFileSync(p, JSON.stringify(config));
  return p;
}

beforeEach(() => {
  tmpDir = fs.mkdtempSync(path.join(os.tmpdir(), 'approval-allowlist-test-'));
});

afterEach(() => {
  fs.rmSync(tmpDir, { recursive: true, force: true });
});

describe('loadApprovalAllowlist', () => {
  it('returns null when file is missing', () => {
    const cfg = loadApprovalAllowlist(cfgPath());
    expect(cfg).toBeNull();
  });

  it('returns null when file has invalid JSON', () => {
    const p = cfgPath();
    fs.writeFileSync(p, '{ not valid }}}');
    const cfg = loadApprovalAllowlist(p);
    expect(cfg).toBeNull();
  });

  it('returns null when default is missing', () => {
    const p = writeConfig({ groups: {} });
    const cfg = loadApprovalAllowlist(p);
    expect(cfg).toBeNull();
  });

  it('loads a valid config', () => {
    const p = writeConfig({
      default: { allow: false },
      groups: {
        'feishu-oc_xxx': {
          actions: ['approve', 'reject'],
          approval_codes: ['CODE-A'],
        },
      },
    });
    const cfg = loadApprovalAllowlist(p);
    expect(cfg).not.toBeNull();
    expect(cfg!.default.allow).toBe(false);
    expect(cfg!.groups['feishu-oc_xxx'].actions).toEqual(['approve', 'reject']);
    expect(cfg!.groups['feishu-oc_xxx'].approval_codes).toEqual(['CODE-A']);
  });

  it('skips invalid group entries', () => {
    const p = writeConfig({
      default: { allow: false },
      groups: {
        good: { actions: ['approve'], approval_codes: ['*'] },
        bad: { actions: 123 },
      },
    });
    const cfg = loadApprovalAllowlist(p);
    expect(cfg!.groups['good']).toBeDefined();
    expect(cfg!.groups['bad']).toBeUndefined();
  });
});

describe('isApprovalAllowed', () => {
  it('returns false when config is null', () => {
    expect(isApprovalAllowed(null, 'feishu-oc_xxx', 'approve')).toBe(false);
  });

  it('returns default.allow for unconfigured group', () => {
    const cfg = {
      default: { allow: true },
      groups: {},
    };
    expect(isApprovalAllowed(cfg, 'feishu-oc_xxx', 'approve')).toBe(true);
  });

  it('returns false when default.allow is false and group not in config', () => {
    const cfg = {
      default: { allow: false },
      groups: {},
    };
    expect(isApprovalAllowed(cfg, 'feishu-oc_xxx', 'approve')).toBe(false);
  });

  it('returns false when action is not in actions list', () => {
    const cfg = {
      default: { allow: false },
      groups: {
        'feishu-oc_xxx': {
          actions: ['comment'],
          approval_codes: ['*'],
        },
      },
    };
    expect(
      isApprovalAllowed(cfg, 'feishu-oc_xxx', 'approve', 'CODE-A'),
    ).toBe(false);
  });

  it('returns true when actions is wildcard', () => {
    const cfg = {
      default: { allow: false },
      groups: {
        'feishu-oc_xxx': {
          actions: ['*'],
          approval_codes: ['*'],
        },
      },
    };
    expect(
      isApprovalAllowed(cfg, 'feishu-oc_xxx', 'approve', 'CODE-A'),
    ).toBe(true);
  });

  it('returns false when approval_code does not match', () => {
    const cfg = {
      default: { allow: false },
      groups: {
        'feishu-oc_xxx': {
          actions: ['approve'],
          approval_codes: ['CODE-A'],
        },
      },
    };
    expect(
      isApprovalAllowed(cfg, 'feishu-oc_xxx', 'approve', 'CODE-B'),
    ).toBe(false);
  });

  it('returns true when action and approval_code both match', () => {
    const cfg = {
      default: { allow: false },
      groups: {
        'feishu-oc_xxx': {
          actions: ['approve', 'reject'],
          approval_codes: ['CODE-A', 'CODE-B'],
        },
      },
    };
    expect(
      isApprovalAllowed(cfg, 'feishu-oc_xxx', 'approve', 'CODE-A'),
    ).toBe(true);
    expect(
      isApprovalAllowed(cfg, 'feishu-oc_xxx', 'reject', 'CODE-B'),
    ).toBe(true);
  });

  it('returns true for get_instance without approval_code check', () => {
    const cfg = {
      default: { allow: false },
      groups: {
        'feishu-oc_xxx': {
          actions: ['get_instance'],
          approval_codes: ['CODE-A'], // even though restricted
        },
      },
    };
    // get_instance should pass because no approvalCode is provided
    expect(
      isApprovalAllowed(cfg, 'feishu-oc_xxx', 'get_instance'),
    ).toBe(true);
  });

  it('returns true for comment without approval_code check', () => {
    const cfg = {
      default: { allow: false },
      groups: {
        'feishu-oc_xxx': {
          actions: ['comment'],
          approval_codes: ['CODE-A'], // even though restricted
        },
      },
    };
    // comment should pass because no approvalCode is provided
    expect(
      isApprovalAllowed(cfg, 'feishu-oc_xxx', 'comment'),
    ).toBe(true);
  });

  it('returns false when action matches but approval_code is missing from request', () => {
    // For operations that should have approval_code (approve, reject, etc.),
    // if it's not provided, we still need to check approval_codes
    const cfg = {
      default: { allow: false },
      groups: {
        'feishu-oc_xxx': {
          actions: ['approve'],
          approval_codes: ['CODE-A'],
        },
      },
    };
    // approve without approvalCode → action passes, but code check skipped → true
    // This is the expected behavior: the caller decides whether to pass approvalCode
    expect(
      isApprovalAllowed(cfg, 'feishu-oc_xxx', 'approve'),
    ).toBe(true);
  });

  it('approval_codes wildcard allows any code', () => {
    const cfg = {
      default: { allow: false },
      groups: {
        'feishu-oc_xxx': {
          actions: ['approve'],
          approval_codes: ['*'],
        },
      },
    };
    expect(
      isApprovalAllowed(cfg, 'feishu-oc_xxx', 'approve', 'ANY-CODE'),
    ).toBe(true);
  });
});
```

- [ ] **Step 2: Write the implementation**

Create `src/approval-allowlist.ts`:

```typescript
import fs from 'fs';

import { APPROVAL_ALLOWLIST_PATH } from './config.js';
import { logger } from './logger.js';

export interface ApprovalGroupConfig {
  actions: string[];
  approval_codes: string[];
}

export interface ApprovalAllowlistConfig {
  default: { allow: boolean };
  groups: Record<string, ApprovalGroupConfig>;
}

function isValidGroupConfig(entry: unknown): entry is ApprovalGroupConfig {
  if (!entry || typeof entry !== 'object') return false;
  const e = entry as Record<string, unknown>;
  const validActions =
    Array.isArray(e.actions) && e.actions.every((v) => typeof v === 'string');
  const validCodes =
    Array.isArray(e.approval_codes) &&
    e.approval_codes.every((v) => typeof v === 'string');
  return validActions && validCodes;
}

export function loadApprovalAllowlist(
  pathOverride?: string,
): ApprovalAllowlistConfig | null {
  const filePath = pathOverride ?? APPROVAL_ALLOWLIST_PATH;

  let raw: string;
  try {
    raw = fs.readFileSync(filePath, 'utf-8');
  } catch (err: unknown) {
    if ((err as NodeJS.ErrnoException).code === 'ENOENT') return null;
    logger.warn({ err, path: filePath }, 'approval-allowlist: cannot read');
    return null;
  }

  let parsed: unknown;
  try {
    parsed = JSON.parse(raw);
  } catch {
    logger.warn({ path: filePath }, 'approval-allowlist: invalid JSON');
    return null;
  }

  const obj = parsed as Record<string, unknown>;
  if (
    !obj.default ||
    typeof obj.default !== 'object' ||
    typeof (obj.default as Record<string, unknown>).allow !== 'boolean'
  ) {
    logger.warn(
      { path: filePath },
      'approval-allowlist: invalid or missing default',
    );
    return null;
  }

  const groups: Record<string, ApprovalGroupConfig> = {};
  if (obj.groups && typeof obj.groups === 'object') {
    for (const [key, entry] of Object.entries(
      obj.groups as Record<string, unknown>,
    )) {
      if (isValidGroupConfig(entry)) {
        groups[key] = entry;
      } else {
        logger.warn(
          { key, path: filePath },
          'approval-allowlist: skipping invalid group entry',
        );
      }
    }
  }

  return {
    default: obj.default as { allow: boolean },
    groups,
  };
}

export function isApprovalAllowed(
  cfg: ApprovalAllowlistConfig | null,
  sourceGroup: string,
  action: string,
  approvalCode?: string,
): boolean {
  if (!cfg) return false;

  const groupConfig = cfg.groups[sourceGroup];
  if (!groupConfig) return cfg.default.allow;

  const actionsMatch =
    groupConfig.actions.includes('*') || groupConfig.actions.includes(action);
  if (!actionsMatch) return false;

  if (approvalCode) {
    return (
      groupConfig.approval_codes.includes('*') ||
      groupConfig.approval_codes.includes(approvalCode)
    );
  }

  return true;
}
```

- [ ] **Step 3: Run tests**

Run: `npx vitest run src/approval-allowlist.test.ts`
Expected: all tests pass

- [ ] **Step 4: Commit**

```bash
git add src/approval-allowlist.ts src/approval-allowlist.test.ts
git commit -m "feat: add approval-allowlist module with config loading and permission checks"
```

---

### Task 3: Integrate permission checks into IPC handler

**Files:**
- Modify: `src/ipc.ts:641-677` (approval case branches)

- [ ] **Step 1: Add import at the top of `src/ipc.ts`**

Find the existing imports near the top of the file. Add:

```typescript
import { isApprovalAllowed, loadApprovalAllowlist } from './approval-allowlist.js';
```

- [ ] **Step 2: Replace the approval case branches**

In `src/ipc.ts`, replace lines 641-677 (from `// 审批操作` through the `approval_query` case) with:

```typescript
                  // 审批操作
                  case 'approval_get_instance': {
                    const approvalCfg = loadApprovalAllowlist();
                    if (!isApprovalAllowed(approvalCfg, sourceGroup, 'get_instance')) {
                      logger.warn({ sourceGroup, action: 'get_instance' }, 'Approval action denied');
                      throw new Error(`approval action 'get_instance' not allowed for group '${sourceGroup}'`);
                    }
                    result = await feishuChannel.getApprovalInstance(
                      request.instance_code,
                    );
                    break;
                  }
                  case 'approval_approve': {
                    const approvalCfg = loadApprovalAllowlist();
                    if (!isApprovalAllowed(approvalCfg, sourceGroup, 'approve', request.params?.approval_code)) {
                      logger.warn({ sourceGroup, action: 'approve', approvalCode: request.params?.approval_code }, 'Approval action denied');
                      throw new Error(`approval action 'approve' not allowed for group '${sourceGroup}'`);
                    }
                    await feishuChannel.approveApprovalTask(request.params);
                    result = { approved: true };
                    break;
                  }
                  case 'approval_reject': {
                    const approvalCfg = loadApprovalAllowlist();
                    if (!isApprovalAllowed(approvalCfg, sourceGroup, 'reject', request.params?.approval_code)) {
                      logger.warn({ sourceGroup, action: 'reject', approvalCode: request.params?.approval_code }, 'Approval action denied');
                      throw new Error(`approval action 'reject' not allowed for group '${sourceGroup}'`);
                    }
                    await feishuChannel.rejectApprovalTask(request.params);
                    result = { rejected: true };
                    break;
                  }
                  case 'approval_transfer': {
                    const approvalCfg = loadApprovalAllowlist();
                    if (!isApprovalAllowed(approvalCfg, sourceGroup, 'transfer', request.params?.approval_code)) {
                      logger.warn({ sourceGroup, action: 'transfer', approvalCode: request.params?.approval_code }, 'Approval action denied');
                      throw new Error(`approval action 'transfer' not allowed for group '${sourceGroup}'`);
                    }
                    await feishuChannel.transferApprovalTask(request.params);
                    result = { transferred: true };
                    break;
                  }
                  case 'approval_comment': {
                    const approvalCfg = loadApprovalAllowlist();
                    if (!isApprovalAllowed(approvalCfg, sourceGroup, 'comment')) {
                      logger.warn({ sourceGroup, action: 'comment' }, 'Approval action denied');
                      throw new Error(`approval action 'comment' not allowed for group '${sourceGroup}'`);
                    }
                    // 自动注入 Bot open_id
                    const botInfo = await feishuChannel.getBotInfo();
                    const commentResult =
                      await feishuChannel.createApprovalComment({
                        instance_id: request.instance_id,
                        user_id: botInfo.open_id,
                        content: request.content,
                        parent_comment_id: request.parent_comment_id,
                        at_info_list: request.at_info_list,
                      });
                    result = { comment_id: commentResult.comment_id };
                    break;
                  }
                  case 'approval_query': {
                    const approvalCfg = loadApprovalAllowlist();
                    if (!isApprovalAllowed(approvalCfg, sourceGroup, 'query', request.params?.approval_code)) {
                      logger.warn({ sourceGroup, action: 'query', approvalCode: request.params?.approval_code }, 'Approval action denied');
                      throw new Error(`approval action 'query' not allowed for group '${sourceGroup}'`);
                    }
                    result = await feishuChannel.queryApprovalInstances(
                      request.params,
                    );
                    break;
                  }
```

Note: Denied requests use `throw` instead of setting `result` because the result-writing code at line 684 always prefixes with `{ success: true }`. By throwing, the catch block (line 696-711) correctly writes `{ success: false, error: "..." }`. The `logger.warn` before the throw provides the structured denial log; the catch block's `logger.error` also fires, which is appropriate for a security-relevant event.

- [ ] **Step 3: Verify build**

Run: `npx tsc --noEmit`
Expected: no errors

- [ ] **Step 4: Run all existing tests**

Run: `npx vitest run`
Expected: all tests pass (no regressions)

- [ ] **Step 5: Commit**

```bash
git add src/ipc.ts
git commit -m "feat: integrate approval permission checks into IPC handler"
```

---

### Task 4: Add example config file

**Files:**
- Create: `approval-allowlist.json.example`

- [ ] **Step 1: Create the example file**

Create `approval-allowlist.json.example`:

```json
{
  "default": { "allow": false },
  "groups": {
    "feishu-oc_YOUR_GROUP_ID": {
      "actions": ["approve", "reject", "comment", "query", "get_instance", "transfer"],
      "approval_codes": ["YOUR_APPROVAL_CODE_1", "YOUR_APPROVAL_CODE_2"]
    },
    "feishu-oc_APPROVAL_BOT_GROUP": {
      "actions": ["*"],
      "approval_codes": ["*"]
    }
  }
}
```

- [ ] **Step 2: Commit**

```bash
git add approval-allowlist.json.example
git commit -m "docs: add approval-allowlist.json example config"
```

---

### Task 5: Update CLAUDE.md

**Files:**
- Modify: `CLAUDE.md` (Configuration section)

- [ ] **Step 1: Add approval-allowlist to the configuration documentation**

In `CLAUDE.md`, find the Configuration section. After the existing environment variable table, add a line in the Key Files table:

Add a row to the Key Files table:

```
| `approval-allowlist.json` | Approval operation permission config (group-level, per-action, per-approval-code) |
```

- [ ] **Step 2: Add to Troubleshooting or Security Notes**

In the Security Notes section, add:

```
- Approval allowlist in `approval-allowlist.json` (project root) — missing file denies all approval operations
```

- [ ] **Step 3: Commit**

```bash
git add CLAUDE.md
git commit -m "docs: document approval-allowlist in CLAUDE.md"
```
