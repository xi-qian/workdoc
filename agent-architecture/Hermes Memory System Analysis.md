# Hermes Memory 系统分析

## Intuition

Memory 是 Hermes Agent 的"长期记忆"。智能体每次对话都是无状态的——启动时没有上下文，结束后上下文消失。Memory 系统解决的问题是：**让智能体在跨会话、跨平台的情况下，持续积累对用户和环境的认知**。

类似于人类大脑的长期记忆 vs 工作记忆：
- **工作记忆** = 对话历史（每次启动重新来过）
- **长期记忆** = Memory 系统（持久化，跨会话存活）

## 整体架构

```
                        config.yaml
                            |
               memory.memory_enabled: true
               memory.user_profile_enabled: true
               memory.nudge_interval: 10
               memory.provider: "honcho"（可选）
                            |
                 AIAgent.__init__()
                            |
          +-----------------+-----------------+
          |                                   |
     内置 Memory                         外部 Provider
  (tools/memory_tool.py)            (plugins/memory/<name>/)
          |                                   |
     MemoryStore                            MemoryProvider
     +-------+-------+                  (ABC, agent/memory_provider.py)
     |               |
  MEMORY.md       USER.md
  (智能体笔记)     (用户画像)
          |                                   |
          +-----------------+-----------------+
                            |
                  MemoryManager（agent/memory_manager.py）
                            |
                  协调内置 + 外部 provider
```

三层结构：
1. **MemoryStore** — 内置的文件存储（MEMORY.md / USER.md），开箱即用
2. **MemoryProvider** — 可插拔的外部记忆 provider（Honcho、Hindsight、Mem0 等）
3. **MemoryManager** — 编排层，协调内置和外部 provider 的生命周期

## 内置 Memory（MEMORY.md / USER.md）

### 两个存储

| 文件 | 用途 | 默认字符上限 |
|------|------|------------|
| `MEMORY.md` | 智能体的个人笔记——环境事实、项目惯例、工具 quirks | 2200 字符（~800 tokens） |
| `USER.md` | 用户画像——偏好、沟通风格、期望 | 1375 字符（~500 tokens） |

文件位于 `$HERMES_HOME/memories/`，条目之间用双换行分隔。结构是扁平的文本列表，没有层级或分类。

### 工具操作

智能体通过 `memory` 工具操作，支持三个动作：

| 动作 | 参数 | 行为 |
|------|------|------|
| `add` | `target`, `content` | 追加条目（检查上限和去重） |
| `replace` | `target`, `old_text`, `content` | 子串匹配查找，替换 |
| `remove` | `target`, `old_text` | 子串匹配查找，删除 |

### 持久化机制

- **原子写入**：先写临时文件，再 `os.rename()`，防止读到半写状态
- **文件锁**：Unix 用 `fcntl`，Windows 用 `msvcrt`，保证并发安全
- **跨会话共享**：同一 profile 下的 CLI 和网关共享同一组文件，通过文件锁协调

### 安全扫描

`_scan_memory_content()` 在写入前扫描内容，拦截：
- 提示注入模式
- 角色劫持
- 欺骗性内容
- 通过 curl/wget 外泄密钥
- SSH 后门
- 不可见 Unicode 字符

### 内容指导（MEMORY_GUIDANCE）

系统提示中的 `MEMORY_GUIDANCE` 常量告诉智能体：
- **该存**：用户偏好、环境细节、工具 quirks —— 能减少未来用户干预的事实
- **不该存**：任务进度、PR 编号、commit SHA —— 7 天内会过时的信息
- 写成陈述性事实，不要写成给自己的指令
- 流程性的东西放 skills，不放 memory

## 外部 Memory Provider

### Provider ABC

`agent/memory_provider.py` 定义了完整的生命周期接口：

| 方法 | 类型 | 用途 |
|------|------|------|
| `name` | 必须 | 标识符：`"honcho"`, `"hindsight"` 等 |
| `is_available()` | 必须 | 检查配置和依赖 |
| `initialize(session_id, **kwargs)` | 必须 | 初始化连接，接收 `hermes_home`, `platform`, `user_id` 等 |
| `get_tool_schemas()` | 必须 | 返回工具 schema |
| `system_prompt_block()` | 可选 | 注入系统提示的静态文本 |
| `prefetch(query, session_id)` | 可选 | 每轮对话前召回相关上下文 |
| `queue_prefetch(query, session_id)` | 可选 | 后台预取，为下一轮准备 |
| `sync_turn(user, assistant, session_id)` | 可选 | 每轮结束后持久化对话 |
| `on_session_end(messages)` | 可选 | 会话结束时提取记忆 |
| `on_session_switch(...)` | 可选 | session_id 轮换通知 |
| `on_pre_compress(messages)` | 可选 | 上下文压缩前提取（从即将丢弃的消息中抢救信息） |
| `on_memory_write(action, target, content)` | 可选 | 镜像内置 memory 的写入操作 |

### 已有 Provider 实现

| Provider | 特点 |
|----------|------|
| **Honcho** | AI 原生跨会话用户建模，语义搜索，对话式问答，peer cards |
| **Hindsight** | 记忆回顾 |
| **Holographic** | 基于 HRR 的组合检索，实体解析，信任评分 |
| **Mem0** | Mem0 集成 |
| **ByteRover** | 记忆 provider |
| **OpenViking** | 记忆 provider |
| **RetainDB** | 记忆 provider |
| **SuperMemory** | 记忆 provider |

同一时间只能激活一个外部 provider（通过 `memory.provider` 配置）。

### Provider 发现机制

扫描两个目录：
1. **内置插件**：`plugins/memory/<name>/`
2. **用户安装**：`$HERMES_HOME/plugins/<name>/`

通过 `register(ctx)` 模式注册，或自动查找 `MemoryProvider` 子类。

## Memory 在对话中的生命周期

### 1. 会话启动

```
AIAgent.__init__()
    ├─ MemoryStore 加载 MEMORY.md / USER.md
    ├─ 拍摄 system_prompt_snapshot（冻结快照，用于系统提示注入）
    ├─ MemoryManager 初始化所有 provider
    └─ 设置 nudge 计数器 _turns_since_memory = 0
```

### 2. 系统提示注入

系统提示构建时，memory 以**冻结快照**形式注入，顺序：

```
1. Agent 身份（SOUL.md）
2. 工具行为指导（含 MEMORY_GUIDANCE）
3. 用户/网关系统提示
4. MEMORY.md 快照 ← ─┐
5. USER.md 快照   ← ─┤ 内置 memory，冻结在会话开始时
6. 外部 provider 系统提示块 ← ─┘
7. 上下文文件（AGENTS.md 等）
8. 日期时间
9. 平台提示
```

**关键设计**：内置 memory 注入系统提示后，在会话期间**不修改**系统提示。这保证了前缀缓存（prefix cache）不被打破。

### 3. 外部 Provider 上下文注入

外部 provider 的召回结果**不注入系统提示**（会打破缓存），而是注入当前轮次的**用户消息**中：

```
用户消息 = <memory-context>
              [prefetch 结果]
           </memory-context>
           + 用户实际输入
```

这是临时注入——原始 `messages` 列表不被修改，不会泄漏到会话持久化。

### 4. 每轮对话流程

```
用户消息到达
    ├─ 计数器 _turns_since_memory++
    ├─ 检查是否需要 nudge（>= nudge_interval）
    ├─ 外部 provider prefetch（注入用户消息）
    ├─ 调用 LLM
    ├─ 如果有 tool_calls → 分派执行
    │   ├─ memory 工具调用 → MemoryStore.add/replace/remove
    │   │   ├─ 安全扫描
    │   │   ├─ 原子写入文件
    │   │   ├─ 更新运行时状态（memory_entries/user_entries）
    │   │   └─ 通知外部 provider（on_memory_write 镜像）
    │   └─ 其他工具 → 正常执行
    ├─ sync_all(user, assistant) → 外部 provider 持久化本轮对话
    ├─ queue_prefetch_all() → 外部 provider 后台预热下一轮
    └─ 如果 should_review_memory → 启动后台 review agent
```

### 5. Memory Nudge 机制

问题：智能体不会主动使用 memory 工具。解决方案是周期性"轻推"。

```
配置: memory.nudge_interval = 10（每 10 轮用户对话触发一次）

_turns_since_memory 计数器
    ├─ 每个用户轮次 +1
    ├─ 达到 nudge_interval → _should_review_memory = True, 计数器归零
    └─ 智能体调用 memory 工具 → 计数器归零

触发后：
    └─ 在后台启动 review agent
        ├─ 获得 _MEMORY_REVIEW_PROMPT 或 _COMBINED_REVIEW_PROMPT
        └─ 审视对话，判断是否有值得记住的信息
```

网关特殊处理：网关每条消息都创建新 AIAgent 实例，nudge 计数器从对话历史中恢复：

```python
self._turns_since_memory = prior_user_turns % self._memory_nudge_interval
```

### 6. 上下文压缩时的 Memory

当上下文窗口接近上限，触发压缩：

```
上下文压缩
    ├─ 1. memory_manager.on_pre_compress(messages)
    │      → 外部 provider 从即将丢弃的消息中提取记忆
    ├─ 2. commit_memory_session(messages)
    │      → 所有 provider 的 on_session_end()
    ├─ 3. _invalidate_system_prompt()
    │      → 清除缓存
    │      → MemoryStore.load_from_disk()（重新从磁盘加载最新内容）
    ├─ 4. 新 session_id 生成
    └─ 5. memory_manager.on_session_switch(reset=False, reason="compression")
           → 通知 provider 逻辑会话继续，只是物理 session 变了
```

压缩后重新加载 memory 是一个重要的安全阀——如果在压缩过程中有新的记忆写入，压缩后的新上下文能看到最新状态。

### 7. 会话结束

两种路径：

| 路径 | 触发 | 行为 |
|------|------|------|
| `commit_memory_session()` | 压缩前、`/new` | 调用 `on_session_end()`，provider 继续运行 |
| `shutdown_memory_provider()` | CLI 退出、网关会话过期 | 调用 `on_session_end()` + `shutdown_all()`，完全关闭 |

## 设计评价

### 做得好的部分

**1. 缓存友好的注入策略**

内置 memory 冻结在系统提示中，外部 provider 结果注入用户消息。两者都不修改已有的系统提示，完美保护了前缀缓存。这是整个系统最精妙的设计——如果 memory 每轮都更新系统提示，缓存会全部失效，成本会翻数倍。

**2. 双层架构的解耦**

内置 memory 开箱即用（零配置、无外部依赖），外部 provider 按需接入。用户不需要为了基本记忆功能去配数据库或云服务。

**3. 安全扫描**

`_scan_memory_content()` 拦截提示注入、密钥外泄等恶意内容。这在 LLM 自主写入记忆的场景下非常必要——否则恶意用户可以通过对话注入虚假记忆。

**4. Nudge 机制**

解决了 LLM 不会主动使用工具的根本问题。不强制，而是周期性提醒，给模型自主判断的空间。

**5. 压缩时的记忆抢救**

`on_pre_compress()` 让 provider 在消息被丢弃前提取有价值的信息。否则上下文压缩意味着记忆丢失。

### 核心问题

**1. 内置 Memory 的容量严重受限**

MEMORY.md 上限 2200 字符（~800 tokens），USER.md 上限 1375 字符（~500 tokens）。这是系统提示空间的预算约束——每多一个 token 都增加每次 API 调用的成本。

但 800 tokens 意味着智能体只能记住大约 10-15 条事实。对于一个长期使用的 agent 来说，这个容量远远不够。结果是：
- 新信息挤掉旧信息（FIFO 淘汰）
- 智能体被迫做艰难的选择——记住用户的时区偏好还是记住某个项目的构建 quirks
- 信息密度被压缩到极致，丢失细节

**2. 扁平结构没有检索能力**

MEMORY.md 是一个扁平文本文件，没有索引、没有分类、没有语义检索。智能体只能靠 LLM 的上下文窗口来"看到"所有记忆。

这意味着：
- 记忆越多，每条记忆被正确调用的概率越低（淹没在噪声中）
- 无法做精确召回——比如"用户上次讨论项目 X 时提到了什么"
- 添加和删除都依赖子串匹配，不够可靠

**3. 系统提示冻结导致信息滞后**

MEMORY.md 在会话开始时冻结到系统提示中。会话中途写入的新记忆只更新运行时状态和磁盘文件，不影响已注入的系统提示。

这意味着：
- 同一会话中，智能体写入新记忆后，它在后续轮次中"看不到"自己的新记忆（系统提示里的还是旧版本）
- 只有在下一次会话或上下文压缩后重新加载时，新记忆才会出现在系统提示中
- 对于长会话，这可能导致智能体重复写入相同的记忆

**4. Nudge 机制的后台 agent 成本**

后台 review agent 意味着每 10 轮对话触发一次额外的 LLM 调用。这个调用：
- 使用完整的对话历史作为上下文（高 token 成本）
- 可能产生不必要的记忆写入
- 用户无法控制 review 的频率或关闭它（只能设 `nudge_interval: 0`，但这完全禁用了被动记忆）

**5. 外部 Provider 的镜像机制有限**

`on_memory_write()` 将内置 memory 的写入镜像到外部 provider，但反过来不成立——外部 provider 的召回结果不会影响内置 memory。两个系统是单向桥接，不是双向同步。

**6. 网关场景下的记忆隔离**

网关为每个用户/chat 组合提供独立的 `user_id` 和 `chat_id`，外部 provider 可以据此做用户级隔离。但内置 memory（MEMORY.md/USER.md）是 profile 级别共享的——同一个 profile 下的所有用户共享同一份记忆。

这意味着：
- 网关部署中，Alice 和 Bob 通过同一个 Telegram bot 对话，共享同一份 MEMORY.md
- 没有内置的方案来隔离不同用户的记忆

## 改进建议

### 短期：让内置 memory 更可用

- **分类存储**：MEMORY.md 拆分为多个文件（`environment.md`、`preferences.md`、`projects.md`），按需加载而不是全部塞进系统提示
- **写入可见性**：会话中途写入新记忆后，在下一轮的用户消息中追加一个简短的"你刚记住了以下内容"提示，弥补系统提示冻结的信息滞后

### 中期：增加检索层

- **语义索引**：在 MEMORY.md 之上增加一个轻量级的向量索引（如 SQLite + vec 扩展），支持按相关性召回而不是全量注入
- **分层容量**：核心记忆（用户画像、关键偏好）保持在系统提示中；扩展记忆（项目细节、历史对话摘要）通过检索按需注入

### 长期：用户级隔离 + 双向同步

- **网关多用户隔离**：内置 memory 支持 per-user 文件（`memories/<user_id>/MEMORY.md`）
- **双向桥接**：外部 provider 的召回结果可以回写到内置 memory 的扩展存储中
- **记忆版本化**：记录每条记忆的创建时间和来源会话，支持审计和回滚
