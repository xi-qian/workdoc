# Clawith 单 Agent 多用户一对一聊天：记忆更新机制分析

## 记忆存储结构

Agent 的记忆是纯文件系统，没有用户维度：

```
AGENT_DATA_DIR/{agent_id}/
├── memory/
│   ├── memory.md          ← 唯一的记忆文件，所有用户共享
│   ├── reflections.md     ← 自主思考日志
│   └── MEMORY_INDEX.md    ← 记忆索引模板
├── soul.md                ← 人格定义
└── ...
```

路径中没有 user_id，所有与该 Agent 对话的用户读写的是同一个 `memory.md`。

## 会话隔离 vs 记忆共享

| 维度 | 是否隔离 | 证据 |
|------|----------|------|
| ChatSession | 按 user_id + agent_id 隔离 | `websocket.py` 查询条件：`ChatSession.agent_id == agent_id, ChatSession.user_id == user_id` |
| ChatMessage (历史) | 按会话隔离，每个用户最多加载 20 条 | websocket.py 加载历史时也带 user_id 过滤 |
| **memory.md** | **不隔离，全局共享** | `agent_context.py` 读取路径固定为 `{agent_id}/memory/memory.md` |
| soul.md | 不隔离（预期行为，人格是共享的） | 同上 |

## 记忆的数据流

```
用户A 发消息 → websocket.py 接收
    ↓
build_agent_context() 读取同一个 memory.md
    ↓
组装 system prompt（包含 memory 内容）
    ↓
LLM 处理后可能调用 write_file 更新 memory.md
    ↓
用户B 发消息 → build_agent_context() 读到用户A留下的记忆
```

关键代码 `agent_context.py` 约第 169-172 行：

```python
memory = _read_file_safe(ws_root / "memory" / "memory.md", 2000)
# ... 直接拼入 dynamic_parts，不区分当前用户
if memory and memory not in ("_这里记录重要的信息和学到的知识。_", ...):
    dynamic_parts.append(f"\n## Memory\n{memory}")
```

系统提示中有一条指导（`agent_context.py` 约第 413 行）：

> Use `write_file` to update memory/memory.md with important information.

LLM 在任何用户的对话中都可以调用 `write_file` 写入 `memory/memory.md`，且没有任何机制区分写入来源的用户。

## 完整写入链路追踪

记忆写入不是自动的，而是由 LLM 主动调用工具完成。整条链路如下：

### 1. LLM 决定写入记忆

LLM 在任何一轮 tool-calling loop 中（最多 50 轮，`caller.py:344`），如果认为需要记住某些信息，会自主发起一次 `write_file` 或 `edit_file` 工具调用，路径为 `memory/memory.md`。这个决策完全由 LLM 自主做出，系统只通过一条 system prompt 指引（`agent_context.py:413`）：

> Use `write_file` to update memory/memory.md with important information.

**没有任何代码逻辑在 LLM 响应后自动追加记忆** — 记忆更新 100% 依赖 LLM 的工具调用。

### 2. 工具调用经过 `execute_tool` → `_write_file`

```
caller.py:_process_tool_call()
  → agent_tools.py:execute_tool(tool_name, args, agent_id, user_id, session_id)
    → _write_file(ws, rel_path, content, tenant_id)
      → file_path.write_text(content)   # 直接写磁盘，无锁
```

关键观察：

- `execute_tool` 签名中有 `user_id` 和 `session_id` 参数（`agent_tools.py:2113-2118`）
- 但 `_write_file` 内部**完全没有使用 `user_id`**，只用了 `ws`（workspace 路径）和 `rel_path`
- `ws` 由 `ensure_workspace(agent_id)` 生成（`agent_tools.py:2128`），路径为 `AGENT_DATA_DIR/{agent_id}/`，不含任何用户维度
- `rel_path` 由 LLM 在 `arguments` 中指定（如 `"memory/memory.md"`），LLM 自由决定写什么路径

**结论：`user_id` 在整条写入链路中被传递但从未被消费。**

### 3. 记忆读取同样不区分用户

`build_agent_context()` 签名（`agent_context.py:152`）：

```python
async def build_agent_context(
    agent_id: uuid.UUID,
    agent_name: str,
    role_description: str = "",
    current_user_name: str = None   # ← 唯一跟用户有关的参数
) -> tuple[str, str]:
```

`current_user_name` 仅用于在 system prompt 末尾追加一行提示：

```
## Current Conversation
You are currently chatting with **张三**. Address them by name when appropriate.
```

（`agent_context.py:564-565`）

但 memory.md 的读取路径是硬编码的：

```python
memory = _read_file_safe(ws_root / "memory" / "memory.md", 2000)
```

不经过任何用户路由。

### 4. 所有调用路径汇总

| 调用来源 | 是否传 user_id | memory 是否隔离 |
|----------|---------------|----------------|
| WebSocket（Web UI） | ✅ 传真实 user_id | ❌ 共享 |
| Feishu/DingTalk/WeCom/Slack | ✅ 传 platform_user_id | ❌ 共享 |
| Trigger Daemon（自主唤醒） | 传 `agent.creator_id` | ❌ 共享 |
| Gateway（A2A 通信） | 传 `target_creator_id` | ❌ 共享 |
| Heartbeat | 传 `agent.creator_id` | ❌ 共享 |

所有路径最终都进入同一个 `call_llm()` → `build_agent_context()` → 读同一个 `memory.md`。

## 能否插入用户维度的区分？

### 方案一：在 `_write_file` 层拦截（最小改动）

当前 `_write_file` 已经对 `enterprise_info` 路径做了特殊处理（按 `tenant_id` 路由到不同目录），可以复用同样的模式：

```python
# agent_tools.py:_write_file()
if rel_path.startswith("memory/"):
    # 将 memory/ 路由到 memory/{user_id}/
    file_path = (ws / "memory" / str(user_id) / rel_path[len("memory/"):]).resolve()
```

优点：只改一处，所有调用方（Web、IM、Trigger、A2A）自动生效。
缺点：LLM 不知道文件路径变了，`read_file("memory/memory.md")` 可能读不到按 user_id 分区的文件。

### 方案二：在 `build_agent_context` 层路由读取 + 在 `_write_file` 层路由写入（配合改动）

**读取侧**（`agent_context.py:170`）：

```python
# 将 user_id 传入 build_agent_context
memory = _read_file_safe(ws_root / "memory" / str(user_id) / "memory.md", 2000)
```

**写入侧**（`agent_tools.py:_write_file`）：

```python
if rel_path.startswith("memory/") and user_id:
    file_path = (ws / "memory" / str(user_id) / rel_path[len("memory/"):]).resolve()
```

**问题**：LLM 发起的 `write_file(path="memory/memory.md")` 会被路由到 `memory/{user_id}/memory.md`，但 LLM 自己用 `read_file(path="memory/memory.md")` 时也需要被同样路由 — 这意味着 `_read_file` 也要改。

### 方案三：保留共享记忆 + 增加用户专属记忆分区（推荐）

不改现有 `memory/memory.md` 的共享语义（组织知识继续共享），而是在 memory 目录下新增 `users/` 分区：

```
memory/
├── memory.md              ← 组织共享记忆（不变）
├── users/
│   ├── {user_id_A}.md     ← 用户 A 的专属记忆
│   └── {user_id_B}.md     ← 用户 B 的专属记忆
└── reflections.md
```

实现要点：

1. `build_agent_context` 接收 `user_id` 参数，额外读取 `memory/users/{user_id}.md`
2. 在 system prompt 中区分两种记忆：
   ```
   ## Shared Memory (organization knowledge)
   {memory.md 内容}

   ## Personal Notes (about this user)
   {memory/users/{user_id}.md 内容}
   ```
3. 在 `_write_file` / `_edit_file` 中拦截 `memory/users/` 路径写入，防止 LLM 越权写入其他用户的文件
4. LLM 通过 prompt 指引自行判断信息应写入共享记忆还是用户专属记忆

优点：
- 向后兼容，现有记忆不变
- LLM 自主判断信息归属（用户偏好 → users/，组织知识 → memory.md）
- 最小侵入性，不需要修改数据模型或数据库
- 前端 FileBrowser 已经支持浏览 memory/ 目录，新分区自动可见

缺点：
- LLM 的判断不一定准确（可能把隐私信息误写入共享记忆）
- `user_id` 需要被传递到 `build_agent_context`（当前签名没有这个参数，需新增）
- `_read_file` / `_write_file` 中需要新增安全边界检查

### 方案三的改动点清单

| 文件 | 改动 |
|------|------|
| `agent_context.py` | `build_agent_context` 新增 `user_id` 参数，额外读取 `memory/users/{user_id}.md`；system prompt 区分 Shared Memory 和 Personal Notes |
| `agent_tools.py` | `_write_file` / `_edit_file` / `_read_file` 拦截 `memory/users/` 路径，校验只能操作 `memory/users/{current_user_id}.md` |
| `agent_context.py` | system prompt 中的指引更新：告知 LLM 两种记忆路径的用途 |
| `caller.py` | `call_llm` 中传递 `user_id` 到 `build_agent_context`（当前已传 `current_user_name`，需额外传原始 `user_id`） |
| `agent_seeder.py` | 初始化时创建 `memory/users/` 目录 |

## 具体影响场景

### 1. 记忆污染/串扰

用户 A 告诉 Agent "我的项目代号是 Phoenix"，Agent 写入 memory.md。用户 B 对话时也会在 system prompt 中看到 "Phoenix" 这个项目信息，可能产生混淆或误用。

### 2. 写入冲突（竞态条件）

如果用户 A 和用户 B 几乎同时与 Agent 对话，两个并发的 LLM 调用都可能决定更新 memory.md：

```
用户A的LLM: 读取 memory.md → 决定追加内容 → write_file("memory.md", 新内容A)
用户B的LLM: 读取 memory.md → 决定追加内容 → write_file("memory.md", 新内容B)
```

`write_file` 是全量覆盖（不是 append），后写入的会覆盖先写入的。`edit_file` 虽然可以做增量修改，但同样没有文件锁，两个并发 edit 可能互相覆盖。

### 3. 记忆容量被挤占

memory.md 读取时截断为 2000 字符（`_read_file_safe(ws_root / "memory" / "memory.md", 2000)`）。多用户的信息不断写入后，2000 字符可能装不下所有内容，较早的记忆会被截断丢失。

### 4. 隐私泄露

用户 A 的个人信息、偏好、项目细节被 Agent 记入 memory.md 后，用户 B 的对话中会通过 system prompt 间接暴露这些信息。

## 设计意图 vs 实际效果

这个设计是有意为之 — Clawith 定位 Agent 为"组织内的数字员工"，记忆共享意味着 Agent 对整个组织有统一的知识积累，类似于一个新员工了解公司上下文。`ARCHITECTURE_SPEC_EN.md` 中明确说：

> Each agent has a `soul.md` (personality), `memory.md` (long-term memory)... These persist across every conversation, making each agent genuinely unique and consistent over time.

但在多用户场景下缺少了：

- **写入隔离** — 至少按 user_id 分文件存储个人记忆
- **并发控制** — 文件锁或队列化写入防止竞态覆盖
- **容量管理** — 智能摘要/淘汰策略应对多用户信息膨胀
- **隐私边界** — 区分组织公共知识和用户私有信息
