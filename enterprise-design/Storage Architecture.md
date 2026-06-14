# Storage Architecture

> 属于 [[Enterprise Agent Platform Overview]] 的子系统设计

## 8. 存储架构与数据流

### 8.1 存储分层

```mermaid
graph TB
    subgraph Storage["存储架构"]
        subgraph Structured["结构化存储"]
            PG[("PostgreSQL<br/>• Agent 元信息<br/>• 用户绑定关系<br/>• 权限/角色<br/>• Consumer Token<br/>• 对话历史<br/>• 推送审计日志<br/>• LLM 用量/成本<br/>• 实例状态")]
        end

        subgraph Unstructured["非结构化存储"]
            MINIO[("MinIO / S3<br/>• Agent 镜像 (Docker Registry)<br/>• 公共层资源文件<br/>  (persona, skills)<br/>• 群组层资源文件<br/>  (group memory, skills)<br/>• 附件/文件")]
        end

        subgraph VersionControl["版本管理"]
            GIT[("Git (Gitea)<br/>• 资源版本历史<br/>• Diff / 回滚<br/>• 推送触发")]
        end

        subgraph Cache["缓存与消息（Phase 2）"]
            REDIS[("Redis<br/>• Message Gateway 消息总线<br/>• 实例注册表<br/>• Pub/Sub 事件通知")]
        end
    end
```

### 8.2 对话历史存储

每个企业的对话历史存在一张独立的 PostgreSQL 表中，按 `agent_id` 和 `group_id` 区分查询。

**表结构：**

```sql
CREATE TABLE agent_{enterprise_id}_messages (
    id            BIGSERIAL PRIMARY KEY,
    agent_id      VARCHAR(64)  NOT NULL,       -- 企业 agent ID
    group_id      VARCHAR(128) NOT NULL,       -- group key: "{platform}:{chat_id}"
    platform      VARCHAR(16)  NOT NULL,       -- dingtalk / feishu / wework
    chat_type     VARCHAR(8)   NOT NULL,       -- group / dm
    role          VARCHAR(16)  NOT NULL,       -- user / assistant / system
    sender_id     VARCHAR(64),                 -- 发送者 user_id（user 角色时）
    sender_name   VARCHAR(128),                -- 发送者 display_name
    content_type  VARCHAR(16)  NOT NULL,       -- text / image / file / ...
    content_text  TEXT,                         -- 文本内容
    content_meta  JSONB,                        -- 附件 URL、文件名等元数据
    token_count   INT,                          -- 该消息的 token 用量
    model         VARCHAR(64),                 -- 使用的 LLM 模型
    created_at    TIMESTAMPTZ  NOT NULL DEFAULT NOW(),

    -- 按时间和群组查询
    CONSTRAINT uq_message_id UNIQUE (id)
);

-- 核心查询索引
CREATE INDEX idx_agent_group_time
    ON agent_{enterprise_id}_messages (agent_id, group_id, created_at);
```

**关键设计：**

- **一企一表**：每个企业一张独立的消息表，天然数据隔离，便于独立备份/归档/清理
- **按 group 隔离**：所有查询通过 `(agent_id, group_id)` 过滤，同一企业不同 group 的对话完全隔离
- **管理平台查询**：管理平台可跨 group 统计分析（按 `agent_id` 聚合），也可查看指定 group 的对话详情
- **写入路径**：Agent 实例在每次对话完成后写入，或由 Message Gateway 统一写入

### 8.3 数据流：消息处理全链路

```mermaid
sequenceDiagram
    participant U as 用户
    participant IM as 钉钉 Bot
    participant MG as Message Gateway
    participant AE as Agent 容器
    participant AIGW as AI Gateway
    participant LLM as 通义千问 API
    participant MCP as 订单系统 MCP

    U->>IM: 发送 "帮我查一下订单 12345"
    IM->>MG: Bot Webhook

    Note over MG: Adapter 归一化为 UniformMessage<br/>agent_id=agent-kefu, group_id=grp-dingtalk-c123

    MG->>MG: Router 查找 group 实例
    Note over MG: 不存在 → 自主创建容器 → 等待就绪

    MG->>AE: 转发消息

    Note over AE: 加载 Layer 0 (公共层, RO): persona, MEMORY.md, skills<br/>加载 Layer 1 (群组层, RW): group MEMORY.md, sessions

    par 并行调用
        AE->>AIGW: LLM 请求 (Consumer Token 鉴权)
        AIGW->>LLM: 路由到 qwen-max
        LLM-->>AIGW: 返回回答
        AIGW-->>AE: LLM 响应
    and
        AE->>AIGW: MCP 请求 (Consumer Token 鉴权)
        AIGW->>MCP: 路由到 order-query MCP
        MCP-->>AIGW: 返回订单信息
        AIGW-->>AE: MCP 响应
    end

    Note over AE: 组合 LLM 回答 + MCP 查询结果<br/>更新群组 MEMORY.md（可选）<br/>保存对话历史到 PostgreSQL

    AE-->>MG: 返回响应
    MG->>IM: Response Handler 路由回钉钉群
    IM-->>U: 用户收到回复
```

### 8.4 数据流：资源推送

```mermaid
sequenceDiagram
    participant Admin as 管理员
    participant UI as Web UI
    participant API as API Server
    participant Git as Git (Gitea)
    participant MinIO as MinIO / S3
    participant GW as Message Gateway
    participant AE as Agent 实例

    Admin->>UI: 修改 persona.md
    UI->>API: 编辑内容，预览 Diff
    API->>Git: git commit: "update persona: add product knowledge"
    Note over Git: 版本 v1.2.0 → v1.2.1
    Git-->>MinIO: Git webhook → 同步到 MinIO 对应路径

    alt 方案 A（容器隔离）
        API->>GW: 通知资源更新
        GW->>AE: 触发实例 reload
    else 方案 B（进程隔离）
        API->>AE: 通知 Agent 进程热重载
    end

    Note over AE: 从 MinIO 重新拉取公共层资源<br/>不中断正在处理的请求（graceful reload）

    Note over AE: 后续消息使用新版本 persona
```

### 8.5 个人 Agent 数据流

```mermaid
graph TB
    subgraph Personal["员工本地 Agent（如 openclaw）"]
        PA[Agent 进程]
        LOCAL[本地文件系统 / 本地 Git]
    end

    PA ---|直连 IM，不经过 Message Gateway| 钉钉飞书企微[钉钉/飞书/企微 Bot]
    PA -->|Consumer Token 鉴权| AIGW[AI Gateway]
    AIGW -->|LLM / MCP| Providers[LLM / MCP Providers]
    PA --> LOCAL
    PA -.->|可选：同步审计| PG[(PostgreSQL<br/>管理平台)]

    subgraph MgmtOnly["管理平台中仅存元信息"]
        M1[agent_id]
        M2[绑定用户]
        M3[Consumer Token]
        M4[监控数据]
        M5["❌ 不管理资源版本"]
    end
```

个人 Agent 在管理平台中只存在**元信息**（agent_id、绑定用户、Consumer Token、监控数据），不存在资源版本管理。

---

## 9. 接口总览

### 9.1 子系统间接口

| 来源 | 目标 | 接口 | 协议 |
|------|------|------|------|
| IM 平台 | Message Gateway | 入站消息 | Webhook / WebSocket |
| Message Gateway | IM 平台 | 出站响应 | IM Platform SDK |
| Message Gateway | Agent 实例 | 转发消息 | Docker exec / IPC / HTTP |
| Agent 实例 | Message Gateway | 返回响应 | HTTP callback / IPC |
| Agent 实例 | AI Gateway | LLM 调用 | OpenAI / Anthropic 兼容 API |
| Agent 实例 | AI Gateway | MCP 调用 | MCP 协议 (SSE) |
| Management Platform | Message Gateway | 配置/策略 | REST API |
| Management Platform | Git (Gitea) | 资源版本管理 | Git 协议 |
| Management Platform | PostgreSQL | 元数据查询 | SQL |
| Management Platform | MinIO | 资源文件 | S3 API |
| Management Platform | Docker Registry | 镜像管理 | Docker Registry API |
| Message Gateway | PostgreSQL | 对话历史 | SQL |
| Agent 实例 | Memory Service | 保存/加载记忆 | REST API |
| 定时任务 | Memory Service | 批量总结记忆 | REST API |
| Memory Service | AI Gateway | 调用 LLM 提取/总结 | OpenAI / Anthropic 兼容 API |
| Memory Service | PostgreSQL | 读取对话历史、存储记忆 | SQL |
| Agent 实例 | PostgreSQL | 对话历史 | SQL |
| Agent 实例 | MinIO | 资源文件 | S3 API |

### 9.2 外部依赖

| 依赖 | 用途 | 备注 |
|------|------|------|
| 钉钉开放平台 | Bot 消息通道 | Webhook |
| 飞书开放平台 | Bot 消息通道 | 事件订阅 |
| 企业微信开放平台 | Bot 消息通道 | 回调 |
| LLM 提供商（通义/文心等） | LLM 推理 | OpenAI/Anthropic 兼容 API |
| OpenAI | LLM 推理 | GPT 模型，OpenAI 协议 |
| Anthropic | LLM 推理 | Claude 模型，Anthropic 协议 |
| MCP Servers | 外部工具服务 | MCP 协议 |
| Docker Registry | Agent 镜像存储 | Docker Registry HTTP API v2 |

## Related

* [[Enterprise Platform Overview]]
* [[Memory Service Design]]
* [[Management Platform Design]]
* [[Message Gateway Design]]

## Tags

#enterprise #storage #postgresql #minio #data-flow
