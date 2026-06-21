# Approval Permissions Design

## Problem

NanoClaw supports Feishu approval operations (approve, reject, transfer, comment, query) via IPC from container agents. Currently, any group's agent can execute any approval action without restriction. This poses a security risk — a rogue or misconfigured agent in any group could approve or reject pending approvals.

With the addition of webhook-triggered tasks that use approval operations, we need a permission system to control which groups can perform which approval actions.

## Requirements

1. Per-group permission control keyed by `group_folder` (the IPC layer's `sourceGroup`)
2. Per-operation-type granularity: approve, reject, transfer, comment, query, get_instance
3. Per-approval-code granularity: restrict which approval definitions a group can operate on
4. Wildcard support: `"*"` in actions or approval_codes means "all"
5. Default-deny: groups not in the allowlist are blocked; missing config file also denies all
6. Compatible with both regular IPC and isolated IPC modes
7. Layered check: `get_instance` and `comment` only check action permission (no approval_code check), since their request params don't contain approval_code

## Configuration

### File Location

`approval-allowlist.json` in the project root directory (alongside `.env`).

Path constant in `src/config.ts`:
```typescript
const APPROVAL_ALLOWLIST_PATH = path.join(PROJECT_ROOT, 'approval-allowlist.json');
```

### Schema

```typescript
interface ApprovalAllowlistConfig {
  default: { allow: boolean };
  groups: Record<string, ApprovalGroupConfig>;
}

interface ApprovalGroupConfig {
  actions: string[];        // ["approve", "reject", ...] or ["*"]
  approval_codes: string[]; // ["C97A40F4-xxx", ...] or ["*"]
}
```

### Example (`approval-allowlist.json.example`)

```json
{
  "default": { "allow": false },
  "groups": {
    "feishu-oc_xxx": {
      "actions": ["approve", "reject", "comment", "query", "get_instance", "transfer"],
      "approval_codes": ["C97A40F4-xxx", "F42B8E1A-yyy"]
    },
    "feishu-oc_approval_bot": {
      "actions": ["*"],
      "approval_codes": ["*"]
    }
  }
}
```

### Behavior When File Is Missing

If `approval-allowlist.json` does not exist or fails to parse, all approval operations are denied. This ensures the system is secure by default.

## Authorization Logic

### Module: `src/approval-allowlist.ts`

**Functions:**

- `loadApprovalAllowlist(): ApprovalAllowlistConfig | null` — loads and parses the config file from disk on each call (no caching), supporting hot updates. Returns null if file is missing or invalid.
- `isApprovalAllowed(sourceGroup: string, action: string, approvalCode?: string): boolean` — checks whether a specific approval action is permitted for the given group.

**`isApprovalAllowed` flow:**

1. Load config. If null → return `false`.
2. Look up `groups[sourceGroup]`.
3. If not found → return `default.allow`.
4. Check action: if `actions` contains `"*"` or the requested action → proceed. Otherwise → return `false`.
5. If `approvalCode` is provided (non-null, non-empty):
   - Check if `approval_codes` contains `"*"` or the requested code → return result.
   - Otherwise → return `false`.
6. If no `approvalCode` provided → return `true` (action check passed, no code check needed).

### Action Names

The action strings used in config and checks:

| Config action name | IPC request type |
|--------------------|------------------|
| `get_instance` | `approval_get_instance` |
| `approve` | `approval_approve` |
| `reject` | `approval_reject` |
| `transfer` | `approval_transfer` |
| `comment` | `approval_comment` |
| `query` | `approval_query` |

### Layered approval_code Check

| Operation | approval_code available in request? | approval_code check? |
|-----------|--------------------------------------|----------------------|
| `approval_get_instance` | No | No |
| `approval_approve` | Yes (`params.approval_code`) | Yes |
| `approval_reject` | Yes (`params.approval_code`) | Yes |
| `approval_transfer` | Yes (`params.approval_code`) | Yes |
| `approval_comment` | No | No |
| `approval_query` | Yes (`params.approval_code`) | Yes |

## Integration Point

### `src/ipc.ts` — Feishu IPC Handler

In the `processIpcFiles` function, within the feishu requests processing loop (around lines 641-677), each approval case branch gets a permission check before executing the API call:

```typescript
case 'approval_approve':
  if (!isApprovalAllowed(sourceGroup, 'approve', request.params?.approval_code)) {
    result = { error: "approval action 'approve' not allowed for group '" + sourceGroup + "'" };
    break;
  }
  await feishuChannel.approveApprovalTask(request.params);
  result = { approved: true };
  break;
```

The same pattern applies to all six approval operations. For `get_instance` and `comment`, the `approvalCode` parameter is omitted (only action check).

### Error Handling

- Denied requests return `{ success: false, error: "<message>" }` to the container via the IPC result file.
- Host logs a `logger.warn` with `{ sourceGroup, action, approvalCode }`.
- Other IPC requests in the same poll cycle are unaffected.

## Isolated IPC Compatibility

The approval permission check works identically for both regular and isolated IPC modes because:

1. Both modes are processed in the same `for (const { dir, sourceGroup, isMain } of ipcDirs)` loop in `processIpcFiles`.
2. The `sourceGroup` is correctly resolved for isolated IPC dirs via `discoverIsolatedIpcDirs()` which extracts the group folder from the path `data/sessions/{group}/isolated-ipc-*/`.
3. No changes needed to the isolated IPC infrastructure.

## File Changes

| File | Change |
|------|--------|
| `src/approval-allowlist.ts` | **New** — config loading and `isApprovalAllowed` logic |
| `src/approval-allowlist.test.ts` | **New** — unit tests |
| `src/ipc.ts` | **Modify** — add permission checks in approval case branches |
| `src/config.ts` | **Modify** — add `APPROVAL_ALLOWLIST_PATH` constant |
| `approval-allowlist.json.example` | **New** — example configuration |

## Test Cases

1. Config file missing → all operations denied
2. Config file invalid JSON → all operations denied
3. Group not in config → follows `default.allow`
4. `actions: ["*"]` → all actions pass
5. `approval_codes: ["*"]` → all codes pass
6. Exact match: action and code both match → allowed
7. Action matches but code doesn't → denied
8. Code matches but action doesn't → denied
9. `get_instance` and `comment` skip approval_code check
10. No approval_code provided (for operations that support it) → only action check
11. Hot reload: config file changes take effect on next IPC poll cycle
