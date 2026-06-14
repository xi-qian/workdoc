# Management Platform Design

> 属于 [[Enterprise Agent Platform Overview]] 的子系统设计

## 7. 管理平台

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


## Related

* [[Enterprise Platform Overview]]
* [[Storage Architecture]]
* [[AI Gateway Design]]
* [[Message Gateway Design]]

## Tags

#enterprise #management #monitoring #git
