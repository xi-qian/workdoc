# Manager Agent 多用户通讯 Memory 分析

> **分析对象**：[HiClaw](https://github.com/alibaba/hiclaw) v1.0.9 — 开源协作式多 Agent 运行时平台
> **分析日期**：2026-04-20
> **相关源码路径**：`manager/agent/SOUL.md`、`manager/agent/AGENTS.md`、`manager/scripts/init/`

## Memory 机制概述

Manager 的 "memory" 不是持久向量记忆，而是**纯文件系统方案**：

```
~/memory/YYYY-MM-DD.md    # 每日原始记录（每次会话启动读取今天+昨天）
~/MEMORY.md                # 长期精选洞察（仅 DM 会话加载）
~/state.json               # 活跃任务注册表
~/workers-registry.json    # Worker 注册信息
```

关键规则（`AGENTS.md:63-66`）：

> You wake up fresh each session. Files are your continuity.

Manager 是**单个 OpenClaw/CoPaw 进程**，所有 Matrix 房间的消息通过 @mention 串行触发。

## 逐项分析

### 1. 会话（session）之间的 memory 断裂 — 主要问题

Manager 每次被唤醒是新 session，靠读文件恢复上下文。多用户场景下：

```
用户 A 在 DM 房间："创建 worker alice 做前端"
  → Manager 写 memory/2026-04-20.md: "创建了 alice"

5 分钟后，用户 B 在另一个 DM 房间："把最近的进度汇总一下"
  → Manager 新 session 启动，读今天 memory，看到 "创建了 alice"
  → 但用户 B 根本不知道 alice 是什么，Manager 也分不清这条记录的归属
```

**问题**：memory 没有 per-user 隔离，所有用户的操作混在同一个日期文件里。Manager 无法区分哪些上下文与当前对话的用户相关。

### 2. MEMORY.md 在 DM 中加载的安全风险 — 已知设计决策

`AGENTS.md:74-75` 明确写了：

> **ONLY load in DM sessions** with the human admin (not in group Rooms with Workers)
> This is for **security** — contains Worker assessments, operational context

但如果**多个 authorized Human 用户**都能 DM Manager（`SOUL.md:44`），那么：

```
用户 A 的 DM → Manager 加载 MEMORY.md → 看到 "用户 B 的代码质量差，建议不要分配关键任务"
用户 B 的 DM → Manager 同样看到这条 → 行为被这条记忆影响
```

MEMORY.md 中的用户评估对所有 DM 用户可见。这不是 bug，是已知的安全 tradeoff——它假设只有一个 human admin。

### 3. state.json / workers-registry.json 的并发写入 — 无问题

Manager 是单进程，OpenClaw 的 @mention 处理是**串行的**，所以不存在真正的文件并发写冲突。竞态窗口分析：

```
@mention 触发 → Manager session 启动 → 读 state.json → 处理 → 写 state.json → 回复

如果在这个窗口内另一个房间的 @mention 到达：
  OpenClaw 会排队，等当前 session 结束后再启动新 session
```

- 两个用户的请求如果依赖同一 state（比如同时要求创建同名 Worker），后处理的会看到前一个的更新，行为是正确的
- 不存在 lost update 问题

### 4. 上下文窗口污染 — 轻微

Manager 的 context window 是**全局共享的**。每个 @mention 触发的 session：

```
Session 1 (用户 A): 读 SOUL.md + memory/2026-04-20.md + 处理用户 A 的请求
Session 2 (用户 B): 读 SOUL.md + memory/2026-04-20.md + 处理用户 B 的请求
```

虽然 session 之间不共享 context，但 memory 文件里混合了所有用户的操作记录。当 memory 文件增长到 200 行限制被截断时，**哪个用户的记忆被丢掉是不可控的**。

### 5. Worker 房间里的隐私边界

```
Room "Worker: Alice" → Human Admin + Manager + Alice
```

如果一个 authorized Human 用户 B 也在这个房间里（`SOUL.md:42` 的 `groupAllowFrom`），用户 B 可以看到 Manager 和 Alice 的所有交互历史。这取决于 Matrix 房间的成员配置，不是 memory 系统的问题。

## 总结

| 问题 | 严重程度 | 原因 |
|------|---------|------|
| memory 无 per-user 隔离 | **中** | 所有用户操作混在同一个日期文件，Manager 无法区分归属 |
| MEMORY.md 对所有 DM 用户可见 | **低** | 已知设计决策，假设单一 admin；多用户时隐私泄露 |
| 文件并发写冲突 | **无** | 单进程串行处理，不存在 race condition |
| memory 截断导致不可控丢失 | **低** | 200 行限制下，混合记录的丢失不可预测 |
| Context window 全局共享 | **无** | 每个 session 独立，不跨 session 共享 context |

## 结论

当前架构**假设单一 human admin + 多个 Worker**。memory 系统没有 per-user 命名空间隔离。如果要支持真正的多用户场景，需要在 `memory/` 目录下加一层用户维度（比如 `memory/{user}/{date}.md`），并在 `AGENTS.md` 的 "Every Session" 流程中加上根据当前房间/发送者加载对应 memory 的逻辑。
