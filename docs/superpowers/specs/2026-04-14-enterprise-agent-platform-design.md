# 企业 Agent 平台 — 全局架构设计

> 日期：2026-04-14
> 状态：草稿
> 范围：覆盖所有子系统的顶层架构及接口关系，不深入实现细节

---

## 1. 概述

### 1.1 平台目标

构建一个企业级 Agent 平台，支持两类 Agent：

- **个人 Agent**：与员工身份绑定，作为私人助理（类似 openclaw）。不需要 group 隔离，直连 IM 平台，共享 AI Gateway。
- **企业 Agent**：独立的数字员工，可响应多个用户的私聊请求，也可在多个群中响应用户请求。需要 group 级别的数据隔离。

### 1.2 核心需求

| 需求 | 说明 |
|------|------|
| 多 SDK 混合 | 平台层抽象统一接口，不同 Agent 可使用不同 SDK |
| IM 平台 | 钉钉、飞书、企业微信 |
| 部署方式 | Docker 先行，预留 K8s 演进路径 |
| 管理平台 | Web UI 管理后台 |
| LLM 支持 | 国产模型（通义/文心等）、OpenAI GPT、Anthropic Claude、OpenAI/Anthropic 兼容协议 |
| MCP 协议 | Agent 通过 MCP 协议调用外部工具服务 |
| 存储 | 混合方案：PostgreSQL（对话历史/元数据）+ MinIO/S3（资源文件） |
| 版本管理 | 基于 Git 的资源版本管理（仅企业 Agent） |
| 个人 Agent | 无 group 隔离，平台不管理其资源版本，用户自行管理 |

### 1.3 参考项目

| 项目 | 参考内容 |
|------|---------|
| NanoClaw | 容器级 group 隔离、凭证代理（AI Gateway）、Claude Agent SDK |
| HiClaw | 消息网关（Matrix/Tuwunel）、AI 网关（Higress + Consumer Token）、Manager-Worker 架构 |
| Hermes | Profile 进程隔离、group 隔离设计（ContextVar vs per-group 进程） |

---

## 2. 系统总体架构

### 2.1 系统全景

```mermaid
flowchart TD
    DT[钉钉]
    FS[飞书]
    WW[企业微信]

    GW_IN[Platform Adapters] --> UMF[统一消息格式] --> ROUTER[Router<br/>按 group_id 路由]
    ROUTER --> IM_MGR[Instance Manager<br/>自治创建/回收]
    IM_MGR --> RESP[Response Handler]

    PA[个人 Agent]
    EA1[Group A 实例]
    EA2[Group B 实例]
    EAN[Group N 实例]

    IN2[入口] --> AUTH[Consumer Token 鉴权]
    AUTH --> LLM_R[LLM Router]
    AUTH --> MCP_R[MCP Router]
    LLM_R --> QUOTA[Quota &amp; Audit]
    MCP_R --> QUOTA

    LLM1[国产模型]
    LLM2[OpenAI]
    MCPS[MCP Servers]

    MgmtAPI[REST API Server]
    MgmtGit[Git（Gitea）]
    PG[(PostgreSQL)]
    MINIO[(MinIO / S3)]

    DT -->|Webhook| GW_IN
    FS -->|Event| GW_IN
    WW -->|Callback| GW_IN
    PA ---|直连| DT
    PA ---|直连| FS
    PA ---|直连| WW
    IM_MGR -.-> EA1 & EA2 & EAN
    RESP -.- DT & FS & WW
    PA & EA1 & EA2 & EAN -->|Consumer Token| IN2
    LLM_R --> LLM1 & LLM2
    MCP_R --> MCPS

    MgmtAPI --> PG & MINIO
    MgmtGit -->|资源版本管理| MINIO
    MgmtAPI -.->|配置策略 / 监控| ROUTER
    MgmtAPI -.->|配置策略 / 监控| IN2

    subgraph layer1["① 消息层：IM 平台"]
        direction LR
        DT & FS & WW
    end
    subgraph layer2["② Agent 运行层"]
        subgraph pa_box["个人 Agent"]
            direction LR
            PA
        end
        subgraph ea_box["企业 Agent = Message Gateway + Group 实例"]
            direction LR
            GW_IN & UMF & ROUTER & IM_MGR & RESP & EA1 & EA2 & EAN
        end
    end
    subgraph layer3["③ AI 网关"]
        direction TB
        IN2 & AUTH & LLM_R & MCP_R & QUOTA
    end
    subgraph layer4["④ 管理平台：Management Platform"]
        direction LR
        MgmtAPI & MgmtGit & PG & MINIO
    end
    subgraph layer5["⑤ 外部服务"]
        direction LR
        LLM1 & LLM2 & MCPS
    end

    style pa_box fill:#fef3c7,stroke:#f59e0b,stroke-width:2px,color:#78350f
    style ea_box fill:#dbeafe,stroke:#3b82f6,stroke-width:2px,color:#1e3a5f
```

### 2.2 子系统职责

| 子系统 | 职责 | 对外接口 |
|--------|------|---------|
| **Management Platform** | Agent 生命周期管理、企业 Agent 资源版本管理与推送、镜像管理、监控 | REST API（管理前端 + Message Gateway 调用）、Git 协议 |
| **Message Gateway** | 每个企业 Agent 一个实例，持有 IM Bot SDK，接收消息并路由到对应 group 实例 | IM Platform API（入站）、Agent IPC（出站） |
| **Agent Runtime** | 运行 agent 实例，加载人设/记忆/skill，调用 LLM/MCP | Agent SDK API（Gateway 入站）、AI Gateway API（出站） |
| **AI Gateway** | LLM/MCP 调用的集中鉴权、路由、限流、计量 | OpenAI/Anthropic 兼容 API（给 Agent）、MCP Proxy |
| **Storage** | PostgreSQL（对话历史/元数据）、MinIO/S3（资源/附件）、Git（版本管理） | SQL / S3 API / Git 协议 |

### 2.3 Agent 类型对比

| | 个人 Agent | 企业 Agent |
|---|---|---|
| 绑定对象 | 1 个员工 | 1 个企业（数字员工身份） |
| 消息接入 | 直连 IM 平台 | 通过 Message Gateway |
| Group 隔离 | 不需要 | 需要（多人多群） |
| 共享层 | 无（全部私有） | 公共人设/记忆/skill（管理员维护，只读） |
| 私有层 | 全部（用户自行管理） | per-group：session/记忆/skill（可读写） |
| 运行实例 | 1 个容器/进程 | 多个 group 实例 |
| 资源版本管理 | 用户本地自行管理 | 通过管理平台（Git） |
| 管理平台中的角色 | 仅元信息（agent_id、用户绑定、监控、Consumer Token） | 完整管理（元信息 + 资源 + 镜像 + 监控） |

---

## 3. 企业 Agent 的 Group 隔离设计

### 3.1 数据分层

每个企业 Agent 的数据分为两层：

```mermaid
graph TB
    subgraph EnterpriseAgent["企业 Agent：客服小助手"]
        subgraph L0["Layer 0：公共层（Enterprise Shared）"]
            P0[人设/Persona<br/>只读，管理员维护]
            M0[公共记忆/Memory<br/>只读，管理员维护]
            S0[公共 Skill<br/>只读，管理员维护]
            C0[配置/model settings 等]
        end

        subgraph L1["Layer 1：群组层（per-Group Private）"]
            direction LR
            G1A["Group A<br/>钉钉群 c123"]
            G1B["Group B<br/>飞书群 g456"]
            G1N["Group N<br/>1:1 私聊 user789"]
        end
    end

    subgraph GroupData["每个 Group 包含"]
        SES[对话历史/Session<br/>可读写]
        GMEM[群组记忆/Memory<br/>可读写]
        GSK[私有 Skill<br/>可读写，仅本 group 生效]
    end

    L0 -.->|只读基础| G1A
    L0 -.->|只读基础| G1B
    L0 -.->|只读基础| G1N
    G1A --> GroupData
```

### 3.2 方案 A：容器级隔离

每个 group 运行在独立容器中。公共层以只读方式挂载，群组层以读写方式挂载。

```mermaid
graph TB
    subgraph Agent["企业 Agent：客服小助手"]
        subgraph C1["Container: grp-dingtalk-c123"]
            direction TB
            SH1["/shared/  (RO, bind mount)<br/>├ persona.md<br/>├ MEMORY.md<br/>└ skills/"]
            D1["/data/  (RW, bind mount)<br/>├ sessions/<br/>├ MEMORY.md<br/>└ skills/"]
            SDK1["Agent SDK 进程"]
            SH1 --> SDK1
            D1 --> SDK1
        end

        subgraph C2["Container: grp-feishu-g456"]
            direction TB
            SH2["/shared/  (RO)"]
            D2["/data/  (RW)"]
            SDK2["Agent SDK 进程"]
            SH2 --> SDK2
            D2 --> SDK2
        end

        subgraph C3["Container: grp-dm-user-789<br/>1:1 私聊"]
            direction TB
            SH3["/shared/  (RO)"]
            D3["/data/  (RW)"]
            SDK3["Agent SDK 进程"]
            SH3 --> SDK3
            D3 --> SDK3
        end
    end

    HostMinio["MinIO: agents/agent-kefu/shared/"] -.->|RO bind mount| SH1
    HostMinio -.-> SH2
    HostMinio -.-> SH3

    HostG1["宿主机: agents/agent-kefu/groups/grp-dingtalk-c123/"] -.->|RW bind mount| D1
    HostG2["宿主机: agents/agent-kefu/groups/grp-feishu-g456/"] -.-> D2
    HostG3["宿主机: agents/agent-kefu/groups/grp-dm-user-789/"] -.-> D3
```

**容器路径映射：**

| 容器内路径 | 宿主机路径 | 读写 |
|-----------|-----------|------|
| `/shared/` | `agents/{agent_id}/shared/`（MinIO 同步） | RO |
| `/data/` | `agents/{agent_id}/groups/{group_id}/` | RW |

**生命周期：**

```mermaid
stateDiagram-v2
    [*] --> 首次消息: Message Gateway 自动创建容器
    首次消息 --> 运行中: 容器启动完成
    运行中 --> 空闲等待: idle timeout
    空闲等待 --> 运行中: 收到新消息
    空闲等待 --> 已停止: 资源压力 / preempt
    运行中 --> 已停止: 资源压力 / preempt
    已停止 --> 运行中: 收到新消息，重启容器
    已停止 --> [*]
```

**优势：**
- OS 级隔离（最强）
- 多 SDK 天然支持（容器完全隔离）
- K8s 演进路径清晰（容器直接映射为 Pod）
- 个人 Agent 可复用同一模型（1 容器 = 1 group）

**劣势：**
- 资源开销高（每个 group 一个容器）
- 冷启动秒级
- 不支持热重载（需重建容器）

### 3.3 方案 B：进程级隔离

每个企业 Agent 对应一个或多个进程，通过进程级别的 `HERMES_HOME` 环境变量或 ContextVar 实现 group 隔离。

#### B1：多进程（每 group 一个进程）

```mermaid
graph TB
    subgraph Agent["企业 Agent: 客服小助手"]
        subgraph P1["进程 1<br/>HERMES_HOME=grp-dingtalk-c123"]
            SDK1["Agent SDK 实例"]
            DATA1["进程私有数据<br/>session / memory / skills"]
        end

        subgraph P2["进程 2<br/>HERMES_HOME=grp-feishu-g456"]
            SDK2["Agent SDK 实例"]
            DATA2["进程私有数据<br/>session / memory / skills"]
        end

        subgraph PN["进程 N<br/>HERMES_HOME=grp-dm-user-789"]
            SDKN["Agent SDK 实例"]
            DATAN["进程私有数据<br/>session / memory / skills"]
        end
    end

    MSG[Message Gateway] -->|按 group_id 路由| P1
    MSG --> P2
    MSG --> PN
```

- 每个 group 一个独立进程，`HERMES_HOME` 指向各自的 group 数据目录
- 进程级隔离，crash 不影响其他 group
- 参考 Hermes 的 profile 隔离策略
- Message Gateway 按 group_id 路由到对应进程

#### B2：单进程多实例（进程内逻辑隔离）

```mermaid
graph TB
    subgraph Process["企业 Agent 进程"]
        CV[ContextVar: current_group_id]
        POOL["SDK 实例池（per-group）"]

        subgraph I1["instance: grp-dingtalk-c123"]
            CACHE1["公共层: 内存缓存<br/>persona, skills"]
            PRIV1["私有层: 从 DB/MinIO 加载"]
        end

        subgraph I2["instance: grp-feishu-g456"]
            CACHE2["公共层: 内存缓存"]
            PRIV2["私有层: 从 DB/MinIO 加载"]
        end

        subgraph IN["instance: grp-dm-user-789"]
            CACHEN["公共层: 内存缓存"]
            PRIVN["私有层: 从 DB/MinIO 加载"]
        end
    end

    MSG[Message Gateway] -->|路由到匹配实例| POOL
    POOL --> I1
    POOL --> I2
    POOL --> IN
```

- 一个进程内维护多个 SDK 实例，通过 ContextVar 切换当前 group 上下文
- 实例长期驻留，数据常驻内存
- 同进程内 crash 互相影响

### 3.4 方案对比

| | A: 容器隔离 | B1: 多进程 | B2: 单进程多实例 |
|---|---|---|---|
| 隔离强度 | OS 级 | 进程级 | 进程内逻辑隔离 |
| 内存开销 | 高（每 group 一个容器） | 中（每 group 一个进程） | 低（多实例共享进程） |
| 冷启动 | 秒级 | 毫秒级 | 无 |
| 多机扩展 | 容器编排天然支持 | 需 sharding + 消息队列 | 需 sharding + 消息队列 |
| 故障隔离 | 最强 | 强（进程 crash 不互相影响） | 弱（同进程 crash 连带） |

### 3.5 选型策略

三种方案不互斥，可按实际需求选择：

| 需求 | 推荐方案 |
|------|---------|
| 安全性要求高，资源充足 | 方案 A：容器隔离 |
| 需要进程级故障隔离，资源适中 | 方案 B1：多进程 |
| group 数量多，资源有限 | 方案 B2：单进程多实例 |

---

## 4. Message Gateway

### 4.1 定位

**每个企业 Agent 部署一个 Message Gateway 实例**，它是企业 Agent 的组成部分（而非独立的共享组件）。个人 Agent 直连 IM 平台，不经过 Message Gateway。

一个企业 Agent 的完整部署单元：

```
企业 Agent = Message Gateway 实例 + Group 实例（0..N）
                │
                ├── 钉钉 Bot SDK 实例（对应钉钉上的一个机器人）
                ├── 飞书 Bot SDK 实例（对应飞书上的一个机器人）
                └── 企微 Bot SDK 实例（对应企微上的一个机器人）
```

### 4.2 架构图

```mermaid
graph TB
    subgraph IM["IM 平台"]
        DT[钉钉]
        FS[飞书]
        WW[企业微信]
    end

    subgraph EA["企业 Agent：客服小助手"]
        subgraph GW["Message Gateway 实例"]
            DT_SDK[钉钉 Bot SDK<br/>app_key: xxx]
            FS_SDK[飞书 Bot SDK<br/>app_id: yyy]
            UMF[统一消息格式<br/>UniformMessage]
            ROUTER[Router<br/>按 group_id 路由]
            IM_MGR[Instance Manager<br/>自治生命周期管理]
            RESP[Response Handler]

            DT_SDK --> UMF
            FS_SDK --> UMF
            UMF --> ROUTER --> IM_MGR
            IM_MGR --> RESP
        end

        subgraph GROUPS["Group 实例"]
            EA1[Group A<br/>钉钉群 c123]
            EA2[Group B<br/>飞书群 g456]
            EAN[Group N<br/>1:1 私聊]
        end
    end

    DT -->|Webhook| DT_SDK
    FS -->|Event Subscription| FS_SDK

    IM_MGR -.->|转发消息| EA1 & EA2 & EAN
    EA1 & EA2 & EAN -.->|返回响应| RESP
    RESP --> DT
    RESP --> FS
```

### 4.3 统一消息格式

各 IM 平台消息被 Gateway 归一化为统一格式：

```typescript
interface UniformMessage {
  id: string;                    // 消息唯一 ID
  group_id: string;              // group key: "{platform}:{chat_id}"
  platform: "dingtalk" | "feishu" | "wework";
  chat_type: "group" | "dm";    // 群聊或 1:1
  sender: {
    user_id: string;
    display_name: string;
  };
  content: {
    type: "text" | "file" | "image" | ...;
    text?: string;
    file_url?: string;
  };
  timestamp: number;
  reply_to?: string;             // 引用回复的消息 ID
}
```

> 注意：`agent_id` 不再需要，因为每个 Gateway 实例天然属于一个企业 Agent。

### 4.4 路由与实例管理

Gateway **自主管理** group 实例的生命周期，Management Platform 也可手动创建/销毁实例。

```mermaid
flowchart TD
    MSG[收到消息] --> PARSE[解析 platform + chat_id → group_id]

    PARSE --> EXISTS{group 实例存在且健康？}
    EXISTS -->|是| FORWARD[直接转发消息]

    EXISTS -->|否| CREATE[创建实例<br/>加载公共层 + 群组层数据<br/>等待就绪 → 转发]

    PARSE --> RESOURCE{资源充足？}
    RESOURCE -->|否| PREEMPT[Preempt 机制<br/>选择 idle 最长的实例<br/>优雅停止 → 释放资源<br/>创建新实例]
```

### 4.5 实例生命周期

```mermaid
stateDiagram-v2
    [*] --> Running: 首次消息 / 手动创建
    Running --> Running: 正常处理消息
    Running --> Idle: idle timeout
    Idle --> Running: 收到新消息，重启
    Idle --> Preempted: 资源压力，抢占回收
    Running --> Preempted: 资源压力，抢占回收
    Running --> Stopped: 手动停止
    Preempted --> Stopped: 优雅停止完成
    Idle --> Stopped: 优雅停止
    Stopped --> Running: 收到新消息 / 手动启动
    Stopped --> [*]
```

**管理策略（通过 Management Platform 配置）：**
- `max_instances`：每台机器最大实例数
- `idle_timeout`：空闲多久后停止
- `preempt_strategy`：LRU / 优先级
- `health_check`：定期探测实例健康状态

### 4.6 演进路径

| 阶段 | 部署方式 | 路由方式 |
|------|---------|---------|
| **Phase 1：单机** | Gateway + Group 实例在同一台机器 | Docker network / localhost |
| **Phase 2：多机** | Gateway 和 Group 实例分布在多台机器 | Redis Pub/Sub / NATS 消息总线 |
| **Phase 3：弹性** | K8s 集群 | K8s Service + 自定义 Operator |

### 4.7 Management Platform 的角色

Management Platform **支持手动创建/销毁实例**，同时 Gateway 也可自治管理实例生命周期，其职责为：

- 管理 Agent 镜像（构建/推送/版本）
- 管理企业 Agent 公共层资源（人设/记忆/skill 版本管理与推送）
- 手动创建/销毁/重启实例
- 配置实例管理策略（max_instances、idle_timeout 等）
- 监控实例状态

---

## 5. AI Gateway

### 5.1 职责

AI Gateway 是所有 Agent（个人 + 企业）调用 LLM 和 MCP 服务的统一入口：

```mermaid
graph TB
    Agent[Agent<br/>个人/企业] -->|"OpenAI / Anthropic API<br/>+ Consumer Token"| AIGW

    subgraph AIGW["AI Gateway"]
        Auth["Auth Layer<br/>Consumer Token → Agent 身份"]
        LLM_R["LLM Router<br/>模型路由 + 限流"]
        MCP_R["MCP Router<br/>MCP Server 鉴权 + 路由"]
        QA["Quota &amp; Audit<br/>用量计量 + 审计日志"]
        Auth --> LLM_R --> QA
        Auth --> MCP_R --> QA
    end

    LLM_R --> LLM1[国产模型 API<br/>通义/文心/...<br/>兼容协议]
    LLM_R --> LLM2[OpenAI GPT]
    MCP_R --> MCPS[MCP Servers<br/>外部工具服务]
```

### 5.2 Consumer Token 机制

每个 Agent 分配一个 Consumer Token，用于 AI Gateway 鉴权：

```yaml
consumers:
  - name: "agent-客服小助手"
    token: "hc_xxxx"           # Consumer Token
    agent_id: "agent-kefu"
    agent_type: "enterprise"    # enterprise | personal
    permissions:
      llm:
        - model: "qwen-max"
          rpm_limit: 60
          tpm_limit: 100000
        - model: "gpt-4o"
          rpm_limit: 30
      mcp:
        - server: "web-search"
          allowed_tools: ["search", "extract"]
        - server: "database"
          allowed_tools: ["query"]
    quota:
      daily_llm_tokens: 500000
      monthly_cost_limit: 1000  # USD
```

**安全模型：**
- 真实 API Key 只在 AI Gateway 内部，Agent 只持有 Consumer Token
- AI Gateway 按 Consumer Token 做鉴权、限流、计量
- 支持按 agent / 按 group 粒度的限流

### 5.3 LLM Router

```yaml
routes:
  - name: "qwen-max"
    type: "openai_compatible"
    base_url: "https://dashscope.aliyuncs.com/compatible-mode/v1"
    api_key: "${DASHSCOPE_API_KEY}"    # 从环境变量读取，不暴露给 agent
    models: ["qwen-max", "qwen-plus"]
    fallback: "gpt-4o"                 # 降级模型

  - name: "gpt-4o"
    type: "openai"
    base_url: "https://api.openai.com/v1"
    api_key: "${OPENAI_API_KEY}"
    models: ["gpt-4o", "gpt-4o-mini"]

  - name: "claude-opus"
    type: "anthropic"
    base_url: "https://api.anthropic.com"
    api_key: "${ANTHROPIC_API_KEY}"
    models: ["claude-opus-4-6", "claude-sonnet-4-6"]
```

Agent 使用 OpenAI 或 Anthropic 协议调用，AI Gateway 负责协议转换并转发到正确的后端。

### 5.4 MCP Router

Agent 通过 MCP 协议调用外部工具服务，AI Gateway 作为 MCP Proxy：

```mermaid
sequenceDiagram
    participant Agent
    participant AIGW as AI Gateway (MCP Router)
    participant MCPS as MCP Server

    Agent->>AIGW: MCP 请求: tools/call { name: "web_search", args: {...} }
    AIGW->>AIGW: 1. 验证 Consumer Token 是否有权限
    AIGW->>AIGW: 2. 路由到对应的 MCP Server
    AIGW->>MCPS: 转发 MCP 请求
    MCPS-->>AIGW: 返回结果
    AIGW-->>Agent: 回传结果
```

### 5.5 参考实现

- HiClaw 的 Higress AI Gateway + Consumer Token 鉴权
- 或基于 OpenAI/Anthropic 兼容 API 规范自建轻量代理（LiteLLM、One API 等）

---

## 6. 管理平台

### 6.1 功能概览

```mermaid
graph LR
    subgraph Mgmt["Management Platform（Web UI）"]
        subgraph AM["Agent 管理"]
            A1[Agent 列表]
            A2[创建/删除]
            A3[镜像管理<br/>构建/推送]
            A4[配置管理]
            A5[权限管理]
            A6[IM 绑定]
        end

        subgraph RM["资源管理<br/>仅企业 Agent"]
            R1[人设管理<br/>版本/Diff]
            R2[记忆管理<br/>版本/Diff]
            R3[Skill 管理<br/>版本/Diff]
            R4[单点推送]
            R5[批量推送]
            R6[灰度推送]
            R7[推送历史]
            R8[回滚]
        end

        subgraph OM["运维监控"]
            O1[实例状态总览]
            O2[资源使用量]
            O3[LLM 用量/成本]
            O4[日志查看]
            O5[告警规则]
            O6[实例管理策略<br/>max_instances 等]
        end
    end
```

> **注意：** 资源版本管理（人设/记忆/skill）仅适用于**企业 Agent**。个人 Agent 的资源由用户本地自行管理。

### 6.2 Agent 镜像管理（一键部署）

```yaml
agent:
  id: "agent-kefu"
  name: "客服小助手"
  type: "enterprise"           # enterprise | personal
  sdk: "claude-agent-sdk"      # 可选: hermes-python, openai-agent, custom
  image: "registry.local/agents/kefu:v1.2.0"
  config:
    isolation: "container"     # container | process-multi | process-reload
    max_instances: 20
    idle_timeout: "30m"
    preempt_strategy: "lru"
  im_bindings:
    - platform: "dingtalk"
      app_key: "${DINGTALK_APP_KEY}"
    - platform: "feishu"
      app_id: "${FEISHU_APP_ID}"
```

**镜像生命周期：**

```mermaid
flowchart LR
    DEV[开发] --> BUILD[构建 Docker Image]
    BUILD --> PUSH[推送到 Registry]
    PUSH --> REG[在管理平台注册]
    REG --> DEPLOY[一键部署/更新]
    DEPLOY --> GW[Gateway 拉取新镜像]
    GW --> ROLL[逐步替换旧实例]
```

### 6.3 资源版本管理与推送

资源（人设/记忆/skill）通过 Git 管理，管理平台提供 Web 操作界面。

**Git 仓库结构：**

```
agents/
└── agent-kefu/
    ├── shared/                    ← Layer 0 公共层
    │   ├── persona.md
    │   ├── MEMORY.md
    │   └── skills/
    │       ├── faq-answering.md
    │       └── order-query.md
    └── groups/                    ← Layer 1 群组层（可选）
        ├── grp-dingtalk-c123/
        │   ├── MEMORY.md
        │   └── skills/
        │       └── group-specific.md
        └── grp-feishu-g456/
            └── ...
```

**推送流程：**

```mermaid
flowchart TD
    EDIT[管理员在 Web UI 编辑资源] --> CHOICE{推送方式}
    CHOICE -->|单点推送| SINGLE[修改 persona.md<br/>→ push 到 Git<br/>→ 通知目标实例 reload]
    CHOICE -->|批量推送| BATCH[更新 MEMORY.md<br/>→ push 到 Git<br/>→ 通知该 Agent 所有实例 reload]
    CHOICE -->|灰度推送| GRAY[先推到 N 个实例<br/>→ 观察效果<br/>→ 确认后全量推送]
```

**版本查看：**
- Diff 对比（任意两个版本）
- 回滚到历史版本
- 推送审计日志（谁、何时、推了什么、到哪些实例）

### 6.4 监控与告警

| 指标 | 数据源 |
|------|--------|
| 运行中的实例数 | Message Gateway |
| 实例 CPU/内存 | Docker stats / 进程监控 |
| LLM 调用量/延迟/错误率 | AI Gateway |
| LLM Token 用量 / 成本 | AI Gateway |
| MCP 工具调用统计 | AI Gateway |
| 活跃 group 数 | Message Gateway |
| 消息吞吐量 | Message Gateway |

### 6.5 技术选型建议

| 组件 | 建议 | 理由 |
|------|------|------|
| 前端 | React + Ant Design | 企业级 UI 组件库 |
| 后端 API | Python (FastAPI) | 与 hermes 生态一致 |
| Git 服务 | Gitea | 轻量自托管 |
| 数据库 | PostgreSQL | 对话历史 + 元数据 |
| 对象存储 | MinIO | 资源文件 + 附件 |
| 监控 | Prometheus + Grafana | 标准监控栈 |

---

## 7. 存储架构与数据流

### 7.1 存储分层

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

### 7.2 数据流：消息处理全链路

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

### 7.3 数据流：资源推送

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

### 7.4 个人 Agent 数据流

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

## 8. 接口总览

### 8.1 子系统间接口

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
| Agent 实例 | PostgreSQL | 对话历史 | SQL |
| Agent 实例 | MinIO | 资源文件 | S3 API |

### 8.2 外部依赖

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
