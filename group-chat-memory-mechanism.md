# 群聊记忆机制分析

> **分析对象**：[OpenClaw](../openclaw/) — 开源多通道 AI Agent 平台
> **分析日期**：2026-04-20
> **相关源码路径**：`src/`、`packages/`、`extensions/`、`skills/`

## 1. 记忆架构概览

### 基于 Session Key 的隔离

记忆通过 session key 组织，编码了平台、聊天类型和目标 ID：

| 聊天类型 | Session Key 格式 | 示例 |
|---------|-----------------|------|
| 私聊 | `agent:main:{provider}:direct:{userId}` | `agent:main:telegram:direct:user123` |
| 群聊 | `agent:main:{provider}:group:{groupId}` | `agent:main:telegram:group:-1001234567890` |
| 频道 | `agent:main:{provider}:channel:{channelId}` | `agent:main:telegram:channel:channel123` |

### 存储结构

```
~/.openclaw/agents/<agentId>/
├── memory/
│   └── <agentId>.sqlite       # 内置记忆索引
├── memory/*.md                # 持久化记忆文件
├── sessions/
│   ├── sessions.json          # 会话注册表
│   └── *.jsonl                # 会话记录
└── qmd/                       # QMD 后端数据（启用时）
```

### 两个记忆后端对比

| | 内置 SQLite | QMD |
|--|-------------|-----|
| 搜索方式 | 仅向量（或 sqlite-vec BM25） | BM25 + 向量 + 重排序 |
| 嵌入模型 | 远程 API（OpenAI、Gemini 等） | 本地（node-llama-cpp，免费） |
| Scope 隔离 | 不支持 | 通过 scope 规则控制访问 |
| 离线可用 | 否 | 是 |
| 外部依赖 | 无 | 需安装 `qmd` CLI |

## 2. 单 Agent 多群：记忆隔离现状

### 内置 SQLite：群间不隔离

SQLite schema 中**没有 sessionKey 列**：

```
~/.openclaw/agents/<agentId>/memory/<agentId>.sqlite
├── files       ← 无 sessionKey 字段
├── chunks      ← 无 sessionKey 字段
└── embedding_cache
```

`memory_search` 接受 `sessionKey` 参数，但仅用于缓存预热，**不用于过滤搜索结果**。所有群的 session 文件和记忆文件被索引到同一个数据库中，搜索返回所有群混合的结果。

### QMD Scope 规则：访问控制，非结果过滤

QMD 的 scope 规则控制的是**某个 session 是否被允许调用 memory_search**，但**不会**按 session 过滤搜索结果。

```yaml
memory:
  backend: "qmd"
  qmd:
    scope:
      default: "deny"
      rules:
        # 允许所有私聊
        - action: "allow"
          match: { chatType: "direct" }

        # 允许特定 Telegram 群
        - action: "allow"
          match:
            channel: "telegram"
            chatType: "group"
            keyPrefix: "telegram:group:-1001234567890"

        # 禁止所有 Discord 频道
        - action: "deny"
          match:
            channel: "discord"
            chatType: "channel"
```

匹配维度：
- `channel` — 平台（telegram、discord、slack 等）
- `chatType` — 聊天类型（`direct` / `group` / `channel`）
- `keyPrefix` — 归一化后的 session key 前缀（小写，已去除 `agent:<id>:` 前缀）
- `rawKeyPrefix` — 原始 session key 前缀

默认 scope 仅允许私聊：
```typescript
{ default: "deny", rules: [{ action: "allow", match: { chatType: "direct" } }] }
```

### Scope 的本质："能不能搜" vs "搜到什么"

| 需求 | Scope 能否做到 |
|------|--------------|
| 禁止某些群搜索记忆 | 能，用 deny 规则 |
| 允许所有群搜索记忆 | 能，一条 allow 规则 |
| 每个群只能搜到自己的记忆 | **不能** |

Scope 是二元访问控制（按 session 允许/拒绝），不是结果过滤器。QMD 索引在 agent 级别构建，不是按 session 拆分的。Scope 控制的是"谁能查"，不是"返回什么数据"。

## 3. 群聊特有功能

### 历史消息限制

- `DEFAULT_GROUP_HISTORY_LIMIT = 50` — 每个群保留的消息数
- `MAX_HISTORY_KEYS = 1000` — 最大群历史 key 数量（LRU 淘汰策略）

### 群聊访问策略

```typescript
type GroupPolicy = "disabled" | "allowlist" | "open"
```

- `disabled` — 禁用群聊
- `allowlist` — 仅允许白名单中的群
- `open` — 允许所有群

### 会话记忆持久化

当触发 `/new` 或 `/reset` 时，当前对话上下文会被保存为带日期的 markdown 文件（如 `2026-04-14-team-planning-session.md`）。

### 记忆检索流程

```
用户消息 → 解析 Session Key → 检测聊天类型
→ MemoryManager.getMemorySearchManager()
→ 向量嵌入语义搜索 + 全文关键词搜索
→ 返回带引用的相关记忆结果
→ 注入 agent 上下文
```

### Agent 可用的记忆工具

- `memory_search` — 搜索当前 agent 的记忆索引（所有群混合）
- `memory_get` — 读取当前 agent 记忆工作区中的文件

两个工具都在单个 agent 范围内运行，不支持跨群过滤。

## 4. 隔离总结

### 跨 Agent：完全隔离

每个 agent 拥有独立的记忆索引、记忆文件、会话记录和配置，agent 之间不共享任何数据。

### 单 Agent 内：群间不隔离

| 后端 | 群间隔离能力 | 原因 |
|------|------------|------|
| 内置 SQLite | 无 | schema 中无 sessionKey 字段，搜索结果混合 |
| QMD | 仅访问控制 | scope 规则允许/拒绝 session 查询，但不过滤结果 |

### 实际影响

- 单 agent 多群场景下，群 A 的对话内容可能出现在群 B 的 `memory_search` 结果中
- QMD scope 可以阻止某些群使用记忆功能，但无法让群只看到自己的数据
- 所有群的会话记录被索引到一起

## 5. 真正群间隔离的解决方案

| 方案 | 做法 | 代价 |
|------|------|------|
| 多 Agent | 每个群分配独立 agent | 完全隔离（记忆、会话、配置），运维成本较高 |
| 多 Agent + 汇总 Agent | 独立 agent + 定期拉取各 agent 记忆的协调 agent | 隔离 + 按需汇总，架构复杂 |
| 单 Agent 关闭记忆 | `sessions.enabled: false` | 退化为仅上下文窗口，无持久记忆 |
| QMD + deny 规则 | 禁止所有群的记忆搜索，仅允许私聊 | 部分方案：私聊有记忆，群聊没有 |

### 推荐方案

对于需要群间隔离的生产场景：
1. **首选**：每个群使用独立 agent（完全隔离）
2. **汇总需求**：如需跨群总结，增加一个专用 supervisor agent，通过外部脚本读取多个 agent 的记忆目录
