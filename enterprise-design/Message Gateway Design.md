# Message Gateway Design

> 属于 [[Enterprise Agent Platform Overview]] 的子系统设计

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


## Related

* [[Enterprise Platform Overview]]
* [[AI Gateway Design]]
* [[Group Isolation Design]]
* [[Hermes Distributed Architecture]]
* [[Multi Agent Isolation]]

## Tags

#enterprise #message-gateway #routing #im
