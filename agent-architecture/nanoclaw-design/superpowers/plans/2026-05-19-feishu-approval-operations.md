# Feishu Approval Operations Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add approval operations (approve, reject, transfer, comment, query) for Feishu, enabling container agents to manage approval workflows.

**Architecture:** Follows existing MCP + IPC pattern: FeishuClient defines API methods → FeishuChannel proxies → IPC handler processes requests → MCP tools define interface → Skill documents usage.

**Tech Stack:** TypeScript, Feishu Open API (Approval v4), MCP SDK, Zod for schema validation

---

## File Structure

| File | Change | Responsibility |
|------|--------|----------------|
| `src/feishu/client.ts` | Modify | Add getBotInfo() and 6 approval API methods |
| `src/channels/feishu.ts` | Modify | Add 7 proxy methods for approval operations |
| `src/ipc.ts` | Modify | Add case handlers for approval IPC requests |
| `container/agent-runner/src/ipc-mcp-stdio.ts` | Modify | Add 6 MCP tool definitions for approval operations |
| `container/skills/feishu-approval/SKILL.md` | Create | Document usage, triggers, parameters, examples |

---

## Task 1: FeishuClient - Add getBotInfo()

**Files:**
- Modify: `src/feishu/client.ts` (after existing methods, around line 3140)

- [ ] **Step 1: Add getBotInfo method**

Add after the Task methods section (around line 2835, after `searchTasklist`):

```typescript
  // ---------------------------------------------------------------------------
  // Bot Info Operations
  // ---------------------------------------------------------------------------

  /**
   * 获取 Bot 信息
   * GET /open-apis/bot/v3/info
   * 返回 Bot 的 open_id，用于审批评论等操作
   */
  async getBotInfo(): Promise<{ open_id: string; app_id: string }> {
    const response = await this.client.request({
      url: '/open-apis/bot/v3/info',
      method: 'GET',
    });
    if (response.code !== 0) {
      throw new Error(
        `Failed to get bot info: ${response.msg} (code: ${response.code})`,
      );
    }
    log.info({ bot_open_id: response.data?.bot?.open_id }, 'Bot info retrieved');
    return response.data?.bot;
  }
```

- [ ] **Step 2: Commit**

```bash
git add src/feishu/client.ts
git commit -m "feat(feishu): add getBotInfo() method for Bot identity retrieval"
```

---

## Task 2: FeishuClient - Add Approval API Methods

**Files:**
- Modify: `src/feishu/client.ts` (after getBotInfo, around line 3150)

- [ ] **Step 1: Add approval methods**

Add after `getBotInfo()`:

```typescript
  // ---------------------------------------------------------------------------
  // Approval (飞书审批) Operations — Approval v4 API
  // ---------------------------------------------------------------------------

  /**
   * 获取审批实例详情
   * GET /open-apis/approval/v4/instances/{instance_code}
   */
  async getApprovalInstance(instanceCode: string): Promise<any> {
    const response = await this.client.request({
      url: `/open-apis/approval/v4/instances/${instanceCode}`,
      method: 'GET',
      params: { user_id_type: 'open_id' },
    });
    if (response.code !== 0) {
      throw new Error(
        `Failed to get approval instance: ${response.msg} (code: ${response.code})`,
      );
    }
    log.info({ instanceCode }, 'Approval instance retrieved');
    return response.data?.instance;
  }

  /**
   * 同意审批任务
   * POST /open-apis/approval/v4/tasks/approve
   */
  async approveApprovalTask(params: {
    approval_code: string;
    instance_code: string;
    user_id: string;
    task_id: string;
    comment?: string;
    form?: string;
  }): Promise<void> {
    const response = await this.client.request({
      url: '/open-apis/approval/v4/tasks/approve',
      method: 'POST',
      params: { user_id_type: 'open_id' },
      data: params,
    });
    if (response.code !== 0) {
      throw new Error(
        `Failed to approve approval task: ${response.msg} (code: ${response.code})`,
      );
    }
    log.info({ instance_code: params.instance_code, task_id: params.task_id }, 'Approval task approved');
  }

  /**
   * 拒绝审批任务
   * POST /open-apis/approval/v4/tasks/reject
   */
  async rejectApprovalTask(params: {
    approval_code: string;
    instance_code: string;
    user_id: string;
    task_id: string;
    comment?: string;
    form?: string;
  }): Promise<void> {
    const response = await this.client.request({
      url: '/open-apis/approval/v4/tasks/reject',
      method: 'POST',
      params: { user_id_type: 'open_id' },
      data: params,
    });
    if (response.code !== 0) {
      throw new Error(
        `Failed to reject approval task: ${response.msg} (code: ${response.code})`,
      );
    }
    log.info({ instance_code: params.instance_code, task_id: params.task_id }, 'Approval task rejected');
  }

  /**
   * 转交审批任务
   * POST /open-apis/approval/v4/tasks/transfer
   */
  async transferApprovalTask(params: {
    approval_code: string;
    instance_code: string;
    user_id: string;
    task_id: string;
    transfer_user_id: string;
    comment?: string;
  }): Promise<void> {
    const response = await this.client.request({
      url: '/open-apis/approval/v4/tasks/transfer',
      method: 'POST',
      params: { user_id_type: 'open_id' },
      data: params,
    });
    if (response.code !== 0) {
      throw new Error(
        `Failed to transfer approval task: ${response.msg} (code: ${response.code})`,
      );
    }
    log.info(
      { instance_code: params.instance_code, task_id: params.task_id, transfer_user_id: params.transfer_user_id },
      'Approval task transferred',
    );
  }

  /**
   * 创建审批评论
   * POST /open-apis/approval/v4/instances/{instance_id}/comments
   * user_id 由 Host 自动注入（Bot open_id）
   */
  async createApprovalComment(params: {
    instance_id: string;
    user_id: string;
    content: string;
    parent_comment_id?: string;
    at_info_list?: Array<{ user_id: string; name: string; offset: string }>;
  }): Promise<{ comment_id: string }> {
    const response = await this.client.request({
      url: `/open-apis/approval/v4/instances/${params.instance_id}/comments`,
      method: 'POST',
      params: { user_id_type: 'open_id', user_id: params.user_id },
      data: {
        content: params.content,
        parent_comment_id: params.parent_comment_id,
        at_info_list: params.at_info_list,
      },
    });
    if (response.code !== 0) {
      throw new Error(
        `Failed to create approval comment: ${response.msg} (code: ${response.code})`,
      );
    }
    log.info({ instance_id: params.instance_id }, 'Approval comment created');
    return { comment_id: response.data?.comment_id };
  }

  /**
   * 查询审批实例列表
   * POST /open-apis/approval/v4/instances/query
   */
  async queryApprovalInstances(params: {
    approval_code?: string;
    instance_status?: string;
    user_id?: string;
    start_time?: number;
    end_time?: number;
    page_size?: number;
    page_token?: string;
  }): Promise<any> {
    const response = await this.client.request({
      url: '/open-apis/approval/v4/instances/query',
      method: 'POST',
      params: { user_id_type: 'open_id' },
      data: params,
    });
    if (response.code !== 0) {
      throw new Error(
        `Failed to query approval instances: ${response.msg} (code: ${response.code})`,
      );
    }
    log.info({ approval_code: params.approval_code, count: response.data?.instances?.length }, 'Approval instances queried');
    return response.data;
  }
```

- [ ] **Step 2: Commit**

```bash
git add src/feishu/client.ts
git commit -m "feat(feishu): add approval API methods (getInstance, approve, reject, transfer, comment, query)"
```

---

## Task 3: FeishuChannel - Add Proxy Methods

**Files:**
- Modify: `src/channels/feishu.ts` (after Task methods, around line 895)

- [ ] **Step 1: Add approval proxy methods**

Add after `removeTasklistMembers` method (around line 894):

```typescript
  // ==================== 审批操作方法（供 Host IPC 调用） ====================

  async getBotInfo(): Promise<{ open_id: string; app_id: string }> {
    return await this.client.getBotInfo();
  }

  async getApprovalInstance(instanceCode: string): Promise<any> {
    return await this.client.getApprovalInstance(instanceCode);
  }

  async approveApprovalTask(params: any): Promise<void> {
    return await this.client.approveApprovalTask(params);
  }

  async rejectApprovalTask(params: any): Promise<void> {
    return await this.client.rejectApprovalTask(params);
  }

  async transferApprovalTask(params: any): Promise<void> {
    return await this.client.transferApprovalTask(params);
  }

  async createApprovalComment(params: any): Promise<{ comment_id: string }> {
    return await this.client.createApprovalComment(params);
  }

  async queryApprovalInstances(params: any): Promise<any> {
    return await this.client.queryApprovalInstances(params);
  }
```

- [ ] **Step 2: Commit**

```bash
git add src/channels/feishu.ts
git commit -m "feat(feishu): add approval proxy methods to FeishuChannel"
```

---

## Task 4: IPC Handler - Add Approval Cases

**Files:**
- Modify: `src/ipc.ts` (in the switch statement, after tasklist cases, around line 582)

- [ ] **Step 1: Add approval IPC case handlers**

Add before `default:` case (around line 582):

```typescript
                  // 审批操作
                  case 'approval_get_instance':
                    result = await feishuChannel.getApprovalInstance(
                      request.instance_code,
                    );
                    break;
                  case 'approval_approve':
                    await feishuChannel.approveApprovalTask(request.params);
                    result = { approved: true };
                    break;
                  case 'approval_reject':
                    await feishuChannel.rejectApprovalTask(request.params);
                    result = { rejected: true };
                    break;
                  case 'approval_transfer':
                    await feishuChannel.transferApprovalTask(request.params);
                    result = { transferred: true };
                    break;
                  case 'approval_comment': {
                    // 自动注入 Bot open_id
                    const botInfo = await feishuChannel.getBotInfo();
                    const commentResult = await feishuChannel.createApprovalComment({
                      instance_id: request.instance_id,
                      user_id: botInfo.open_id,
                      content: request.content,
                      parent_comment_id: request.parent_comment_id,
                      at_info_list: request.at_info_list,
                    });
                    result = { comment_id: commentResult.comment_id };
                    break;
                  }
                  case 'approval_query':
                    result = await feishuChannel.queryApprovalInstances(
                      request.params,
                    );
                    break;
```

- [ ] **Step 2: Commit**

```bash
git add src/ipc.ts
git commit -m "feat(ipc): add approval request handlers with Bot identity auto-injection"
```

---

## Task 5: MCP Tools - Add Approval Tool Definitions

**Files:**
- Modify: `container/agent-runner/src/ipc-mcp-stdio.ts` (after feishu_task tools, around line 2300)

- [ ] **Step 1: Add feishu_approval_get_instance tool**

Add after the last feishu_task tool (find end of file or after tasklist tools):

```typescript
// ==================== 飞书审批工具 ====================

server.tool(
  'feishu_approval_get_instance',
  '获取审批实例详情，返回 task_list（审批任务列表）和审批人信息。用于通过/拒绝/转交前获取必要参数。',
  {
    instance_code: z.string().describe('审批实例 Code'),
  },
  async (args) => {
    const requestId = writeIpcFile(FEISHU_REQUESTS_DIR, {
      type: 'approval_get_instance',
      instance_code: args.instance_code,
      groupFolder,
      timestamp: new Date().toISOString(),
    });

    try {
      const result = await waitForFeishuResult(requestId);
      if (result.success) {
        return {
          content: [{ type: 'text' as const, text: JSON.stringify(result, null, 2) }],
        };
      } else {
        return { content: [{ type: 'text' as const, text: `获取审批实例失败: ${result.error}` }], isError: true };
      }
    } catch (error) {
      return { content: [{ type: 'text' as const, text: `获取审批实例超时: ${error instanceof Error ? error.message : String(error)}` }], isError: true };
    }
  },
);

server.tool(
  'feishu_approval_approve',
  '同意审批任务。需先调用 feishu_approval_get_instance 获取 task_id 和审批人 user_id。',
  {
    approval_code: z.string().describe('审批定义 Code（从 get_instance 返回的 approval.code 获取）'),
    instance_code: z.string().describe('审批实例 Code'),
    user_id: z.string().describe('审批人 open_id（从 get_instance 的 task_list 中获取）'),
    task_id: z.string().describe('审批任务 ID（从 get_instance 的 task_list 中获取）'),
    comment: z.string().optional().describe('审批意见（可选）'),
  },
  async (args) => {
    const requestId = writeIpcFile(FEISHU_REQUESTS_DIR, {
      type: 'approval_approve',
      params: {
        approval_code: args.approval_code,
        instance_code: args.instance_code,
        user_id: args.user_id,
        task_id: args.task_id,
        comment: args.comment,
      },
      groupFolder,
      timestamp: new Date().toISOString(),
    });

    try {
      const result = await waitForFeishuResult(requestId);
      if (result.success) {
        return { content: [{ type: 'text' as const, text: `审批已同意!` }] };
      } else {
        return { content: [{ type: 'text' as const, text: `同意审批失败: ${result.error}` }], isError: true };
      }
    } catch (error) {
      return { content: [{ type: 'text' as const, text: `同意审批超时: ${error instanceof Error ? error.message : String(error)}` }], isError: true };
    }
  },
);

server.tool(
  'feishu_approval_reject',
  '拒绝审批任务。需先调用 feishu_approval_get_instance 获取 task_id 和审批人 user_id。',
  {
    approval_code: z.string().describe('审批定义 Code'),
    instance_code: z.string().describe('审批实例 Code'),
    user_id: z.string().describe('审批人 open_id'),
    task_id: z.string().describe('审批任务 ID'),
    comment: z.string().optional().describe('审批意见（可选）'),
  },
  async (args) => {
    const requestId = writeIpcFile(FEISHU_REQUESTS_DIR, {
      type: 'approval_reject',
      params: {
        approval_code: args.approval_code,
        instance_code: args.instance_code,
        user_id: args.user_id,
        task_id: args.task_id,
        comment: args.comment,
      },
      groupFolder,
      timestamp: new Date().toISOString(),
    });

    try {
      const result = await waitForFeishuResult(requestId);
      if (result.success) {
        return { content: [{ type: 'text' as const, text: `审批已拒绝!` }] };
      } else {
        return { content: [{ type: 'text' as const, text: `拒绝审批失败: ${result.error}` }], isError: true };
      }
    } catch (error) {
      return { content: [{ type: 'text' as const, text: `拒绝审批超时: ${error instanceof Error ? error.message : String(error)}` }], isError: true };
    }
  },
);

server.tool(
  'feishu_approval_transfer',
  '转交审批任务给其他人。需先调用 feishu_approval_get_instance 获取 task_id 和审批人 user_id。',
  {
    approval_code: z.string().describe('审批定义 Code'),
    instance_code: z.string().describe('审批实例 Code'),
    user_id: z.string().describe('当前审批人 open_id'),
    task_id: z.string().describe('审批任务 ID'),
    transfer_user_id: z.string().describe('被转交人 open_id'),
    comment: z.string().optional().describe('转交说明（可选）'),
  },
  async (args) => {
    const requestId = writeIpcFile(FEISHU_REQUESTS_DIR, {
      type: 'approval_transfer',
      params: {
        approval_code: args.approval_code,
        instance_code: args.instance_code,
        user_id: args.user_id,
        task_id: args.task_id,
        transfer_user_id: args.transfer_user_id,
        comment: args.comment,
      },
      groupFolder,
      timestamp: new Date().toISOString(),
    });

    try {
      const result = await waitForFeishuResult(requestId);
      if (result.success) {
        return { content: [{ type: 'text' as const, text: `审批已转交给 ${args.transfer_user_id}!` }] };
      } else {
        return { content: [{ type: 'text' as const, text: `转交审批失败: ${result.error}` }], isError: true };
      }
    } catch (error) {
      return { content: [{ type: 'text' as const, text: `转交审批超时: ${error instanceof Error ? error.message : String(error)}` }], isError: true };
    }
  },
);

server.tool(
  'feishu_approval_comment',
  '在审批实例下创建评论。使用 Bot 身份（自动注入），Agent 只需提供 content。',
  {
    instance_id: z.string().describe('审批实例 Code'),
    content: z.string().describe('评论内容，JSON 字符串格式，如 {"text": "评论内容"}'),
    parent_comment_id: z.string().optional().describe('父评论 ID（用于回复评论）'),
  },
  async (args) => {
    const requestId = writeIpcFile(FEISHU_REQUESTS_DIR, {
      type: 'approval_comment',
      instance_id: args.instance_id,
      content: args.content,
      parent_comment_id: args.parent_comment_id,
      groupFolder,
      timestamp: new Date().toISOString(),
    });

    try {
      const result = await waitForFeishuResult(requestId);
      if (result.success) {
        return { content: [{ type: 'text' as const, text: `评论已创建! 评论 ID: ${result.comment_id}` }] };
      } else {
        return { content: [{ type: 'text' as const, text: `创建评论失败: ${result.error}` }], isError: true };
      }
    } catch (error) {
      return { content: [{ type: 'text' as const, text: `创建评论超时: ${error instanceof Error ? error.message : String(error)}` }], isError: true };
    }
  },
);

server.tool(
  'feishu_approval_query',
  '查询审批实例列表。支持按审批定义、状态、用户等条件筛选。',
  {
    approval_code: z.string().optional().describe('审批定义 Code（可选）'),
    instance_status: z.string().optional().describe('实例状态：PENDING/APPROVED/REJECTED（可选）'),
    user_id: z.string().optional().describe('用户 open_id（可选）'),
    start_time: z.number().optional().describe('开始时间（毫秒时间戳，可选）'),
    end_time: z.number().optional().describe('结束时间（毫秒时间戳，可选）'),
    page_size: z.number().optional().describe('每页数量（默认 20）'),
    page_token: z.string().optional().describe('分页 token'),
  },
  async (args) => {
    const requestId = writeIpcFile(FEISHU_REQUESTS_DIR, {
      type: 'approval_query',
      params: args,
      groupFolder,
      timestamp: new Date().toISOString(),
    });

    try {
      const result = await waitForFeishuResult(requestId);
      if (result.success) {
        const instances = result.instances || [];
        if (instances.length === 0) {
          return { content: [{ type: 'text' as const, text: '未找到审批实例。' }] };
        }
        const formatted = instances
          .map((inst: any) => `- ${inst.title || '无标题'}\n  实例 Code: ${inst.instance_code}\n  状态: ${inst.status}`)
          .join('\n\n');
        return { content: [{ type: 'text' as const, text: `找到 ${instances.length} 个审批实例:\n\n${formatted}` }] };
      } else {
        return { content: [{ type: 'text' as const, text: `查询审批失败: ${result.error}` }], isError: true };
      }
    } catch (error) {
      return { content: [{ type: 'text' as const, text: `查询审批超时: ${error instanceof Error ? error.message : String(error)}` }], isError: true };
    }
  },
);
```

- [ ] **Step 2: Commit**

```bash
git add container/agent-runner/src/ipc-mcp-stdio.ts
git commit -m "feat(mcp): add feishu_approval_* MCP tools for approval operations"
```

---

## Task 6: Create Container Skill Document

**Files:**
- Create: `container/skills/feishu-approval/SKILL.md`

- [ ] **Step 1: Create skill directory and file**

```bash
mkdir -p container/skills/feishu-approval
```

- [ ] **Step 2: Write SKILL.md**

```markdown
---
name: feishu-approval
description: |
  飞书审批操作工具。

  **当以下情况时使用此 Skill**：
  (1) 需要同意、拒绝、转交审批任务
  (2) 需要在审批实例下添加评论
  (3) 需要查询审批实例详情或列表
  (4) 用户提到"审批"、"approve"、"reject"、"转交"

  **重要说明**：
  - 通过/拒绝/转交前必须先调用 feishu_approval_get_instance 获取 task_id 和审批人 user_id
  - 评论操作使用 Bot 身份（自动注入），无需提供 user_id
  - 所有操作需要飞书凭证，已自动注入
  - 不要使用 Bash 命令，使用对应的 MCP 工具
---

# Feishu Approval (飞书审批) SKILL

## 操作方式

- 所有操作通过 MCP 工具完成
- 审批实例 Code 格式：UUID（如 `81D31358-93AF-92D6-7425-01A5D67C4E71`）
- 用户 ID 使用 open_id（如 `ou_xxx`）

---

## 快速索引：意图 → MCP 工具

| 用户意图 | MCP 工具 | 说明 |
|---------|---------|------|
| 查询审批实例详情 | `feishu_approval_get_instance` | 获取 task_list 和审批人信息 |
| 同意审批 | `feishu_approval_approve` | 需先查询获取 task_id/user_id |
| 拒绝审批 | `feishu_approval_reject` | 需先查询获取 task_id/user_id |
| 转交审批 | `feishu_approval_transfer` | 需先查询并指定被转交人 |
| 创建评论 | `feishu_approval_comment` | 使用 Bot 身份（自动） |
| 查询审批列表 | `feishu_approval_query` | 按条件筛选审批实例 |

---

## 通过/拒绝/转交流程（两步调用）

### 步骤 1：查询审批实例

```
feishu_approval_get_instance(instance_code: "审批实例Code")
```

**返回关键信息**：
- `approval.code`: 审批定义 Code（用于 approve/reject/transfer）
- `task_list`: 审批任务列表
  - `id`: task_id（审批任务 ID）
  - `custom_approver`: 审批人信息
    - `user_id`: 审批人 open_id
  - `status`: 任务状态（PENDING/APPROVED/REJECTED）

**审批人判断逻辑**：
- 从 task_list 中找 status 为 `PENDING` 的任务
- 通常第一个 PENDING 任务是当前待处理节点
- 如果有多个 PENDING 任务，需根据上下文判断（如用户指定了审批人）

### 步骤 2：执行审批操作

```
// 同意审批
feishu_approval_approve(
  approval_code: "从 get_instance 返回的 approval.code",
  instance_code: "审批实例Code",
  user_id: "从 task_list 中获取的审批人 open_id",
  task_id: "从 task_list 中获取的 id",
  comment: "审批意见（可选）"
)

// 拒绝审批
feishu_approval_reject(...) // 参数同 approve

// 转交审批
feishu_approval_transfer(
  ...,
  transfer_user_id: "被转交人的 open_id"
)
```

---

## 评论操作

```
feishu_approval_comment(
  instance_id: "审批实例Code",
  content: '{"text": "评论内容"}',  // JSON 字符串格式
  parent_comment_id: "父评论ID（可选，用于回复）"
)
```

**content 格式说明**：
- 必须是 JSON 字符串：`{"text": "评论内容"}`
- 如需 @用户：在 text 中添加 `@username`，并设置 at_info_list（通常由 Host 处理）

**身份说明**：评论使用 Bot 身份，Host 自动获取 Bot open_id 并注入。

---

## 查询审批列表

```
feishu_approval_query(
  approval_code: "审批定义Code（可选）",
  instance_status: "PENDING",  // PENDING/APPROVED/REJECTED
  user_id: "用户open_id（可选）",
  start_time: 1700000000000,   // 毫秒时间戳
  end_time: 1710000000000,
  page_size: 20
)
```

---

## 使用场景示例

### 场景 1：同意审批

```
// 步骤 1：查询实例
feishu_approval_get_instance(instance_code: "81D31358-...")

// 步骤 2：从返回中提取
// - approval_code = result.approval.code
// - task_id = result.task_list[0].id（找 PENDING 状态的任务）
// - user_id = result.task_list[0].custom_approver.user_id

// 步骤 3：同意审批
feishu_approval_approve(
  approval_code: "7C468A54-...",
  instance_code: "81D31358-...",
  user_id: "ou_xxx",
  task_id: "12345",
  comment: "同意，请继续执行"
)
```

### 场景 2：转交审批

```
// 步骤 1-2：同上，获取必要参数

// 步骤 3：转交给其他人
feishu_approval_transfer(
  approval_code: "...",
  instance_code: "...",
  user_id: "当前审批人 open_id",
  task_id: "...",
  transfer_user_id: "ou_yyy",  // 被转交人
  comment: "转交给张三处理"
)
```

### 场景 3：添加评论

```
// 直接添加评论，无需先查询
feishu_approval_comment(
  instance_id: "81D31358-...",
  content: '{"text": "我已了解该审批内容，稍后会处理"}'
)
```

---

## 常见错误

| 错误码 | 描述 | 解决方案 |
|--------|------|----------|
| `1390001` | 参数无效 | 检查必填字段和格式 |
| `1390002` | 审批定义 Code 不存在 | 确认 approval_code 正确 |
| `1390003` | 审批实例 Code 不存在 | 确认 instance_code 正确 |
| `1390010` | 审批任务 ID 不存在 | 先调用 get_instance 获取正确的 task_id |
| `1390018` | 不支持手写签名 | 需在飞书客户端内处理 |

---

## 权限要求

| 操作 | 所需权限 |
|------|---------|
| 查询审批实例 | `approval:approval` 或 `approval:approval:readonly` |
| 同意/拒绝/转交 | `approval:task` |
| 创建评论 | `approval:instance.comment` |
```

- [ ] **Step 3: Commit**

```bash
git add container/skills/feishu-approval/SKILL.md
git commit -m "docs(skill): add feishu-approval skill documentation"
```

---

## Self-Review Checklist

- [x] **Spec coverage**: All 5 operations covered (get_instance, approve, reject, transfer, comment, query)
- [x] **No placeholders**: All code snippets are complete and ready to use
- [x] **Type consistency**: Method names and parameters match across all layers
- [x] **IPC types match MCP tools**: approval_get_instance, approval_approve, etc.
- [x] **Bot identity injection**: approval_comment case handler calls getBotInfo()

---

## Verification Steps

After implementation, verify:

1. **Build succeeds**:
   ```bash
   npm run build
   ```

2. **Container build succeeds**:
   ```bash
   ./container/build.sh
   ```

3. **Manual test** (if Feishu credentials available):
   - Query an approval instance
   - Add a comment to approval instance
   - Check that Bot identity is correctly injected