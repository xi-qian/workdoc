# Hermes 会话学习循环分析

## Intuition

Hermes 的每次对话不仅仅是完成任务，还通过三条路径"学习"：**Memory**（记住用户是谁、环境是什么）、**Skill**（沉淀怎么做某类任务）、**Curator**（定期整理和合并技能）。类似于一个实习生每次干活后写工作笔记、整理操作手册、定期归档——下次遇到类似问题就能更快更好。

## 全流程概览

```
对话启动
  ├─ maybe_run_curator() — 如果距上次 curator 超过 7 天，触发技能整理
  ├─ MemoryStore.load_from_disk() — 加载 MEMORY.md / USER.md 快照
  ├─ 系统提示注入 — 冻结的 memory 快照 + 技能索引
  └─ 外部 provider 初始化
        │
每轮对话 ──────────────────────────────────────────
  │
  ├─ 用户消息到达
  ├─ 外部 provider prefetch（召回相关上下文）
  ├─ 调用 LLM
  ├─ 工具执行（可能包含 memory / skill_manage 调用）
  ├─ sync_all() — 外部 provider 持久化本轮对话
  ├─ 计数器 tick: _turns_since_memory++, _iters_since_skill++
  ├─ 检查是否触发 review
  │   ├─ memory: 每 10 轮用户对话
  │   └─ skill:  每 10 次工具调用迭代
  └─ 如果触发 → 后台 fork 一个 AIAgent 做 review
        │
对话结束
  ├─ on_session_end(messages) — 所有 provider 做最终提取
  └─ shutdown_all() — 关闭 provider 资源
        │
下次对话 ──────────────────────────────────────────
  ├─ 看到上一轮写入的 MEMORY.md / USER.md（load_from_disk）
  ├─ 看到更新后的技能索引（扫描 ~/.hermes/skills/）
  └─ 外部 provider prefetch 召回历史上下文
```

## 三条学习路径

### 路径一：Memory — 记住"用户是谁"

#### 写入时机

智能体在对话中通过 `memory` 工具写入，有两个触发来源：

**1. 前台主动写入**

系统提示中的 `MEMORY_GUIDANCE` 指导智能体在以下情况主动保存：
- 用户纠正了智能体（"不要这样做"）
- 用户分享了偏好、习惯、个人细节
- 智能体发现了关于环境的事实
- 学到了某个 API quirks 或工作流

指导原则：
- 写成陈述性事实，不写指令（"用户喜欢简洁的回答"，不是"总是简洁地回答"）
- 7 天内会过时的不存
- 流程性的东西放 skills，不放 memory
- 优先级：用户偏好和纠正 > 环境事实 > 过程性知识

**2. 后台 review 写入**

每 10 轮用户对话（可配置 `memory.nudge_interval`），系统在后台 fork 一个 AIAgent，用 `_MEMORY_REVIEW_PROMPT` 审视完整对话历史：

```
回顾上面的对话，考虑是否需要保存到记忆中。

关注：
1. 用户是否透露了关于自己的信息——性格、愿望、偏好或个人细节？
2. 用户是否表达了对你行为方式的期望——工作风格、操作习惯？

如果有值得记住的，用 memory 工具保存。
如果没有，就说 "Nothing to save." 然后停止。
```

#### 写入后的影响

| 时机 | 行为 |
|------|------|
| 写入瞬间 | 立即持久化到磁盘（原子写入 + 文件锁）；运行时状态更新 |
| 当前会话后续轮次 | **系统提示中的 memory 快照不变**（冻结设计，保护前缀缓存） |
| 下一轮用户消息 | 外部 provider 的 prefetch 可能召回新信息（如有外部 provider） |
| 上下文压缩后 | 系统提示重建，`load_from_disk()` 重新加载，新 memory **可见** |
| 下次会话 | `load_from_disk()` 加载最新 MEMORY.md / USER.md，全部可见 |

关键约束：同一会话中写入的 memory 对智能体自身有"盲区"——系统提示中的快照不更新，只有压缩或新会话才能看到。

### 路径二：Skill — 沉淀"怎么做这类任务"

#### 写入时机

**1. 前台主动创建**

`SKILLS_GUIDANCE` 系统提示指导智能体：
- 完成复杂任务后（5+ 次工具调用）
- 修复了一个棘手的错误后
- 发现了非平凡的工作流后

主动调用 `skill_manage(action="create")` 保存操作方法。前台创建的技能标记为 **user-owned**，curator 永远不会动它。

**2. 后台 review 创建**

每 10 次工具调用迭代（可配置 `skills.creation_nudge_interval`），后台 fork 的 AIAgent 用 `_SKILL_REVIEW_PROMPT` 审视对话：

```
要主动——大多数会话至少产生一个技能更新。

关注以下信号：
- 用户对风格/语气/格式的纠正
- 工作流的修正
- 非平凡的技术或 workaround
- 已加载的技能被发现是错误或过时的

优先级：
(1) 更新当前已加载的技能
(2) 更新已有的类级 umbrella 技能
(3) 在已有 umbrella 下添加支撑文件
(4) 创建新的类级 umbrella 技能

当用户抱怨某个任务的处理方式时，教训应该沉淀到
管理该任务的 skill 里，而不仅仅是 memory。
```

后台 review 创建的技能标记为 **agent-created**，curator 可以管理它。

#### 技能来源标记（Provenance）

通过 `ContextVar` 追踪写入来源：

```python
# 正常对话时
_write_origin = "foreground"  → 创建的技能是 user-owned

# 后台 review 时
_write_origin = "background_review" → 创建的技能是 agent-created
```

这个标记决定了 curator 能否对技能做合并/归档操作。

#### skill_manage 工具支持的操作

| 动作 | 用途 |
|------|------|
| `create` | 创建 SKILL.md + 目录 |
| `edit` | 全文重写 SKILL.md |
| `patch` | 定向查找替换 SKILL.md 或支撑文件 |
| `delete` | 删除技能（受 pin 保护） |
| `write_file` | 添加支撑文件（references/、templates/、scripts/、assets/） |
| `remove_file` | 删除支撑文件 |

每次修改后，技能系统提示缓存立即清除，下一轮对话就能看到更新。

### 路径三：Curator — 定期整理技能库

#### 触发条件

CLI 启动时检查，距上次 curator 运行超过 7 天（`DEFAULT_INTERVAL_HOURS = 168`）才触发。是一个低频的维护任务。

#### 做什么

**1. 自动状态流转**（不涉及 LLM）

```
技能未被使用超过 30 天 → 标记为 stale
技能未被使用超过 90 天 → 标记为 archived
已归档的技能被重新使用 → 标记回 active
被 pin 的技能 → 永远不碰
user-owned 的技能 → 永远不碰
```

**2. LLM 驱动的合并**（fork 一个 AIAgent）

```
CURATOR_REVIEW_PROMPT 指导：
- 识别前缀相关的窄技能簇
- 合并为类级 umbrella 技能
- 把会话级细节移入 references/、templates/、scripts/
- 归档被吸收的兄弟技能
- 永远不删除——只归档（可恢复）
```

#### 只碰 agent-created 技能

user-owned 技能完全免疫——curator 不会修改、合并或归档它们。

## 后台 Review Agent 的实现细节

### fork 机制

后台 review 不是一个简单的 prompt，而是 fork 了一个**完整的 AIAgent 实例**：

```python
review_agent = AIAgent(
    model=parent.model,           # 同模型
    provider=parent.provider,     # 同提供商
    enabled_toolsets=["memory", "skills"],  # 只开放 memory + skill 工具
    _memory_write_origin="background_review",  # 标记来源
    _memory_nudge_interval=0,     # 禁止递归 review
    _skill_nudge_interval=0,      # 禁止递归 review
)
```

关键设计：
- **禁止递归**：review agent 的 nudge interval 设为 0，不会触发下一层 review
- **工具限制**：只开放 memory 和 skills 工具，不能执行终端命令、写文件等
- **完整上下文**：继承父 agent 的完整对话历史
- **守护线程**：在 daemon 线程中运行，不阻塞用户对话
- **结果摘要**：完成后扫描 review agent 的消息，提取工具调用结果，生成摘要如 `"Self-improvement review: Memory updated · Skill 'deploy' patched"`

### 何时触发

| 触发类型 | 计数器 | 默认阈值 | 重置条件 |
|---------|--------|---------|---------|
| Memory review | `_turns_since_memory`（用户轮次） | 10 | 达到阈值时重置；agent 主动调用 memory 工具也重置 |
| Skill review | `_iters_since_skill`（工具迭代） | 10 | 达到阈值时重置；agent 主动调用 skill_manage 也重置 |

两者独立计数，可以同时触发（此时用 `_COMBINED_REVIEW_PROMPT` 合并处理）。

## 会话中的上下文流转

```
                    ┌─────────────────────────────────┐
                    │        系统提示（冻结前缀）        │
                    │  ┌───────────────────────────┐  │
                    │  │ MEMORY.md 快照（会话开始时） │  │
                    │  │ USER.md 快照（会话开始时）   │  │
                    │  │ 技能索引                    │  │
                    │  └───────────────────────────┘  │
                    └─────────────────────────────────┘
                                   │
    ┌──────────────────────────────┼──────────────────────────────┐
    │                              │                              │
    ▼                              ▼                              ▼
 用户消息                      LLM 调用                     助手响应
 + <memory-context>           完整 messages              + 可能有 tool_calls
   (外部 provider            包含冻结系统提示
    prefetch 结果)
    │                              │                              │
    │                              ▼                              │
    │                     ┌─────────────────┐                     │
    │                     │   工具执行分派    │                     │
    │                     ├─────────────────┤                     │
    │                     │ memory tool     │→ 写入 MEMORY.md     │
    │                     │ skill_manage    │→ 创建/更新技能       │
    │                     │ 其他工具        │→ 正常执行            │
    │                     └─────────────────┘                     │
    │                              │                              │
    │                              ▼                              │
    │                     ┌─────────────────┐                     │
    │                     │ sync_all()      │→ 外部 provider 持久化│
    │                     │ queue_prefetch  │→ 预热下一轮          │
    │                     └─────────────────┘                     │
    │                              │                              │
    │                              ▼                              │
    │                     ┌─────────────────┐                     │
    │                     │ 检查 nudge 触发  │                     │
    │                     │ → 后台 review   │→ fork AIAgent       │
    │                     └─────────────────┘                     │
    └──────────────────────────────┼──────────────────────────────┘
                                   │
                                   ▼
                           下一轮用户消息
```

## 设计评价

### 做得好的部分

**1. 三层学习路径分工明确**

- Memory：事实性知识（用户是谁、环境是什么）—— 轻量、即时
- Skill：过程性知识（怎么做某类任务）—— 结构化、可复用
- Curator：长期维护（合并、归档）—— 避免技能库腐化

三者的职责边界清晰，不重叠。

**2. 后台 review 不干扰用户**

review 在 daemon 线程中运行，用户已经收到响应，不会被阻塞。而且 review agent 只有 memory 和 skills 工具，不能执行危险操作。

**3. Provenance 追踪区分来源**

user-owned vs agent-created 的区分让 curator 可以大胆整理 agent 创建的技能，同时不会破坏用户手动创建的。这是一个很好的信任边界。

**4. 递归防护**

review agent 的 nudge interval 设为 0，不会触发无限嵌套的 review 循环。

### 核心问题

**1. 后台 review 的成本不透明**

每次 review 会 fork 一个完整的 AIAgent，携带完整对话历史做一次 LLM 调用。这意味着：
- Token 成本可能很高（长会话的对话历史很大）
- 用户不知道这个成本正在发生
- 没有预算或频率上限（只有 interval 控制，没有总次数限制）

**2. Memory 的"盲区"问题**

会话中途写入的 memory 不更新系统提示中的冻结快照。对于长会话（不触发压缩），这意味着：
- 智能体可能重复写入相同信息（忘了自己已经记住）
- 用户说"记住这个"后，智能体下次对话才能"想起来"
- 只有压缩或新会话才能消除盲区

**3. Skill nudge 的触发条件不合理**

Skill review 的计数器基于**工具调用迭代次数**（`_iters_since_skill`），不是用户轮次。这意味着：
- 一次复杂任务（很多工具调用）可能在一个用户轮次内就触发 skill review
- 而十次简单对话（没有工具调用）永远不会触发 skill review
- 触发时机与"学习机会"的相关性不高

**4. Curator 的合并逻辑依赖 LLM 判断**

curator 用 LLM 来决定哪些技能应该合并成 umbrella，但 LLM 可能：
- 过度合并（把不相关的技能强行归类）
- 合并方向错误（破坏已有的好结构）
- 用户没有机会 preview 或 veto 合并结果

虽然 user-owned 技能被保护，但 agent-created 技能的合并完全自动，用户可能不知道自己的 agent 学到了什么又被整理了什么。

**5. "Nothing to save" 的判断标准模糊**

review prompt 说"没有值得记住的就说什么都不用存"，但这个判断完全依赖模型的自我评估。模型可能：
- 过度保存（每次 review 都写点什么，哪怕没什么价值）
- 保存过于泛化的信息（"用户喜欢好的代码"——没有信息量）

没有质量门控或去重机制来过滤低价值记忆。

## 改进建议

### 短期：可见性和控制

- **Review 成本提示**：在 review 触发时给用户一个轻量通知（"Running self-improvement review..."），让用户知道有额外成本
- **Review 预算**：增加每个会话的最大 review 次数限制
- **Memory 写入可见**：智能体写入 memory 后，在下一轮对话开头简短确认（"我记住了：..."），弥补盲区

### 中期：质量门控

- **去重检查**：写入 memory 前检查是否已有语义相似的条目，避免重复积累
- **Review 质量评分**：对 review 产出的记忆/技能做简单质量评估（信息密度、具体性），过滤低质量输出
- **Skill review 改用用户轮次**：与 memory review 统一为用户轮次触发，而不是工具迭代

### 长期：用户参与

- **Curator preview**：合并操作前展示计划，让用户确认或调整
- **Memory/Skill 审计日志**：记录每次写入的来源、时间、触发原因，方便用户回溯
- **分级记忆**：核心记忆（始终在系统提示）vs 扩展记忆（按需检索），突破 2200 字符的容量限制
