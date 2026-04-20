# Plugin 机制分析

## 1. Plugin 系统概述

Plugin 是 OpenClaw 的模块化扩展系统，允许在不修改核心代码的情况下扩展功能。插件通过 Plugin SDK（`src/plugin-sdk/`）提供的标准接口与核心交互。

Extension 是 Plugin 的载体，是一个独立的 npm 包，位于 `extensions/` 目录下。一个 Extension 可以注册一个或多个 Plugin。

### Plugin 分类

| 类型 | 说明 | 示例 |
|------|------|------|
| Channel Plugin | 消息平台适配器 | MSTeams、Matrix、Zalo、Voice Call |
| Provider Plugin | AI 模型提供商 | OpenAI、Anthropic、Ollama |
| Memory Plugin | 记忆存储后端 | QMD、自定义向量库 |
| Context Engine Plugin | 上下文处理扩展 | 自定义嵌入、文档索引 |

### Plugin 与 Extension 的关系

Plugin 是逻辑单元，Extension 是具体载体：

```
extensions/
├── msteams/
│   ├── package.json          # openclaw.extensions 指向入口
│   ├── openclaw.plugin.json  # Plugin 清单
│   ├── index.ts              # 入口：调用 api.registerChannel()
│   └── src/                  # 插件实现
├── matrix/
│   └── ...
└── voice-call/
    └── ...
```

## 2. Plugin 能做什么

### 2.1 Hook（生命周期钩子）

插件可以挂载到 agent 运行的各个阶段：

```
消息流入:  message_received → inbound_claim → before_agent_start → session_start
Agent 运行: before_prompt_build → before_model_resolve → before_tool_call → after_tool_call
LLM 调用:  llm_input → llm_output
消息流出:  message_sending → before_message_write → message_sent
会话管理:  before_compaction → after_compaction → before_reset → session_end
子 Agent:  subagent_spawning → subagent_spawned → subagent_ended → subagent_delivery_target
网关:      gateway_start → gateway_stop
```

### 2.2 注册能力

| 能力 | API | 说明 |
|------|-----|------|
| 消息通道 | `api.registerChannel()` | 接入新消息平台 |
| Agent 工具 | `api.registerTool()` | 给 agent 添加自定义工具 |
| CLI 命令 | `api.registerCli()` | 扩展 CLI 命令 |
| HTTP 路由 | `api.registerHttpRoute()` | 添加自定义 Web 端点 |
| AI 模型提供商 | manifest `providers` 字段 | 接入新模型 |
| 记忆后端 | manifest `kind: "memory"` | 自定义记忆存储 |

## 3. Plugin 加载流程

```
发现（Discovery）
├── 工作区扩展 ./extensions/
├── 全局扩展 ~/.claude/plugins/
├── 内置扩展（bundled）
└── npm 包 @openclaw/*
    ↓
验证（安全检查、路径安全、权限校验）
    ↓
加载（Jiti 加载 TS/JS 模块，解析 SDK 别名）
    ↓
注册（创建记录，注册 hooks / tools / commands / routes）
```

发现时如有重复插件，优先级：配置 > 显式安装 > 内置 > 工作区 > 全局。

### Plugin 隔离机制

- 文件系统限制：插件只能访问指定目录
- 路径安全检查：防止路径穿越攻击
- 权限验证：检查文件权限和所有权
- 命令超时：防止长时间运行阻塞
- 依赖隔离：插件依赖放在 extension 的 `package.json` 中，不污染根项目

## 4. Plugin 清单格式

`openclaw.plugin.json`：

```json
{
  "id": "plugin-id",
  "name": "Plugin Name",
  "version": "1.0.0",
  "kind": "memory",
  "channels": ["channel-id"],
  "providers": ["provider-id"],
  "providerAuthEnvVars": {
    "provider-id": ["API_KEY"]
  },
  "configSchema": { },
  "uiHints": {
    "config-key": {
      "label": "显示名称",
      "help": "帮助文本",
      "advanced": false
    }
  }
}
```

## 5. Plugin 安装与分发

| 来源 | 安装方式 |
|------|---------|
| npm | `npm install @openclaw/plugin-name` |
| Git | `git@github.com:user/repo.git#path` |
| 本地路径 | `./local-plugin` |
| Clawhub（插件市场） | `claw install marketplace:plugin-name` |

管理命令：

```
claw plugins list          # 列出已安装插件
claw plugins enable <id>   # 启用插件
claw plugins disable <id>  # 禁用插件
claw plugins install <src> # 安装插件
```

---

# Channel Plugin 工作机制

## 6. Channel Plugin 接口

Channel Plugin 通过实现一组 Adapter 来描述自己的能力：

```typescript
type ChannelPlugin = {
  // ---- 必选 ----
  id: ChannelId;                          // 唯一标识，如 "msteams"
  meta: ChannelMeta;                      // 元信息（名称、文档、排序）
  capabilities: ChannelCapabilities;      // 声明支持的能力
  config: ChannelConfigAdapter;           // 账号配置管理

  // ---- 可选 Adapter ----
  setup?: ChannelSetupAdapter;            // 初始化设置向导
  pairing?: ChannelPairingAdapter;        // 设备配对
  security?: ChannelSecurityAdapter;      // 安全策略（DM 策略等）
  groups?: ChannelGroupAdapter;           // 群聊策略
  outbound?: ChannelOutboundAdapter;      // 出站消息发送
  status?: ChannelStatusAdapter;          // 状态监控 / 探活
  gateway?: ChannelGatewayAdapter;        // Webhook / HTTP 服务
  auth?: ChannelAuthAdapter;              // 认证流程
  commands?: ChannelCommandAdapter;       // 通道级命令
  messaging?: ChannelMessagingAdapter;    // 目标解析与消息格式化
  mentions?: ChannelMentionAdapter;       // @提及处理
  threading?: ChannelThreadingAdapter;    // 线程 / 回复
  streaming?: ChannelStreamingAdapter;    // 流式输出
  directory?: ChannelDirectoryAdapter;    // 通讯录
  agentTools?: ChannelAgentToolFactory;   // 给 agent 添加通道特有工具
  lifecycle?: ChannelLifecycleAdapter;    // 生命周期管理
  heartbeat?: ChannelHeartbeatAdapter;    // 心跳 / 长连接保活
};
```

## 7. 消息流转

### 入站（平台 → Agent）

```
消息平台 Webhook
    ↓
gateway adapter（HTTP 服务接收请求）
    ↓
monitor-handler（解析平台特定格式，提取消息）
    ↓
安全检查（allowlist、DM 策略、群策略）
    ↓
消息格式归一化（转为统一的 MsgContext）
    ↓
路由解析（决定由哪个 agent 处理）
    ↓
dispatchInboundMessage → Agent 处理
```

### 出站（Agent → 平台）

```
Agent 生成回复
    ↓
payload 归一化（统一的 ReplyPayload）
    ↓
outbound adapter（按平台格式化：分段、媒体上传等）
    ↓
平台 API 调用发送
    ↓
返回 OutboundDeliveryResult
```

## 8. Channel Plugin 生命周期

```
Plugin 发现 → 注册（api.registerChannel）
    ↓
lifecycle.start()
├── 初始化账号配置
├── 启动 Webhook 服务 / 建立长连接
└── 注册状态探活
    ↓
运行中（收发消息、心跳保活、状态上报）
    ↓
lifecycle.stop() → 释放资源、关闭连接
```

生命周期 Adapter 接口：

```typescript
type ChannelLifecycleAdapter = {
  start?: (params: {
    cfg: OpenClawConfig;
    runtime: RuntimeEnv;
    abortSignal?: AbortSignal;
  }) => Promise<Handle>;
  stop?: (handle: Handle) => void | Promise<void>;
  onStop?: () => void | Promise<void>;
};
```

## 9. 内置通道 vs 插件通道

| | 内置通道（Telegram / Discord / Slack） | 插件通道（Teams / Matrix / Zalo） |
|--|-------------------------------------|-------------------------------|
| 代码位置 | `src/telegram`、`src/discord`、`src/slack` | `extensions/msteams/`、`extensions/matrix/` |
| 加载方式 | 编译进主二进制 | 运行时动态加载 |
| 核心访问 | 直接访问内部 API | 通过稳定 plugin API |
| 热更新 | 需重启 | 可独立重载 |
| 版本隔离 | 与核心绑定 | 独立版本 |

两者最终都通过相同的 channel 框架运行，区别只是代码的组织方式和加载时机。

## 10. 群聊与私聊的处理差异

### groups adapter 控制群聊行为

```typescript
type ChannelGroupAdapter = {
  // 是否需要 @机器人 才响应
  resolveRequireMention?: (params) => boolean;
  // 群内首次使用时的提示信息
  resolveGroupIntroHint?: (params) => string;
  // 群内工具使用策略（可限制某些工具）
  resolveToolPolicy?: (params) => GroupToolPolicyConfig;
};
```

- **私聊**：通过 `security` adapter 的 DM 策略控制（allowlist、配对验证等）
- **群聊**：通过 `groups` adapter 控制（@提及触发、工具限制、intro 提示等）
- 两者共享消息归一化和路由框架，差异仅在 adapter 实现中体现

## 11. 能力声明

```typescript
capabilities: {
  chatTypes: ["direct", "group", "channel", "thread"],
  polls: true,       // 支持投票
  threads: true,      // 支持线程回复
  media: true,       // 支持媒体文件
  reactions: false,  // 支持表情反应
}
```

核心通过 `capabilities` 判断该通道支持什么功能，决定是否启用对应特性（如投票组件、线程回复、媒体上传等）。

## 12. 以 MSTeams 为例

### 目录结构

```
extensions/msteams/
├── index.ts                 # 入口：调用 api.registerChannel()
├── src/
│   ├── channel.ts           # msteamsPlugin 对象，实现各 adapter
│   ├── runtime.ts           # 运行时工具
│   ├── monitor.ts           # Webhook 监控服务
│   └── monitor-handler.ts   # 消息解析与处理
```

### 注册入口

```typescript
// index.ts
const plugin = {
  id: "msteams",
  name: "Microsoft Teams",
  register(api: OpenClawPluginApi) {
    setMSTeamsRuntime(api.runtime);
    api.registerChannel({ plugin: msteamsPlugin });
  },
};
```

### 能力声明

```typescript
export const msteamsPlugin: ChannelPlugin<ResolvedMSTeamsAccount> = {
  id: "msteams",
  meta: {
    label: "Microsoft Teams",
    docsPath: "/channels/msteams",
    order: 60,
  },
  capabilities: {
    chatTypes: ["direct", "channel", "thread"],
    polls: true,
    threads: true,
    media: true,
  },
  config: { /* Bot Framework 配置解析 */ },
  gateway: { /* Webhook 服务 */ },
  outbound: { /* 消息发送 */ },
  status: { /* 状态探活 */ },
  groups: { /* 群策略 */ },
};
```

## 13. Webhook / HTTP 集成

Channel Plugin 通过 gateway adapter 接入 Webhook：

- 动态路由注册：按需创建 webhook 端点
- 路径归一化：标准化的 webhook 路径
- 请求守卫：内置速率限制和安全校验
- 服务保活：`keepHttpServerTaskAlive` 维持 HTTP 服务器生命周期

```typescript
// 典型 Webhook 启动模式
export async function monitorProvider(opts) {
  // 创建 HTTP 服务
  // 注册 webhook 路由
  // 启动消息处理器
  // 返回 shutdown 函数
}
```

## 14. 共享基础设施

核心为所有 channel plugin 提供统一的中间层：

| 基础设施 | 说明 |
|---------|------|
| 消息归一化 | `ReplyPayload` 统一出站格式，`MsgContext` 统一入站上下文 |
| 认证管理 | `ChannelAuthAdapter` 统一 OAuth / API Key 流程 |
| 状态监控 | `ChannelStatusAdapter` 统一探活与健康上报 |
| 安全策略 | allowlist、DM 策略、群策略统一框架 |
| 配置管理 | `ChannelConfigAdapter` 统一配置解析与验证 |
| 工具策略 | `resolveToolPolicy` 群内工具使用限制 |
