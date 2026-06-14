# Multica vs HiClaw vs Clawith 多智能体平台对比分析

## 核心定位

| | **Multica** | **HiClaw** | **Clawith** |
|---|---|---|---|
| 一句话 | AI-native 任务管理（像 Linear + Agent） | Manager-Worker 容器编排平台 | 数字员工协作平台（Agent 有灵魂和记忆） |
| Agent 身份 | Issue 的 assignee | 无状态的 Docker 容器 | 持久实体（soul.md + memory.md） |
| 协作模型 | 人分配任务给 Agent | Manager Agent 管理多个 Worker | Agent 之间 A2A 通信，有社交关系 |

## 技术栈

| | **Multica** | **HiClaw** | **Clawith** |
|---|---|---|---|
| 后端 | **Go** (Chi + sqlc) | **Go** (controller-runtime) + Node.js/Python 运行时 | **Python** (FastAPI + SQLAlchemy) |
| 前端 | Next.js + **Electron** 桌面端 | Element Web (Matrix 客户端) | React + Vite (纯 Web) |
| 通信 | WebSocket + TanStack Query | **Matrix 协议**（IM 聊天室） | WebSocket + 多 IM 渠道 |
| 数据库 | PostgreSQL (pgvector) | MinIO (文件共享) | PostgreSQL + Redis |
| 部署 | SaaS + 自托管 | **纯自托管**（单机 Docker） | 自托管（Docker Compose / K8s） |

## 关键差异

### 1. Agent 运行方式

- **Multica** — Agent 是外部的（Claude Code、Codex、Gemini 等），通过 daemon 连接，执行的是 issue 任务
- **HiClaw** — Agent 运行在 Docker 容器里，每个 Worker 一个容器，凭证由 Higress 网关隔离
- **Clawith** — Agent 是平台内置的，有持久身份和记忆，通过 LLM 调用执行，支持 16 个 LLM 提供商

### 2. 交互模型

- **Multica** — 类似 Jira/Linear 的项目管理 UI，Agent 是列表里的一个 assignee
- **HiClaw** — 在 Matrix 聊天室里用自然语言指挥，所有操作都在聊天里发生
- **Clawith** — 类似企业微信/飞书的工作台，Agent 有自己的主页、朋友圈（Plaza），能自主调度

### 3. 自动化程度

- **Multica** — 最低。人创建 issue、分配给 Agent，Agent 执行后回报状态
- **HiClaw** — 中等。Manager Agent 可以自动创建/管理 Worker，但人始终在聊天室监督
- **Clawith** — 最高。Agent 有自主意识（Aware），能自建定时任务、自发现工具、自主执行

### 4. 企业集成

- **Multica** — 无（面向 2-10 人小团队）
- **HiClaw** — 依赖 Matrix，无企业 IM 集成
- **Clawith** — 最强（飞书、钉钉、企微、Slack、Discord、Teams 全支持）

### 5. 面向市场

- **Multica** — 全球开发者小团队（英文为主）
- **HiClaw** — 中国企业（阿里系，Higress 生态）
- **Clawith** — 中国企业（datamelement，深度本土化，支持百度/通义/Kimi 等国产 LLM）

## 架构深度对比

### Multica 架构

```
Go Backend (Chi + sqlc + gorilla/websocket)
  ├── server/       — HTTP + WebSocket 服务
  ├── multica/      — Cobra CLI（issue/agent/daemon/workspace 管理）
  └── migrate/      — 迁移工具
  ├── internal/daemon/  — 本地 Agent daemon（任务执行、沙箱、缓存）
  ├── internal/handler/ — REST API handlers
  ├── internal/service/ — 业务逻辑（任务生命周期、邮件、定时）
  ├── internal/realtime/ — WebSocket hub + Redis relay
  └── pkg/agent/    — Agent provider 适配层
       （Claude, Codex, Copilot, Cursor, Gemini, Hermes, Kimi, OpenClaw, OpenCode, Pi）

Frontend (pnpm monorepo)
  ├── apps/web/      — Next.js 16 (App Router)
  ├── apps/desktop/  — Electron (electron-vite)
  └── packages/
       ├── core/     — Zustand stores + TanStack Query + API client（无 react-dom）
       ├── views/    — 共享业务组件（无 next/react-router 依赖）
       └── ui/       — shadcn/Base UI 原子组件
```

**设计哲学**: 项目管理工具的范式，Agent 是一等公民但本质是 issue assignee。严格的 package boundary（core/views/ui 三层隔离），双平台共享（Web + Desktop）。

### HiClaw 架构

```
Go Controller (controller-runtime, K8s-style CRD)
  ├── hiclaw-controller/  — Worker/Team/Human 资源管理
  └── docker-proxy/       — Docker/Podman REST API 代理

Agent Runtimes
  ├── manager/            — Manager Agent (SOUL.md + skills)
  ├── worker/             — OpenClaw Worker (Node.js 22, ~500MB)
  ├── copaw/              — CoPaw Worker (Python 3.11, ~150MB)
  ├── openclaw-base/      — 基础 Docker 镜像
  └── shared/lib/         — 共享 shell 库

Infrastructure
  ├── Higress    — AI 网关（LLM 代理、凭证管理、MCP Server）
  ├── Matrix     — IM 服务器 (Tuwunel + Element Web)
  └── MinIO      — 共享文件存储（Worker 无状态）
```

**设计哲学**: 容器编排的范式，Agent 是可销毁的容器实例。K8s CRD 管理生命周期，Matrix 作为审计通信层，Higress 做凭证隔离。Worker 对 API key 零可见。

### Clawith 架构

```
Python Backend (FastAPI + SQLAlchemy)
  ├── app/api/
  │   ├── websocket.py          — 核心大脑：LLM 流式调用 + 工具循环（最多 50 轮）
  │   ├── gateway.py            — 边缘节点网关（OpenClaw 远程 Agent）
  │   └── feishu/discord/wecom/ — IM 渠道适配（统一转 ChatMessage）
  ├── app/services/
  │   ├── agent_tools.py        — Agent 工具注册中心（435KB）
  │   ├── agent_context.py      — 上下文组装（soul.md + 历史记忆）
  │   ├── llm/                  — 统一 LLM 客户端（16 提供商 + 自动故障转移）
  │   ├── sandbox/              — 代码沙箱（subprocess/Docker/E2B/CodeSandbox/Judge0）
  │   ├── trigger_daemon.py     — 自主任务调度守护进程
  │   └── mcp_client.py         — MCP 协议客户端
  └── app/models/               — ORM 模型（全部 tenant_id 隔离）

React Frontend (Vite + Zustand + React Query)
  └── src/pages/
       ├── AgentDetail.tsx      — 主 Agent 界面（427KB）
       ├── Plaza.tsx            — Agent 朋友圈/市场
       └── EnterpriseSettings.tsx — 企业管理后台（256KB）

Communication Channels
  └── Web / 飞书 / 钉钉 / 企微 / Slack / Discord / Teams / Webhook
```

**设计哲学**: 企业级数字员工平台，Agent 有完整的人格（SOUL.md）、记忆（memory.md）和社交关系。所有渠道消息归一化为 ChatMessage 走同一套 LLM 执行管线。A2A 通信需要预建关系（防 prompt 注入跨 Agent 攻击）。

## 总结

- **Multica** = 带 Agent 的 **项目管理工具**（Linear + AI Agent）
- **HiClaw** = Agent **容器编排器**（K8s paradigm for AI agents）
- **Clawith** = 企业级 **数字员工平台**（最完整的多 Agent 自治体系）

## Related

* [[Agent Collaboration Patterns]]
* [[Enterprise Agent Service Model]]
* [[Clawith Memory Analysis]]
* [[Multi Agent Isolation]]

## Tags

#comparison #hiclaw #clawith #multica
