# 飞书审批操作功能设计

## 需求概述

为 Container Agent 提供飞书审批操作能力：
- 同意审批任务
- 拒绝审批任务
- 转交审批任务
- 创建审批评论
- 查询审批实例详情

## 身份设计

| 操作 | 执行身份 | 说明 |
|-----|-----|-----|
| 同意审批 | Agent 提供审批人 | Agent 先查询实例获取 task_id 和审批人 user_id，再调用 approve |
| 拒绝审批 | Agent 提供审批人 | 同上 |
| 转交审批 | Agent 提供审批人 | Agent 先查询实例获取信息，再调用 transfer |
| 创建评论 | Bot 身份（自动） | Host 自动获取 Bot open_id，Agent 只需传 content |
| 查询实例 | Bot 身份 | 使用 tenant_access_token |

**通过/拒绝流程**（两步调用）：
1. 调用 `approval_get_instance` 获取实例详情
2. 从返回的 `task_list` 中找到当前待处理的审批任务（status=PENDING）
3. 获取该任务的 `id`（task_id）和审批人信息
4. 调用 `approval_approve` 或 `approval_reject`

## 架构设计

遵循现有 MCP + IPC 架构模式：

1. **MCP 工具定义**（`ipc-mcp-stdio.ts`）- 定义工具，写入 IPC 请求，等待结果
2. **Host IPC 处理**（`src/ipc.ts`）- 处理请求，调用 FeishuChannel
3. **FeishuChannel**（`src/channels/feishu.ts`）- 代理方法
4. **FeishuClient**（`src/feishu/client.ts`）- 飞书 API 调用
5. **Container Skill**（`container/skills/feishu-approval/SKILL.md`）- 文档

### 文件变更

| 文件 | 变更 |
|-----|-----|-----|
| `container/agent-runner/src/ipc-mcp-stdio.ts` | 添加 MCP 工具定义 |
| `src/feishu/client.ts` | 添加审批 API 方法和 getBotInfo |
| `src/channels/feishu.ts` | 添加代理方法 |
| `src/ipc.ts` | 添加 IPC case 处理 |
| `container/skills/feishu-approval/SKILL.md` | 新建 Skill 文档 |

## API 方法

### FeishuClient 新增方法

#### 1. getBotInfo()

```
GET /open-apis/bot/v3/info
```

返回 Bot 信息，包括 open_id。

#### 2. getApprovalInstance(instanceCode)

```
GET /open-apis/approval/v4/instances/{instance_code}
```

返回审批实例详情，包括 task_list（审批任务列表）。

#### 3. approveApprovalTask(params)

```
POST /open-apis/approval/v4/tasks/approve
```

参数：
- `approval_code`: 审批定义 Code
- `instance_code`: 审批实例 Code
- `user_id`: 审批人 open_id
- `task_id`: 审批任务 ID
- `comment`?: 审批意见
- `form`?: 表单数据（JSON 字符串）

#### 4. rejectApprovalTask(params)

```
POST /open-apis/approval/v4/tasks/reject
```

参数同 approve。

#### 5. transferApprovalTask(params)

```
POST /open-apis/approval/v4/tasks/transfer
```

参数：
- 同 approve
- `transfer_user_id`: 被转交人 open_id

#### 6. createApprovalComment(params)

```
POST /open-apis/approval/v4/instances/{instance_id}/comments
```

参数：
- `instance_id`: 审批实例 Code
- `user_id`: Bot open_id（自动填充）
- `content`: 评论内容（JSON 字符串格式）
- `parent_comment_id`?: 父评论 ID（回复评论）
- `at_info_list`?: @用户信息列表

返回：`{ comment_id: string }`

## MCP 工具与 Skill 文档的分工

| 层级 | 文件 | 作用 |
|-----|-----|-----|
| **代码实现** | `ipc-mcp-stdio.ts` | 定义工具参数 schema、IPC 通信逻辑、返回结果格式化 |
| **使用文档** | `SKILL.md` | 描述触发条件、使用场景、参数示例、错误处理 |

**类比**：
- MCP 工具定义 = API 实现（定义接口、处理请求）
- Skill 文档 = API 使用手册（告诉 Agent 如何调用）

**Agent 工作流程**：
1. 从 Skill 文档学习何时使用该工具（触发条件）
2. 从 Skill 文档获取参数格式和示例
3. 调用 MCP 工具
4. 根据 Skill 文档的错误说明处理返回结果

在 `ipc-mcp-stdio.ts` 中添加以下工具：

| MCP 工具 | IPC type | 说明 |
|-----|-----|-----|-----|
| `feishu_approval_get_instance` | `approval_get_instance` | 获取实例详情 |
| `feishu_approval_approve` | `approval_approve` | 同意审批 |
| `feishu_approval_reject` | `approval_reject` | 拒绝审批 |
| `feishu_approval_transfer` | `approval_transfer` | 转交审批 |
| `feishu_approval_comment` | `approval_comment` | 创建评论 |
| `feishu_approval_query` | `approval_query` | 查询实例列表 |

### 工具参数

**feishu_approval_get_instance**
- `instance_code`: 审批实例 Code

**feishu_approval_approve**
- `approval_code`: 审批定义 Code
- `instance_code`: 审批实例 Code
- `user_id`: 审批人 open_id
- `task_id`: 审批任务 ID
- `comment`: 审批意见（可选）

**feishu_approval_reject**
- 同 approve

**feishu_approval_transfer**
- 同 approve
- `transfer_user_id`: 被转交人 open_id

**feishu_approval_comment**
- `instance_id`: 审批实例 Code
- `content`: 评论内容（JSON 字符串格式）
- `parent_comment_id`: 父评论 ID（可选，用于回复）

**feishu_approval_query**
- 筛选条件参数（instance_status, user_id 等）

### 评论操作自动注入 Bot 身份

Host 端处理 `approval_comment` 时：
1. 调用 `getBotInfo()` 获取 Bot open_id
2. 将 open_id 作为 user_id 参数传入 API
3. Container 只需传入 instance_id 和 content

## Container Skill

### 文件位置

`container/skills/feishu-approval/SKILL.md`

### 内容结构

1. YAML frontmatter：触发条件
2. 快速索引表：意图 → IPC 工具
3. 各操作详解
4. 使用场景示例（查询 → 获取审批人 → 通过）
5. 错误码说明
6. 权限要求

### 关键说明

- 评论 content 格式：JSON 字符串 `{"text": "评论内容"}`
- 通过/拒绝前需先查询实例获取 task_id 和审批人 user_id
- 审批人判断逻辑：从 task_list 找 status 为 PENDING 的任务
  - 如果有多个 PENDING 任务，Agent 需根据上下文判断（如用户明确指定了审批人）
  - task_list 按审批流程顺序排列，通常第一个 PENDING 任务是当前节点

### approval_code 获取

- 从 `approval_get_instance` 返回结果的 `approval.code` 字段获取
- 或从审批管理后台 URL 中提取

## 权限要求

飞书应用需具备以下权限：
- `approval:approval` 或 `approval:approval:readonly` - 查询审批实例
- `approval:task` - 审批任务操作（同意、拒绝、转交）
- `approval:instance.comment` - 创建评论

## 实现顺序

1. FeishuClient 添加 getBotInfo 和审批 API 方法
2. FeishuChannel 添加代理方法
3. IPC 添加处理逻辑（src/ipc.ts）
4. MCP 工具定义（ipc-mcp-stdio.ts）
5. 创建 Container Skill 文档