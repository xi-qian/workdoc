# 飞书消息发送人验证设计

## 背景

当前 NanoClaw 的飞书通道中，任何 IPC 操作（发送消息、创建文档、操作多维表格等）都无法追溯到触发该操作的用户。这意味着：

1. **无法进行权限控制**：无法验证操作者是否有权限访问目标资源
2. **无法审计追踪**：无法记录哪个用户执行了哪个操作
3. **安全隐患**：恶意用户可能通过 agent 执行未授权操作

## 目标

实现端到端的消息 ID 传递机制，确保每个 IPC 操作都能追溯到原始触发消息的发送人，并基于此进行权限验证。

### 非目标

- 不依赖 LLM 进行验证（纯机制保障）
- 不修改现有的 IPC 文件通信协议（仅扩展）

## 需求

| 方面 | 决定 |
|------|------|
| 验证范围 | 所有 IPC 操作 |
| 验证逻辑 | 根据操作类型和目标资源验证发送人权限 |
| 权限来源 | 飞书 API 实时查询 |
| 失败处理 | 拒绝操作并抛出错误给容器 agent |
| 功能开关 | 环境变量 `FEISHU_VERIFY_SENDER=true` |

## 架构概览

```mermaid
sequenceDiagram
    participant FC as Feishu Channel
    participant DB as request_contexts 表
    participant IPC as IPC Input
    participant CTX as current_context.json
    participant AR as agent-runner
    participant MCP as MCP Server
    participant W as IPC Watcher
    participant API as Feishu API

    Note over FC,API: 消息进入阶段
    FC->>DB: 1. 生成 requestId，存储映射<br/>{requestId: {messageId, sender, chatJid}}
    FC->>IPC: 2. 写入消息文件<br/>{type: "message", text, sourceRequestId}
    FC->>CTX: 3. 写入上下文文件<br/>{sourceRequestId, ...}

    Note over AR,MCP: 容器处理阶段
    AR->>IPC: 4. 读取消息文件
    AR->>CTX: 5. 更新 current_context.json
    MCP->>CTX: 6. 读取当前上下文
    MCP->>IPC: 7. 写入 IPC 请求<br/>自动注入 sourceRequestId

    Note over W,API: 权限验证阶段
    W->>IPC: 8. 读取 IPC 请求
    W->>DB: 9. 用 sourceRequestId 查询 sender
    alt 启用验证
        W->>API: 10. 调用飞书 API 验证权限
        alt 权限通过
            W->>W: 执行操作
        else 权限拒绝
            W->>IPC: 写入错误结果
        end
    else 未启用验证
        W->>W: 直接执行操作
    end
```

### 定时任务场景

```mermaid
sequenceDiagram
    participant FC as Feishu Channel
    participant DB as request_contexts 表
    participant TS as scheduled_tasks 表
    participant AR as agent-runner
    participant W as IPC Watcher
    participant API as Feishu API

    Note over FC,TS: 创建任务阶段
    FC->>DB: 用户消息触发，生成 requestId
    AR->>W: schedule_task IPC 请求<br/>带 sourceRequestId
    W->>DB: 查询创建者信息
    W->>TS: 存储任务 + created_by_* 字段

    Note over AR,API: 执行任务阶段（无用户消息）
    AR->>AR: 定时触发任务执行
    AR->>DB: 创建临时 request_context<br/>关联到 created_by_sender_id
    AR->>AR: 执行容器，带 tempRequestId
    AR->>W: IPC 请求带 tempRequestId
    W->>DB: 查询 sender（获取创建者身份）
    W->>API: 验证创建者权限
    AR->>DB: 清理临时 request_context
```

## 数据模型

### 新增数据库表：request_contexts

```sql
CREATE TABLE request_contexts (
  request_id TEXT PRIMARY KEY,           -- UUID 格式
  message_id TEXT NOT NULL,              -- 飞书原始消息 ID
  chat_jid TEXT NOT NULL,                -- 聊天 JID（feishu:oc_xxx）
  sender_open_id TEXT NOT NULL,          -- 发送人 open_id
  sender_name TEXT,                      -- 发送人名称（缓存）
  trigger_message TEXT,                  -- 触发消息内容（审计用）
  created_at TEXT NOT NULL,              -- 创建时间
  expires_at TEXT NOT NULL               -- 过期时间（默认 24 小时）
);

CREATE INDEX idx_request_contexts_created ON request_contexts(created_at);
CREATE INDEX idx_request_contexts_expires ON request_contexts(expires_at);
```

### 修改现有表：scheduled_tasks

添加创建者信息字段，用于定时任务执行时的权限验证：

```sql
-- 新增字段
ALTER TABLE scheduled_tasks ADD COLUMN created_by_sender_id TEXT;
ALTER TABLE scheduled_tasks ADD COLUMN created_by_sender_name TEXT;
ALTER TABLE scheduled_tasks ADD COLUMN created_by_request_id TEXT;
```

**字段说明：**
- `created_by_sender_id`：创建者的 open_id，用于权限验证
- `created_by_sender_name`：创建者名称，用于日志和错误消息
- `created_by_request_id`：创建时的 requestId，用于审计追溯

### 新增类型定义

```typescript
// src/types.ts

export interface RequestContext {
  requestId: string;           // UUID
  messageId: string;           // 飞书消息 ID
  chatJid: string;             // 聊天 JID
  senderOpenId: string;        // 发送人 open_id
  senderName?: string;         // 发送人名称
  triggerMessage?: string;     // 触发消息内容
  createdAt: string;           // ISO 时间戳
  expiresAt: string;           // ISO 时间戳
}

export interface IpcRequestWithSource {
  // ... existing IPC fields
  sourceRequestId?: string;    // 来源请求 ID
}

export interface CurrentContext {
  sourceRequestId: string;      // 来源请求 ID（从 request_contexts 表）
  messageId?: string;           // 飞书消息 ID（从 request_contexts 表）
  senderOpenId?: string;        // 发送人 open_id（从 request_contexts 表）
  senderName?: string;          // 发送人名称（从 request_contexts 表）
  chatJid: string;              // 聊天 JID（从 container-runner 传入）
  groupFolder: string;          // 群组文件夹名（从 container-runner 传入，非 request_contexts 表）
  timestamp: string;            // 当前时间戳
}
```

**数据来源说明：**
- `sourceRequestId`、`messageId`、`senderOpenId`、`senderName` 来自 `request_contexts` 表
- `chatJid`、`groupFolder` 来自容器启动时的 `ContainerInput` 或后续 IPC input 消息
- `timestamp` 在写入 `current_context.json` 时生成
```

## 组件修改

### 1. 数据库层 (src/db.ts)

新增函数：

```typescript
// 创建请求上下文
export function createRequestContext(ctx: RequestContext): void;

// 获取请求上下文
export function getRequestContext(requestId: string): RequestContext | undefined;

// 删除请求上下文
export function deleteRequestContext(requestId: string): void;

// 清理过期上下文
export function cleanupExpiredRequestContexts(): number;
```

**修改现有函数：**

```typescript
// createTask 需要支持新字段
export function createTask(task: {
  // ... existing fields ...
  created_by_sender_id?: string;
  created_by_sender_name?: string;
  created_by_request_id?: string;
}): void;

// getTaskById 返回值需要包含新字段
export interface ScheduledTask {
  // ... existing fields ...
  created_by_sender_id?: string;
  created_by_sender_name?: string;
  created_by_request_id?: string;
}
```

### 2. 飞书通道 (src/channels/feishu.ts)

在 `handleMessageEvent` 中，生成 requestId 并存储：

```typescript
private async handleMessageEvent(event: FeishuEvent): Promise<void> {
  // ... existing parsing logic ...

  const requestId = crypto.randomUUID();
  const now = new Date();
  const expiresAt = new Date(now.getTime() + 24 * 60 * 60 * 1000); // 24 hours

  createRequestContext({
    requestId,
    messageId: msg.message_id,
    chatJid: jid,
    senderOpenId,
    senderName,
    triggerMessage: text.slice(0, 500), // 截断存储
    createdAt: now.toISOString(),
    expiresAt: expiresAt.toISOString(),
  });

  // 将 requestId 传递给 onMessage 回调
  const newMessage: NewMessage = {
    // ... existing fields ...
    requestId, // 新增字段
  };

  this.onMessage(jid, newMessage);
}
```

### 3. 主消息处理 (src/index.ts)

修改 `NewMessage` 类型和消息处理逻辑：

```typescript
// 在 formatMessages 时携带 requestId
// 在 runAgent 时传递 sourceRequestId
```

### 4. GroupQueue (src/group-queue.ts)

修改 `sendMessage` 方法，携带 sourceRequestId：

```typescript
sendMessage(groupJid: string, text: string, sourceRequestId?: string): boolean {
  // ... existing logic ...

  const payload: IpcInputMessage = {
    type: 'message',
    text,
    sourceRequestId, // 新增字段
  };

  fs.writeFileSync(tempPath, JSON.stringify(payload));
  // ...
}
```

### 5. 容器运行器 (src/container-runner.ts)

在 `ContainerInput` 中添加 `sourceRequestId`：

```typescript
export interface ContainerInput {
  // ... existing fields ...
  sourceRequestId?: string;  // 触发消息的 request ID
}
```

同时更新 `current_context.json` 的写入逻辑。

### 6. Agent Runner (container/agent-runner/src/index.ts)

处理 IPC 消息时，更新 `current_context.json`：

```typescript
function drainIpcInput(): { messages: string[]; sourceRequestId?: string } {
  // ... existing logic ...

  let sourceRequestId: string | undefined;
  for (const file of files) {
    const data = JSON.parse(content);
    if (data.sourceRequestId) {
      sourceRequestId = data.sourceRequestId;
    }
    if (data.type === 'message' && data.text) {
      messages.push(data.text);
    }
  }

  // 更新 current_context.json
  if (sourceRequestId) {
    updateCurrentContext({ sourceRequestId, ... });
  }

  return { messages, sourceRequestId };
}
```

### 7. MCP Server (container/agent-runner/src/ipc-mcp-stdio.ts)

在每个工具调用中注入 sourceRequestId：

```typescript
function getCurrentContext(): CurrentContext | null {
  const contextPath = '/workspace/ipc/current_context.json';
  try {
    if (fs.existsSync(contextPath)) {
      return JSON.parse(fs.readFileSync(contextPath, 'utf-8'));
    }
  } catch {}
  return null;
}

// 在每个工具中使用
server.tool('feishu_fetch_doc', ..., async (args) => {
  const context = getCurrentContext();
  const requestId = writeIpcFile(FEISHU_REQUESTS_DIR, {
    type: 'fetch_doc',
    doc_id: args.doc_id,
    sourceRequestId: context?.sourceRequestId,  // 注入
    // ... other fields ...
  });
  // ...
});

// schedule_task 同样需要注入 sourceRequestId
server.tool('schedule_task', ..., async (args) => {
  const context = getCurrentContext();
  const data = {
    type: 'schedule_task',
    taskId,
    prompt: args.prompt,
    schedule_type: args.schedule_type,
    schedule_value: args.schedule_value,
    context_mode: args.context_mode || 'group',
    targetJid,
    sourceRequestId: context?.sourceRequestId,  // 注入，用于存储创建者信息
    // ... other fields ...
  };
  writeIpcFile(TASKS_DIR, data);
});
```

**注意：** `sourceRequestId` 的注入是程序行为，由 MCP Server 自动完成。Agent 不需要知道这个字段的存在，也不需要在调用工具时提供。

### 8. IPC Watcher (src/ipc.ts)

添加权限验证逻辑：

```typescript
async function verifySenderPermission(
  request: IpcRequestWithSource,
  feishuChannel: FeishuChannel,
): Promise<{ authorized: boolean; reason?: string }> {
  // 功能关闭时直接通过
  if (process.env.FEISHU_VERIFY_SENDER !== 'true') {
    return { authorized: true };
  }

  // 没有 sourceRequestId 时，允许通过（支持其他通道）
  // 注意：启用此功能时，飞书消息触发的 IPC 会验证权限，
  // 但其他通道（WhatsApp、Telegram）的消息不会有 sourceRequestId，
  // 因此它们的 IPC 请求会直接通过。
  if (!request.sourceRequestId) {
    return { authorized: true };
  }

  // 查询请求上下文
  const ctx = getRequestContext(request.sourceRequestId);
  if (!ctx) {
    return { authorized: false, reason: 'Request context not found or expired' };
  }

  // 根据操作类型验证权限
  switch (request.type) {
    case 'send_message':
    case 'send_card':
    case 'send_file':
      return await verifyChatAccess(feishuChannel, ctx.senderOpenId, request.chat_id);

    case 'fetch_doc':
      return await verifyDocAccess(feishuChannel, ctx.senderOpenId, request.doc_id, 'read');

    case 'update_doc':
      return await verifyDocAccess(feishuChannel, ctx.senderOpenId, request.doc_id, 'edit');

    case 'create_doc':
      // 创建文档需要验证目标文件夹权限
      if (request.folder_token) {
        return await verifyFolderAccess(feishuChannel, ctx.senderOpenId, request.folder_token, 'edit');
      }
      return { authorized: true };

    // ... other cases ...

    default:
      return { authorized: true }; // 未知类型默认通过
  }
}
```

### 8.1 定时任务创建 (src/ipc.ts processTaskIpc)

在处理 `schedule_task` IPC 请求时，存储创建者信息：

```typescript
case 'schedule_task':
  // ... existing validation ...

  // 获取创建者信息（程序自动获取，不需要 agent 提供）
  let createdBySenderId: string | undefined;
  let createdBySenderName: string | undefined;
  let createdByRequestId: string | undefined;

  if (data.sourceRequestId) {
    const ctx = getRequestContext(data.sourceRequestId);
    if (ctx) {
      createdBySenderId = ctx.senderOpenId;
      createdBySenderName = ctx.senderName;
      createdByRequestId = data.sourceRequestId;
    }
  }

  createTask({
    id: taskId,
    group_folder: targetFolder,
    chat_jid: targetJid,
    prompt: data.prompt,
    schedule_type: scheduleType,
    schedule_value: data.schedule_value,
    context_mode: contextMode,
    next_run: nextRun,
    status: 'active',
    created_at: new Date().toISOString(),
    // 新增字段
    created_by_sender_id: createdBySenderId,
    created_by_sender_name: createdBySenderName,
    created_by_request_id: createdByRequestId,
  });
```

### 8.2 定时任务执行 (src/task-scheduler.ts)

任务执行时，创建临时 request_context 用于权限验证：

```typescript
async function executeTask(task: ScheduledTask): Promise<void> {
  // 如果任务有创建者信息，创建临时 request_context
  let tempRequestId: string | undefined;

  if (task.created_by_sender_id && FEISHU_VERIFY_SENDER === 'true') {
    tempRequestId = `task-${task.id}-${Date.now()}`;
    const now = new Date();
    const expiresAt = new Date(now.getTime() + 60 * 60 * 1000); // 1 小时过期

    createRequestContext({
      requestId: tempRequestId,
      messageId: `scheduled-task:${task.id}`,
      chatJid: task.chat_jid,
      senderOpenId: task.created_by_sender_id,
      senderName: task.created_by_sender_name,
      triggerMessage: `[Scheduled Task] ${task.prompt.slice(0, 200)}`,
      createdAt: now.toISOString(),
      expiresAt: expiresAt.toISOString(),
    });
  }

  try {
    // 执行任务，传递 tempRequestId 作为 sourceRequestId
    await runContainerAgent(group, {
      prompt: task.prompt,
      sessionId: undefined, // 新 session
      groupFolder: task.group_folder,
      chatJid: task.chat_jid,
      isMain: false,
      isScheduledTask: true,
      sourceRequestId: tempRequestId,
    });
  } finally {
    // 清理临时 request_context
    if (tempRequestId) {
      deleteRequestContext(tempRequestId);
    }
  }
}
```

**流程总结：**
```
用户发送消息 → 创建 requestId
     ↓
Agent 调用 schedule_task
     ↓
MCP Server 自动注入 sourceRequestId（程序行为）
     ↓
IPC watcher 处理，查询 request_contexts 获取创建者信息
     ↓
存入 scheduled_tasks 表（created_by_* 字段）
     ↓
任务执行时，创建临时 request_context 关联到创建者
     ↓
权限验证使用创建者身份
```

### 9. 飞书客户端 (src/feishu/client.ts)

新增权限验证方法：

```typescript
async function verifyChatAccess(
  client: FeishuClient,
  senderOpenId: string,
  chatId: string,
): Promise<{ authorized: boolean; reason?: string }> {
  // 调用飞书 API 检查用户是否在群组中
  // GET /im/v1/chats/:chatId/members?member_id_type=open_id
  // 检查 senderOpenId 是否在成员列表中
}

async function verifyDocAccess(
  client: FeishuClient,
  senderOpenId: string,
  docId: string,
  permission: 'read' | 'edit',
): Promise<{ authorized: boolean; reason?: string }> {
  // 调用飞书 API 检查文档权限
  // 使用 GET /drive/v1/permissions/:token/members 获取权限成员列表
  // 检查 senderOpenId 是否在成员列表中且有对应权限
  // 注意：不能使用 docx/v1/documents/:documentId，因为那只验证 bot 的访问权限
}

async function verifyFolderAccess(
  client: FeishuClient,
  senderOpenId: string,
  folderToken: string,
  permission: 'read' | 'edit',
): Promise<{ authorized: boolean; reason?: string }> {
  // 调用飞书 API 检查文件夹权限
  // GET /drive/v1/permissions/:folderToken/members
  // 检查 senderOpenId 是否有对应权限
}
```

## 环境变量

| 变量名 | 默认值 | 说明 |
|--------|--------|------|
| `FEISHU_VERIFY_SENDER` | `false` | 是否启用发送人权限验证 |
| `REQUEST_CONTEXT_TTL_HOURS` | `24` | 请求上下文过期时间（小时） |

**注意：** 当 `FEISHU_VERIFY_SENDER=true` 时，只有飞书消息触发的 IPC 操作会被验证权限。其他通道（WhatsApp、Telegram 等）的消息没有 `sourceRequestId`，因此它们的 IPC 请求会直接通过，不受此功能影响。

## 错误处理

当权限验证失败时，IPC 处理器返回错误：

```json
{
  "success": false,
  "error": "Permission denied: User ou_xxx does not have access to chat oc_xxx",
  "errorType": "PERMISSION_DENIED",
  "sourceRequestId": "req_xxx"
}
```

容器内的 agent 可以根据 `errorType` 决定如何处理（重试、通知用户等）。

## 安全考虑

1. **requestId 不可伪造**：在 Host 端生成，容器无法修改
2. **时效性**：requestContext 有过期时间，防止历史请求被滥用
3. **最小权限**：每个操作类型独立验证权限
4. **审计日志**：记录所有验证失败的请求

## 测试计划

1. **单元测试**
   - request_contexts 表 CRUD 操作
   - 权限验证函数
   - context 文件读写

2. **集成测试**
   - 端到端消息流：飞书消息 → IPC 请求 → 验证 → 执行
   - 权限拒绝场景
   - 过期上下文清理

3. **手动测试**
   - 启用/禁用功能开关
   - 不同操作类型的权限验证
   - 长时间运行后上下文清理

## 实施步骤

1. **Phase 1: 基础设施**
   - 添加 request_contexts 表
   - 添加类型定义
   - 实现 DB CRUD 函数

2. **Phase 2: 上下文传递**
   - 修改 Feishu Channel 生成 requestId
   - 修改消息处理流程传递 requestId
   - 实现 current_context.json 读写

3. **Phase 3: MCP 集成**
   - 修改 MCP Server 读取并注入 sourceRequestId
   - 所有 IPC 请求携带 sourceRequestId

4. **Phase 4: 权限验证**
   - 实现飞书 API 权限查询
   - 在 IPC Watcher 中集成验证逻辑
   - 添加环境变量开关

5. **Phase 5: 清理与优化**
   - 实现过期上下文清理
   - 添加审计日志
   - 性能优化

## 后续扩展

1. **细粒度权限**：支持更复杂的权限规则（如：只能读取，不能写入）
2. **权限缓存**：减少飞书 API 调用频率
3. **审计面板**：提供 Web UI 查看权限验证历史
4. **其他通道**：将类似机制扩展到 WhatsApp、Telegram 等通道