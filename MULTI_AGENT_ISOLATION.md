# NanoClaw 多 Agent 架构与数据隔离

展示 NanoClaw 如何通过多服务实例实现多 Agent 接入，以及通过目录挂载实现数据隔离的设计。

---

## 多实例部署架构

```mermaid
graph TB
    subgraph CHANNEL["📱 消息通道 (飞书/Telegram/Slack)"]
        W1[飞书 Network]
        W2[同一账号<br/>接入多个服务]
    end

    subgraph INSTANCES["🖥️ NanoClaw 服务实例 (多实例部署)"]
        direction LR
        I1[实例 1<br/>nanoclaw-andy<br/>Assistant: Andy]
        I2[实例 2<br/>nanoclaw-bob<br/>Assistant: Bob]
        I3[实例 3<br/>nanoclaw-dev<br/>Assistant: DevHelper]
        I4[实例 N<br/>nanoclaw-xxx<br/>Assistant: ...]
    end

    subgraph STORAGE["💾 数据存储 (按实例隔离)"]
        direction LR
        S1[实例 1 数据<br/>groups/ store/ data/]
        S2[实例 2 数据<br/>groups/ store/ data/]
        S3[实例 3 数据<br/>groups/ store/ data/]
        S4[实例 N 数据<br/>groups/ store/ data/]
    end

    subgraph CONTAINERS["🐳 Agent 容器 (按组隔离)"]
        direction LR
        subgraph C1["实例 1 容器"]
            G1A[Group A<br/>容器]
            G1B[Group B<br/>容器]
        end

        subgraph C2["实例 2 容器"]
            G2A[Group A<br/>容器]
            G2C[Group C<br/>容器]
        end

        subgraph C3["实例 3 容器"]
            G3D[Group D<br/>容器]
        end
    end

    %% 连接关系
    W1 -->|同一账号<br/>多服务接收| W2
    W2 --> I1 & I2 & I3 & I4

    I1 --> S1
    I2 --> S2
    I3 --> S3
    I4 --> S4

    S1 --> G1A & G1B
    S2 --> G2A & G2C
    S3 --> G3D

    %% 样式
    style CHANNEL fill:#e3f2fd
    style INSTANCES fill:#fff3e0
    style STORAGE fill:#f3e5f5
    style CONTAINERS fill:#ffebee
```

---

## 设计说明

### 1. 多 Agent 接入策略

**问题：** 如何让同一个 飞书 账号服务多个不同的 Agent？

**解决方案：** 部署多个 NanoClaw 实例，每个实例对应一个 Agent

| 实例 | Assistant Name | 触发词 | 用途 |
|------|---------------|--------|------|
| `nanoclaw-andy` | Andy | `@Andy` | 通用助手 |
| `nanoclaw-bob` | Bob | `@Bob` | 技术助手 |
| `nanoclaw-dev` | DevHelper | `@DevHelper` | 开发助手 |

**优势：**
- ✅ 无需修改 gateway 代码
- ✅ 每个实例独立运行、独立配置
- ✅ 故障隔离（一个实例崩溃不影响其他）
- ✅ 易于扩展和维护

**实现方式：**
```bash
# 实例 1 - Andy
cd /opt/nanoclaw-andy
ASSISTANT_NAME=Andy npm start

# 实例 2 - Bob
cd /opt/nanoclaw-bob
ASSISTANT_NAME=Bob npm start

# 实例 3 - DevHelper
cd /opt/nanoclaw-dev
ASSISTANT_NAME=DevHelper npm start
```

---

### 2. Group 数据隔离

**隔离层次：**

```mermaid
graph TD
    subgraph INSTANCE["NanoClaw 实例"]
        subgraph ROOT["项目根目录 /opt/nanoclaw-andy"]
            direction LR
            GLOBAL[groups/CLAUDE.md<br/>全局内存]

            subgraph GROUPS["Group 目录"]
                direction LR
                G1[飞书_main/<br/>主控制组]
                G2[飞书_family/<br/>家庭群组]
                G3[飞书_work/<br/>工作群组]
                G4[telegram_dev/<br/>开发群组]
            end

            subgraph DATA["数据目录"]
                direction LR
                D1[data/sessions/<br/>会话存储]
                D2[data/ipc/<br/>IPC 通信]
                D3[store/messages.db<br/>消息数据库]
            end
        end
    end

    subgraph MOUNTS["容器挂载"]
        direction LR
        M1[/workspace/group<br/>→ 飞书_main/]
        M2[/workspace/global<br/>→ groups/]
        M3[/home/node/.claude/<br/>→ data/sessions/main/.claude/]
    end

    GLOBAL --> G1 & G2 & G3 & G4
    G1 & G2 & G3 & G4 --> D1 & D2 & D3

    ROOT --> MOUNTS

    style INSTANCE fill:#e8f5e9
    style MOUNTS fill:#fff3e0
```

**隔离机制：**

| 隔离维度 | 实现方式 | 说明 |
|---------|---------|------|
| **实例级隔离** | 独立项目目录 | 每个 nanoclaw 实例有独立的 `groups/`、`data/`、`store/` |
| **组级隔离** | 独立 group 目录 | `groups/{channel}_{group-name}/` |
| **会话隔离** | 独立 session 目录 | `data/sessions/{group}/.claude/` |
| **容器隔离** | 独立挂载点 | 每个容器只挂载自己的 group 目录 |

---

### 3. 消息路由与触发词匹配

```mermaid
sequenceDiagram
    participant U as 用户
    participant WA as 飞书
    participant I1 as 实例 1<br/>@Andy
    participant I2 as 实例 2<br/>@Bob
    participant I3 as 实例 3<br/>@DevHelper

    U->>WA: @Andy 帮我查天气
    WA->>I1: 转发消息
    WA->>I2: 转发消息
    WA->>I3: 转发消息

    I1->>I1: 匹配 @Andy ✅
    I2->>I2: 匹配 @Andy ❌
    I3->>I3: 匹配 @Andy ❌

    I1->>U: 响应用户

    Note over I1,I3: 只有匹配的实例会处理消息
```

**触发词配置：**
```typescript
// 实例 1: Andy
export const ASSISTANT_NAME = 'Andy';
export const TRIGGER_PATTERN = new RegExp(`^@Andy\\b`, 'i');

// 实例 2: Bob
export const ASSISTANT_NAME = 'Bob';
export const TRIGGER_PATTERN = new RegExp(`^@Bob\\b`, 'i');

// 实例 3: DevHelper
export const ASSISTANT_NAME = 'DevHelper';
export const TRIGGER_PATTERN = new RegExp(`^@DevHelper\\b`, 'i');
```

---

### 4. 容器挂载隔离

**每个 Group 的容器挂载配置：**

```mermaid
graph LR
    subgraph HOST["主机文件系统"]
        G1[groups/飞书_main/]
        G2[groups/飞书_family/]
        G3[groups/飞书_work/]

        S1[data/sessions/main/.claude/]
        S2[data/sessions/family/.claude/]
        S3[data/sessions/work/.claude/]
    end

    subgraph C1["容器: Group Main"]
        M1[/workspace/group<br/>↓<br/>飞书_main/]
        M2[/home/node/.claude/<br/>↓<br/>main/.claude/]
    end

    subgraph C2["容器: Group Family"]
        M3[/workspace/group<br/>↓<br/>飞书_family/]
        M4[/home/node/.claude/<br/>↓<br/>family/.claude/]
    end

    subgraph C3["容器: Group Work"]
        M5[/workspace/group<br/>↓<br/>飞书_work/]
        M6[/home/node/.claude/<br/>↓<br/>work/.claude/]
    end

    G1 --> M1
    G2 --> M3
    G3 --> M5

    S1 --> M2
    S2 --> M4
    S3 --> M6

    style HOST fill:#e3f2fd
    style C1 fill:#c8e6c9
    style C2 fill:#fff9c4
    style C3 fill:#ffccbc
```

**挂载规则：**

| 主机路径 | 容器路径 | 访问权限 | 说明 |
|---------|---------|---------|------|
| `groups/飞书_main/` | `/workspace/group` | 读写 | 当前组的目录 |
| `groups/CLAUDE.md` | `/workspace/global` | 只读 | 全局内存 |
| `data/sessions/main/.claude/` | `/home/node/.claude/` | 读写 | 会话数据 |
| `data/env/env` | `/workspace/env-dir/env` | 只读 | 环境变量 |

**隔离保证：**
- ✅ Group A 的容器无法访问 Group B 的文件
- ✅ 不同实例的数据完全隔离
- ✅ 容器崩溃或被攻击不影响其他组
- ✅ 资源限制可按组配置

---

## 部署示例

### 场景：部署 3 个不同 Agent

```bash
# 1. 安装三个实例
git clone https://github.com/your-repo/nanoclaw.git /opt/nanoclaw-andy
git clone https://github.com/your-repo/nanoclaw.git /opt/nanoclaw-bob
git clone https://github.com/your-repo/nanoclaw.git /opt/nanoclaw-dev

# 2. 配置不同的 Assistant Name
# /opt/nanoclaw-andy/.env
ASSISTANT_NAME=Andy
CLAUDE_CODE_OAUTH_TOKEN=sk-ant-oat01-xxx

# /opt/nanoclaw-bob/.env
ASSISTANT_NAME=Bob
CLAUDE_CODE_OAUTH_TOKEN=sk-ant-oat01-yyy

# /opt/nanoclaw-dev/.env
ASSISTANT_NAME=DevHelper
CLAUDE_CODE_OAUTH_TOKEN=sk-ant-oat01-zzz

# 3. 共享 飞书 认证
# 所有实例使用相同的 飞书 session
cp /opt/nanoclaw-andy/store/auth/* /opt/nanoclaw-bob/store/auth/
cp /opt/nanoclaw-andy/store/auth/* /opt/nanoclaw-dev/store/auth/

# 4. 启动服务
cd /opt/nanoclaw-andy && npm start &
cd /opt/nanoclaw-bob && npm start &
cd /opt/nanoclaw-dev && npm start &

# 5. 配置服务管理 (systemd)
sudo systemctl enable nanoclaw-andy
sudo systemctl enable nanoclaw-bob
sudo systemctl enable nanoclaw-dev
sudo systemctl start nanoclaw-andy nanoclaw-bob nanoclaw-dev
```

---

## 数据隔离总结

### 隔离边界

```
┌─────────────────────────────────────────────────────────────┐
│  物理隔离层：独立 NanoClaw 实例                              │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │
│  │ Instance 1  │  │ Instance 2  │  │ Instance 3  │         │
│  │ @Andy       │  │ @Bob        │  │ @DevHelper  │         │
│  └─────────────┘  └─────────────┘  └─────────────┘         │
└─────────────────────────────────────────────────────────────┘
         ↓                   ↓                   ↓
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│ 目录隔离层  │    │ 目录隔离层  │    │ 目录隔离层  │
│ groups/     │    │ groups/     │    │ groups/     │
│ ├─ main/    │    │ ├─ main/    │    │ ├─ main/    │
│ ├─ family/  │    │ ├─ team/    │    │ ├─ dev/     │
│ └─ work/    │    │ └─ project/ │    │ └─ support/ │
└─────────────┘    └─────────────┘    └─────────────┘
         ↓                   ↓                   ↓
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│ 容器隔离层  │    │ 容器隔离层  │    │ 容器隔离层  │
│ Docker 挂载 │    │ Docker 挂载 │    │ Docker 挂载 │
│ 独立挂载点  │    │ 独立挂载点  │    │ 独立挂载点  │
└─────────────┘    └─────────────┘    └─────────────┘
```

### 隔离保证

| 隔离类型 | 实现方式 | 保护内容 |
|---------|---------|---------|
| **实例隔离** | 独立进程、独立目录 | 不同 Agent 的配置、数据、运行状态 |
| **组隔离** | 独立 group 目录 | 不同用户/群组的对话历史、文件、记忆 |
| **会话隔离** | 独立 session 目录 | Claude 会话上下文、对话历史 |
| **容器隔离** | 独立容器、独立挂载 | 运行时环境、文件系统访问、进程空间 |

---

**文档版本:** 1.0
**最后更新:** 2025-03-23
**相关文档:** [SYSTEM_LAYERS.md](SYSTEM_LAYERS.md) | [ARCHITECTURE.md](ARCHITECTURE.md)
