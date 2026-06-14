# 版本 1.7：Provider 抽象与 OpenCode

## 目标

使智能体运行时可插拔 Provider，并添加 OpenCode 作为支持的 Provider，同时保留 Claude 作为回退。

## 非目标

- 不要求立即将 OpenCode 设为默认。
- 不移除 Claude Agent SDK。
- 不承诺与 Claude Code 行为完全对等。

## Provider 接口

采用类似 NanoClaw 2.0 的 Provider 接口：

```ts
export interface AgentProvider {
  readonly supportsNativeSlashCommands: boolean;
  query(input: QueryInput): AgentQuery;
  isSessionInvalid(err: unknown): boolean;
  maybeRotateContinuation?(continuation: string, cwd: string): string | null;
}

export interface QueryInput {
  prompt: string;
  continuation?: string;
  cwd: string;
  systemContext?: {
    instructions?: string;
  };
}

export interface AgentQuery {
  push(message: string): void;
  end(): void;
  events: AsyncIterable<ProviderEvent>;
  abort(): void;
}

export type ProviderEvent =
  | { type: "init"; continuation: string }
  | { type: "activity" }
  | { type: "result"; text: string | null }
  | { type: "progress"; message: string }
  | { type: "error"; message: string; retryable: boolean; classification?: string };
```

## Provider 注册

```ts
registerProvider("claude", opts => new ClaudeProvider(opts));
registerProvider("opencode", opts => new OpenCodeProvider(opts));
registerProvider("mock", opts => new MockProvider(opts));
```

Provider 名称来自 `agent.json`：

```json
{
  "provider": "opencode",
  "model": "anthropic/claude-sonnet-4"
}
```

## OpenCode Provider 设计

OpenCode 集成基于服务端/客户端架构：

```text
OpenCodeProvider
  ensure server for this run
  create or resume session if supported
  subscribe to event stream
  send prompt
  translate events to ProviderEvent
```

初始行为：

- 初始阶段，每次分组运行使用独立的 OpenCode 服务端
- 不跨智能体共享服务端
- 不跨用户共享 OpenCode 配置

这比共享 OpenCode 服务端开销更大，但可避免分组会话泄漏。

## 续接

状态键：

```text
continuation:opencode
```

值可以是：

- OpenCode 会话 ID
- OpenCode 会话状态路径
- 若恢复不可靠则为空

如果第一轮未实现恢复功能，Provider 仍会发出 `init` 续接以便可观测，但 `maybeRotateContinuation` 可保持未定义。

## MCP 翻译

NanoClaw MCP 配置：

```ts
{
  command: "node",
  args: ["/app/mcp/server.js"],
  env: { "NANOCLAW_RUNTIME_DIR": "/runtime/..." }
}
```

OpenCode 本地 MCP 配置：

```ts
{
  type: "local",
  command: ["node", "/app/mcp/server.js"],
  environment: {
    "NANOCLAW_RUNTIME_DIR": "/runtime/..."
  }
}
```

翻译必须是确定性的，并有测试覆盖。

## 工具差异

需要显式处理的 Claude 特有功能：

- Claude 钩子（`PreToolUse`、`PostToolUse`、`PreCompact`）无法直接映射。
- Claude 原生斜杠命令可能不存在。
- Claude 子智能体/任务工具语义不完全相同。
- Claude 转录轮换无法直接套用。

OpenCode Provider 初始应支持：

- 普通文本提示
- OpenCode 暴露的文件系统/代码工具
- NanoClaw 所需的 MCP 工具
- 活动事件
- 结果提取
- 中止/停止

## 后续消息

如果 OpenCode 无法接受 `push` 到活跃查询中，则实现：

```text
push(message):
  store pending follow-up
  abort current query if safe
  restart prompt with pending message after current state settles
```

首个版本中，将后续消息排队到下一个轮询循环是可接受的，前提是不丢失任何消息。

## 测试

- Provider 注册加载 `opencode`。
- OpenCode 配置从智能体配置接收 provider/model。
- MCP 配置翻译正确。
- Provider 发出 `init`、`activity` 和 `result`。
- Provider 错误变为 `ProviderEvent.error`。
- 续接存储在 `continuation:opencode` 下，而非 `continuation:claude`。
- 在 Claude 和 OpenCode 之间切换时不会复用不兼容的续接。

## 验收标准

- 一个智能体可以使用 Claude 运行，另一个使用 OpenCode。
- OpenCode 可以通过 DB IPC 回复简单消息。
- 隔离任务可以通过 OpenCode 运行。
- Claude 仍可作为回退。
- 已知的行为差异记录在 Provider README 或注释中。
