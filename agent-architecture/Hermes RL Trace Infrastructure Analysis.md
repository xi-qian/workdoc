# Hermes RL Trace 基础设施分析

## Intuition

Hermus 有一套完整的 agent trace 收集管线，离可用的 RL 训练数据管线只差一步——奖励信号。类似于一个工厂已经建好了生产线（轨迹采集 → 格式化 → 压缩 → 训练框架），但缺少质检环节（怎么判断一条轨迹是好是坏）。

## 整体管线

```
数据集（JSONL prompt）
    │
    ▼
batch_runner.py（多进程并行）
    │
    ├─ AIAgent(save_trajectories=True)
    │      │
    │      ▼
    │  完整对话 → _convert_to_trajectory_format()
    │      │         │
    │      │         ▼
    │      │   ShareGPT 格式：system → human → gpt → tool → gpt → ...
    │      │   + tool_stats（每个工具的调用/成功/失败）
    │      │   + reasoning_stats（推理覆盖率）
    │      │   + completed / api_calls / toolsets_used
    │      │
    │      ▼
    │  trajectory.py 持久化
    │      ├─ trajectory_samples.jsonl（成功）
    │      └─ failed_trajectories.jsonl（失败）
    │
    ▼
trajectory_compressor.py（后处理）
    │
    ├─ Token 预算裁剪
    ├─ 工具结果截断
    └─ 输出训练就绪的轨迹
    │
    ▼
tinker-atropos/（RL 训练框架）
    │
    ▼
训练产出

                ┌──────────────────────────┐
                │  缺失环节：Reward Signal   │
                │                          │
                │  轨迹 → ？？？ → 训练     │
                │         ↑                │
                │   没有自动化评分机制       │
                └──────────────────────────┘
```

## 已有的数据采集能力

### 1. 轨迹收集

**batch_runner.py**（1302 行）是核心数据生成管线。

**输入**：JSONL 数据集，每行一个 `prompt` 字段。

**运行方式**：
- 多进程并行（`multiprocessing.Pool`，可配置 worker 数）
- 支持断点续跑（`checkpoint.json`）
- 支持从分布中采样（不同的 model、provider、toolset 组合）

**输出**：`trajectories.jsonl`，每行一条轨迹，结构如下：

```json
{
  "conversations": [
    {"from": "system", "value": "..."},
    {"from": "human", "value": "..."},
    {"from": "gpt", "value": "...<tool_call_xml>...</tool_call_xml>"},
    {"from": "tool", "value": "...<tool_result tool_call_id='...' name='...' content='...'/>"},
    {"from": "gpt", "value": "最终回答"}
  ],
  "timestamp": "2026-05-10T...",
  "model": "deepseek-chat",
  "completed": true,
  "partial": false,
  "api_calls": 5,
  "toolsets_used": ["terminal", "web", "file"],
  "tool_stats": {
    "terminal": {"count": 3, "success": 3, "failure": 0},
    "write_file": {"count": 1, "success": 1, "failure": 0},
    "web_search": {"count": 2, "success": 1, "failure": 1}
  },
  "tool_error_counts": {"web_search": 1},
  "reasoning_stats": {
    "total_assistant_turns": 4,
    "turns_with_reasoning": 3,
    "turns_without_reasoning": 1
  }
}
```

**`_convert_to_trajectory_format()`**（`run_agent.py` 第 4300-4467 行）：
- 把 OpenAI 格式的 `messages` 列表转换为 ShareGPT 格式
- reasoning 内容用 `1976` 标记包裹
- 工具调用用尖括号 XML 标记：`<tool_call tool_call_id="..." name="...">...</tool_call_call>`
- 工具结果用 `<tool_result tool_call_id="..." name="..." content="..."/>` 标记
- 完整保留了 (observation → action → outcome) 链条

**`_extract_tool_stats()`**（`batch_runner.py` 第 125-205 行）：
- 遍历消息历史，匹配 assistant 的 `tool_calls` 和 tool 响应
- 通过检查 error 字段、`"success": false` 模式、空内容来分类失败
- 产出每个工具的 `{count, success, failure}` 统计

**`_extract_reasoning_stats()`**（`batch_runner.py` 第 208-241 行）：
- 统计有/无推理的 assistant 轮次
- 覆盖原生 thinking tokens 和 XML scratchpad 两种推理形式

### 2. 专用 RL 工具链

| 文件 | 用途 | 关键细节 |
|------|------|---------|
| `rl_cli.py` | RL 专用 CLI（447 行） | 默认 `save_trajectories=True`，使用 RL 系统提示（引导 agent 发现环境、检查数据、配置训练、测试推理、然后训练），工具集 `["terminal", "web", "rl"]` |
| `tools/rl_training_tool.py` | RL 训练工具集 | 环境发现（AST 扫描 Python 代码中的 gym.Env 子类）、配置管理、训练生命周期控制、WandB 监控 |
| `tinker-atropos/` | RL 训练框架 | Git submodule，需 `git submodule update --init`，目前目录为空 |

### 3. 会话数据库（state.db）

SQLite 数据库，位于 `~/.hermes/state.db`，提供全量历史数据。

**关键表**：

| 表 | RL 相关列 |
|-----|---------|
| `sessions` | `id`, `source`, `model`, `started_at`, `ended_at`, `end_reason`, `message_count`, `tool_call_count`, `input_tokens`, `output_tokens`, `cache_read_tokens`, `cache_write_tokens`, `reasoning_tokens`, `estimated_cost_usd`, `actual_cost_usd` |
| `messages` | `session_id`, `role`, `content`, `tool_call_id`, `tool_calls`（JSON）, `tool_name`, `timestamp`, `token_count`, `finish_reason`, `reasoning`, `reasoning_content`, `reasoning_details` |
| `messages_fts` | FTS5 全文搜索索引 |
| `messages_fts_trigram` | CJK 友好的 trigram 搜索索引 |

**RL 用途**：
- `end_reason` / `finish_reason` 可作为粗糙的完成信号
- token 和 cost 数据可用于效率指标
- 全量消息历史可重建完整轨迹
- FTS 索引可查询错误模式

### 4. 子智能体 Trace

**`delegate_tool.py`** 第 1619-1694 行：

子智能体系统构建 `tool_trace` 列表，通过 `tool_call_id` 配对工具调用和结果：

```python
tool_trace = [
    {"tool": "read_file", "args_bytes": ..., "result_bytes": ..., "status": "ok"},
    {"tool": "terminal", "args_bytes": ..., "result_bytes": ..., "status": "error"},
    ...
]
```

结构足够用于分析子智能体的行为模式。

### 5. Langfuse 观测插件

**`plugins/observability/langfuse/`**

这是最 RL-ready 的观测机制，对每次 agent 轮次创建结构化 trace：

| Hook | 采集内容 |
|------|---------|
| `pre_llm_call` / `post_llm_call` | 输入消息（完整序列化）、输出（content + reasoning + tool_calls）、usage（input/output/cache_read/cache_write/reasoning tokens）、cost、finish_reason、api_duration_s |
| `pre_tool_call` / `post_tool_call` | tool_name、args、result、tool_call_id（结果做了归一化处理，如 read_file 只保留 head/tail） |

根 trace 元数据：`{source: "hermes", task_id, platform, provider, model, api_mode}`

支持采样率配置（`HERMES_LANGFUSE_SAMPLE_RATE`），需要 Langfuse 账号。

### 6. 错误分类

**`agent/error_classifier.py`**

结构化的错误分类体系（`FailoverReason` 枚举）：

| 分类 | 说明 |
|------|------|
| `auth` | 认证失败 |
| `billing` | 额度不足 |
| `rate_limit` | 限流 |
| `overloaded` | 服务端过载 |
| `server_error` | 服务端错误 |
| `timeout` | 超时 |
| `context_overflow` | 上下文溢出 |
| `payload_too_large` | 请求体过大 |
| `image_too_large` | 图片过大 |
| `model_not_found` | 模型不存在 |
| `format_error` | 格式错误 |

每个错误有结构化的 `ClassifiedError` 数据类：`reason`, `status_code`, `provider`, `model`, `message`, `error_context`, `retryable`, `should_compress`。

虽然主要用于 failover/retry 决策，但这个分类可以直接用于轨迹质量评估。

### 7. Kanban 任务完成信号

**`plugins/kanban/`**

任务生命周期：`triage` → `todo` → `ready` → `running` → `done` / `blocked`

`task_runs` 表记录每次尝试：`status`, `outcome`, `summary`, `error`, `worker_pid`, `started_at`, `ended_at`, `metadata`

`task_events` 表是 append-only 事件日志，有 `kind` 和 `payload`。

还有诊断规则引擎检测：幻觉、崩溃、spawn 失败、stuck-blocked 状态。

任务完成状态（done vs blocked）和错误分类可作为任务级奖励信号。

### 8. Achievements 插件

**`plugins/hermes-achievements/`**

扫描会话历史，计算丰富的行为指标：

- tool_call_count、distinct_tool_count、error_count
- files_touched、terminal/web/browser/patch/file_read 计数
- traceback_events、port_conflict_events、permission_denied_events
- 更多基于正则的事件计数器

这些指标存储在 JSON 快照文件中，不是可查询的数据库，但内容很丰富。

## 数据完备性评估

| 数据类型 | 是否有 | 来源 | 格式 | 结构化程度 |
|---------|-------|------|------|-----------|
| 完整对话轨迹 | ✅ | batch_runner.py, trajectory.py | ShareGPT JSONL | 高 |
| 工具调用 trace（name/args/result） | ✅ | run_agent.py, delegate_tool.py | 嵌入轨迹 | 高 |
| 工具成功/失败统计 | ✅ | batch_runner.py | per-trajectory JSON | 高 |
| 推理覆盖统计 | ✅ | batch_runner.py | per-trajectory JSON | 高 |
| LLM 调用 trace（tokens/cost/latency） | ✅ | Langfuse 插件 | Langfuse 平台 | 高 |
| 会话消息历史 | ✅ | state.db | SQLite 行 | 高 |
| 会话级 token/cost | ✅ | state.db sessions 表 | DB 列 | 高 |
| 错误分类 | ✅ | error_classifier.py | 结构化枚举 | 高 |
| 任务完成结果 | ✅ | kanban task_runs | DB 行 | 高 |
| **用户反馈/奖励** | **❌** | — | — | — |
| 工具级延迟 | ⚠️ | 仅 Langfuse | api_duration_s | 中 |
| RL 训练编排 | ✅ | rl_cli.py + rl_training_tool.py | 子进程 | 中 |

## 关键缺失：奖励信号

### 没有任何用户反馈收集机制

- **没有 thumbs up/down**
- 网关平台的 emoji 反应（👀 → ✅ / ❌）是 **bot 发给用户**的状态指示，不是用户反馈
- 用户的 emoji 回复没有被监听或存储
- 没有显式的质量评分或偏好收集

### 奖励信号的潜在来源

| 来源 | 类型 | 可行性 | 说明 |
|------|------|--------|------|
| 测试用例通过/失败 | 自动化 | 高 | 代码任务跑测试，类似 SWE-bench |
| 工具调用成功/失败 | 自动化 | 高 | 已有 tool_stats，可作为过程奖励 |
| 任务完成状态 | 自动化 | 中 | kanban 的 done/blocked |
| LLM-as-Judge | 自动化 | 中 | 用另一个 LLM 评价轨迹质量 |
| 用户显式反馈 | 人工 | 低 | 需要新增 UI 交互 |
| 平台 emoji 回采 | 半自动 | 低 | 需要改网关 adapter 监听用户反应 |

### 补全管线需要的改动

```
现有：
  dataset → batch_runner → trajectories.jsonl + statistics.json
  trajectory_compressor.py → 训练就绪的轨迹
  tinker-atropos/ → RL 训练

需要新增：
  1. 轨迹评分器（Reward Scorer）
     - 代码任务：跑测试用例，pass/fail → reward: 0/1
     - 工具任务：tool_stats.success_rate → process reward
     - 通用：LLM-as-Judge → reward: 0-1

  2. 奖励字段注入
     - 在 trajectory JSONL 中增加 "reward": float
     - 在 batch_runner 输出中增加 reward 计算步骤

  3. 用户反馈通道
     - 网关平台：监听用户对 bot 消息的 emoji 反应
     - CLI：增加 /rate 命令或 /good /bad 快捷反馈
     - API server：增加 POST /v1/feedback 端点

  4. 反馈持久化
     - state.db 增加 feedback 表：session_id, message_id, rating, user_id, timestamp
     - 或写入 trajectory JSONL 的 reward 字段
```

## 设计评价

### 做得好的部分

**1. ShareGPT 格式标准化**

轨迹输出直接兼容 ShareGPT 格式，这是 RLHF/GRPO 训练的主流格式。不需要额外转换就能接入大多数训练框架。

**2. 工具统计的精细度**

每条轨迹都有 per-tool 的 `{count, success, failure}`，这可以直接用作过程奖励（process reward）——不需要等到任务结束才知道好不好，每一步工具调用的成败都是信号。

**3. 推理覆盖统计**

reasoning_stats 让你能区分"模型认真思考了再行动"和"模型直接跳到结论"。对于训练"会思考的 agent"，这是重要的过滤维度。

**4. 成功/失败轨迹分离**

`trajectory_samples.jsonl` 和 `failed_trajectories.jsonl` 分开存储，天然就是正负样本划分。虽然"失败"不一定是轨迹质量差（可能是任务本身很难），但提供了基本的分类。

**5. Langfuse 的全链路 trace**

如果启用了 Langfuse，每个 LLM 调用和工具调用都有完整的 span，包括延迟、token、cost。这比 batch_runner 的统计更精细，可以做到单步级别的分析。

### 核心问题

**1. 奖励信号完全缺失**

这是唯一的关键缺失。没有奖励信号，收集再多轨迹也只能做 SFT（监督微调），不能做 RL。而 SFT 只能学到"模仿"，不能学到"更好"。

**2. tinker-atropos 子模块为空**

`git submodule update --init` 没有被执行过，实际的 RL 训练框架不存在。这意味着 RL 管线的最后一段（训练本身）是不可用的。

**3. 轨迹格式中工具调用用 XML 标记而非结构化 JSON**

`_convert_to_trajectory_format()` 把工具调用编码为尖括号 XML 字符串（`<tool_call name="...">...</tool_call_call>`），嵌入在 `value` 字符串中。这意味着：
- 需要 XML 解析才能提取单个工具调用
- 与标准的 OpenAI function calling JSON 格式不兼容
- 多模态训练框架可能需要额外预处理

**4. 没有环境状态（observation）的标准化编码**

轨迹记录了对话文本，但没有记录环境状态的结构化表示。比如终端命令执行后的结果是什么、文件写入前的内容是什么。这些信息嵌在 tool_result 的字符串里，不是结构化的 observation。

**5. 断点续跑但不支持增量评分**

batch_runner 支持断点续跑（`checkpoint.json`），但评分/奖励信号需要在轨迹生成时同步计算，不能事后追加。如果评分逻辑加入得太晚，已有的轨迹需要全部重新跑。

## 改进建议

### 短期：补全奖励信号

**自动化评分器**：

```
batch_runner.py 输出轨迹
        │
        ▼
reward_scorer.py（新增）
        │
        ├─ 代码任务：运行测试 → pass/fail → reward: {0.0, 1.0}
        ├─ 工具任务：tool_stats → success_rate → reward: [0.0, 1.0]
        └─ 通用任务：LLM-as-Judge → reward: [0.0, 1.0]
        │
        ▼
轨迹 JSONL 增加 "reward" 字段
```

**用户反馈通道**：
- 网关平台：监听用户对 bot 消息的 emoji（👍/👎）回采为 reward
- CLI：增加 `/good` `/bad` 命令
- API server：增加 `POST /v1/feedback` 端点

### 中期：结构化 observation

- 工具调用结果从 XML 字符串改为结构化 JSON（保留文本用于 SFT，增加结构化字段用于 RL）
- 增加 `environment_state` 字段：当前工作目录、文件列表、git status 等环境快照

### 长期：在线学习闭环

- 网关部署中，实时采集用户反馈作为 reward signal
- 定期用新轨迹 + reward 重训模型
- 新模型部署后，通过 A/B 测试验证改进
- 形成"部署 → 采集 → 训练 → 部署"的闭环
