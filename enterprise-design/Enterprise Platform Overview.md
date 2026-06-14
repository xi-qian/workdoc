# Enterprise Agent Platform Overview

> 日期：2026-04-14
> 状态：草稿
> 子文档：[[Message Gateway Design]] | [[AI Gateway Design]] | [[Memory Service Design]] | [[Management Platform Design]] | [[Storage Architecture]]

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
flowchart TB

%% ========== Layer 1 ==========
subgraph L1["① IM 平台层"]
direction LR
DT[钉钉]
FS[飞书]
WW[企业微信]
end


%% ========== Layer 2 ==========
subgraph L2["② Agent 运行层"]
direction TB

subgraph PERSONAL["个人 Agent"]
PA[Personal Agent]
end

subgraph ENTERPRISE["企业 Agent Runtime"]
direction TB

GW[Platform Adapters]

UMF[Unified Message Format]

ROUTER[Router<br/>group_id 路由]

IMGR[Instance Manager<br/>实例生命周期管理]

RESP[Response Handler]

EA1[Group Agent A]
EA2[Group Agent B]
EAN[Group Agent N]

GW --> UMF --> ROUTER --> IMGR
IMGR --> EA1
IMGR --> EA2
IMGR --> EAN

end

end


%% ========== Layer 3 ==========
subgraph L3["③ AI Gateway"]
direction TB

IN[AI Gateway Entry]

AUTH[Consumer Token<br/>Authentication]

LLMR[LLM Router]

MCPR[MCP Router]

QUOTA[Quota / Audit / Cost Control]

IN --> AUTH
AUTH --> LLMR
AUTH --> MCPR
LLMR --> QUOTA
MCPR --> QUOTA

end


%% ========== Layer 4 ==========
subgraph L4["④ 管理平台 (Control Plane)"]
direction LR

API[Management API]

GIT[Git / Gitea<br/>配置与版本]

PG[(PostgreSQL)]

OBJ[(MinIO / S3)]

API --> PG
API --> OBJ
GIT --> OBJ

end


%% ========== Layer 5 ==========
subgraph L5["⑤ 外部 AI 服务"]
direction LR

LLM1[国产模型]

LLM2[OpenAI / Frontier Models]

MCP[MCP Tool Servers]

end


%% ========== Connections ==========

DT --> GW
FS --> GW
WW --> GW

PA --> IN
EA1 --> IN
EA2 --> IN
EAN --> IN

LLMR --> LLM1
LLMR --> LLM2

MCPR --> MCP

API -.配置策略.-> ROUTER
API -.监控配置.-> IN


%% ========== Styles ==========

style L1 fill:#f3f4f6,stroke:#9ca3af
style L2 fill:#eef2ff,stroke:#6366f1
style L3 fill:#ecfeff,stroke:#06b6d4
style L4 fill:#f0fdf4,stroke:#22c55e
style L5 fill:#fff7ed,stroke:#f97316

style PERSONAL fill:#fef3c7,stroke:#f59e0b
style ENTERPRISE fill:#dbeafe,stroke:#3b82f6
```

### 2.2 子系统职责

| 子系统 | 职责 | 对外接口 |
|--------|------|---------|
| **Management Platform** | Agent 生命周期管理、企业 Agent 资源版本管理与推送、镜像管理、监控 | REST API（管理前端 + Message Gateway 调用）、Git 协议 |
| **Message Gateway** | 每个企业 Agent 一个实例，持有 IM Bot SDK，接收消息并路由到对应 group 实例 | IM Platform API（入站）、Agent IPC（出站） |
| **Agent Runtime** | 运行 agent 实例，加载人设/记忆/skill，调用 LLM/MCP | Agent SDK API（Gateway 入站）、AI Gateway API（出站） |
| **AI Gateway** | LLM/MCP 调用的集中鉴权、路由、限流、计量 | OpenAI/Anthropic 兼容 API（给 Agent）、MCP Proxy |
| **Memory Service** | 对话记忆的提取、存储、检索与定期总结更新 | REST API（给 Agent Runtime）、Cron 触发 |
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


## Related

* [[Message Gateway Design]]
* [[AI Gateway Design]]
* [[Memory Service Design]]
* [[Management Platform Design]]
* [[Storage Architecture]]
* [[Enterprise Agent Service Model]]
* [[Group Isolation Design]]

## Tags

#enterprise #platform #architecture #overview
