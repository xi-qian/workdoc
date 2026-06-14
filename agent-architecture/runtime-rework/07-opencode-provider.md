# Version 1.7: Provider Abstraction and OpenCode

## Goal

Make the agent runtime provider-pluggable and add OpenCode as a supported provider while keeping Claude as a fallback.

## Non-goals

- Do not require OpenCode as default immediately.
- Do not remove Claude Agent SDK.
- Do not promise perfect Claude Code behavior parity.

## Provider Interface

Use a provider interface similar to NanoClaw 2.0:

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

## Provider Registry

```ts
registerProvider("claude", opts => new ClaudeProvider(opts));
registerProvider("opencode", opts => new OpenCodeProvider(opts));
registerProvider("mock", opts => new MockProvider(opts));
```

Provider name comes from `agent.json`:

```json
{
  "provider": "opencode",
  "model": "anthropic/claude-sonnet-4"
}
```

## OpenCode Provider Design

OpenCode integration is server/client based:

```text
OpenCodeProvider
  ensure server for this run
  create or resume session if supported
  subscribe to event stream
  send prompt
  translate events to ProviderEvent
```

Initial behavior:

- one OpenCode server per agent run
- no cross-agent shared server
- no cross-user shared OpenCode config

This costs more than a shared OpenCode server, but avoids session leakage.

## Continuation

State key:

```text
continuation:opencode
```

The value may be:

- OpenCode session ID
- path to OpenCode session state
- empty if resume is not reliable

If resume is not implemented in the first pass, the provider still emits an `init` continuation for observability, but `maybeRotateContinuation` can remain absent.

## MCP Translation

NanoClaw MCP config:

```ts
{
  command: "node",
  args: ["/app/mcp/server.js"],
  env: { "NANOCLAW_RUNTIME_DIR": "/runtime/..." }
}
```

OpenCode local MCP config:

```ts
{
  type: "local",
  command: ["node", "/app/mcp/server.js"],
  environment: {
    "NANOCLAW_RUNTIME_DIR": "/runtime/..."
  }
}
```

Translation must be deterministic and covered by tests.

## Tool Differences

Claude-specific features that need explicit handling:

- Claude hooks (`PreToolUse`, `PostToolUse`, `PreCompact`) do not map directly.
- Claude native slash commands may not exist.
- Claude subagent/task tool semantics are not identical.
- Claude transcript rotation does not apply as-is.

OpenCode provider should initially support:

- normal text prompts
- filesystem/code tools exposed by OpenCode
- MCP tools required by NanoClaw
- activity events
- result extraction
- abort/stop

## Follow-up Messages

If OpenCode cannot accept `push` into an active query, implement:

```text
push(message):
  store pending follow-up
  abort current query if safe
  restart prompt with pending message after current state settles
```

For first version, it is acceptable to queue follow-ups for the next loop, provided no messages are lost.

## Tests

- provider registry loads `opencode`.
- OpenCode config receives provider/model from agent config.
- MCP config translation is correct.
- provider emits `init`, `activity`, and `result`.
- provider errors become `ProviderEvent.error`.
- continuation is stored under `continuation:opencode`, not `continuation:claude`.
- switching between Claude and OpenCode does not reuse incompatible continuation.

## Acceptance Criteria

- One agent can run with Claude and another with OpenCode.
- OpenCode can answer a simple message through DB IPC.
- isolated task can run through OpenCode.
- Claude remains available as fallback.
- Known behavior differences are documented in provider README or comments.

