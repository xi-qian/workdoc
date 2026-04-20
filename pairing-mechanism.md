# Pairing 机制与聊天类型

## 1. Pairing 机制

### 1.1 是什么

Pairing 是 OpenClaw 的**显式批准机制**，控制谁能与 bot 交互。未知的发送者必须经过主人批准才能使用 bot。

### 1.2 解决什么问题

防止陌生人直接给 bot 发消息，实现显式授权访问。

### 1.3 DM 策略

DM = Direct Message，即私聊/私信。每个通道可配置 DM 策略：

| 策略 | 行为 |
|------|------|
| `pairing` | 未知用户需配对批准后才能对话 |
| `open` | 任何人都可以直接对话 |
| `disabled` | 禁止所有私聊 |

### 1.4 配对流程

```
1. 陌生人给 bot 发消息
       ↓
2. 检查 DM 策略 → pairing
       ↓
3. Bot 生成 8 位配对码（如 ABCDEFGH），回复给发送者
       ↓
4. 配对请求存储到 ~/.openclaw/credentials/<channel>-pairing.json
       ↓
5. 主人通过 CLI 批准：openclaw pairing approve telegram ABCDEFGH
       ↓
6. 批准后发送者加入 allowlist：<channel>-allowFrom.json
       ↓
7. （可选）通知被批准的用户
```

### 1.5 CLI 命令

```bash
# 查看待批准的配对请求
openclaw pairing list telegram
openclaw pairing list --channel telegram --account work

# 批准配对
openclaw pairing approve telegram ABCDEFGH
openclaw pairing approve --channel telegram --account work ABCDEFGH --notify
```

### 1.6 ChannelPairingAdapter 接口

```typescript
type ChannelPairingAdapter = {
  idLabel: string;                    // 用户 ID 标签，如 "telegramUserId"
  normalizeAllowEntry?: (entry) => string;  // 归一化输入 ID
  notifyApproval?: (params) => Promise<void>; // 批准后通知用户
};
```

各内置通道的实现：

| 通道 | idLabel |
|------|---------|
| Telegram | `telegramUserId` |
| Discord | `discordUserId` |
| Signal | `signalNumber` |
| Slack | `slackUserId` |
| iMessage | `imessageSenderId` |
| WhatsApp | `whatsappSenderId` |

### 1.7 Node 配对（设备接入）

除了 DM 配对，还有 Node 配对，用于 iOS/Android 等设备接入 Gateway：

```
设备连接 Gateway → 请求配对 → 主人批准（CLI 或 UI）
→ Gateway 签发 token → 设备用 token 建立连接
```

### 1.8 安全特性

- 配对码 **1 小时**过期
- 每个通道最多 **3 个**待批准请求（防刷）
- 多账号场景下 allowlist 按账号隔离
- 配对码排除易混淆字符（0O1I）
- 并发文件锁保证安全写入

### 1.9 存储位置

```
~/.openclaw/credentials/
├── telegram-pairing.json         # 待批准的配对请求
├── telegram-allowFrom.json       # 已批准的发送者白名单
├── telegram-work-allowFrom.json  # 多账号场景，按账号隔离
└── ...
```

---

## 2. 聊天类型（ChatType）

### 2.1 "Channel" 一词的两层含义

OpenClaw 中 "channel" 有两个容易混淆的含义：

1. **Channel（消息平台）** — 如 Telegram、Discord、Slack，是平台概念
2. **Channel（chatType）** — 如 Discord 频道、Telegram 频道，是聊天类型

在一个消息平台（channel）内部，有三种聊天类型。

### 2.2 三种聊天类型

| chatType | 含义 | 示例 |
|----------|------|------|
| `direct` | 私聊 | Telegram 私聊、Discord DM |
| `group` | 群聊 | Telegram 群组、Discord 服务器 |
| `channel` | 频道/广播 | Telegram 频道、Discord 频道 |

Session Key 中的体现：

```
agent:main:telegram:direct:user123           # 私聊
agent:main:telegram:group:-1001234567        # 群组
agent:main:telegram:channel:-1009876543      # 频道
```

### 2.3 群组 vs 频道的区别

| | 群组 (group) | 频道 (channel) |
|--|-------------|----------------|
| 普通成员能发消息 | 能 | 不能 |
| 成员之间能互相回复 | 能 | 不能 |
| 定位 | 多人讨论 | 单向广播 |
| Bot 角色 | 可以是普通成员或管理员 | 只能是管理员 |

- 群组：成员之间可以互相通讯，类似微信群
- 频道：只有管理员能发消息，订阅者只能接收，类似公众号/广播
- Bot 在频道中作为管理员，可以接收消息并发表回复，实现双向交互

### 2.4 Capabilities 声明

Channel Plugin 通过 `capabilities.chatTypes` 声明支持的聊天类型：

```typescript
capabilities: {
  chatTypes: ["direct", "group", "channel"],
}
```

### 2.5 各聊天类型的安全策略

| 聊天类型 | 安全控制 |
|---------|---------|
| 私聊 (direct) | `dm.policy`: pairing / open / disabled + allowlist |
| 群聊 (group) | `groups.policy`: disabled / allowlist / open + @提及要求 + 工具策略 |
| 频道 (channel) | Bot 作为管理员加入，由平台控制访问权限 |
