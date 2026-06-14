# Version 1.4: Agent-runner Poll Loop

## Goal

Make agent-runner capable of running as a warm process that polls DB-backed inbound messages, invokes the selected provider, writes outbound messages, and idles until stopped.

## Non-goals

- Do not require single-container runtime yet.
- Do not require OpenCode yet.
- Do not remove single-shot mode.
- Do not remove legacy file tool IPC yet.

## Modes

Agent-runner supports three modes:

```text
legacy-single-shot
live
isolated-task
```

`legacy-single-shot` keeps current stdin/stdout behavior.

`live` runs until idle timeout or close signal.

`isolated-task` processes one task prompt with a fresh runtime directory, writes result, and exits.

## CLI and Environment

Preferred CLI:

```bash
agent-runner \
  --tenant acme \
  --agent finance \
  --mode live \
  --runtime-dir /runtime/tenants/acme/agents/finance/live \
  --cwd /workspace/tenants/acme/agents/finance
```

Environment fallback:

```env
NANOCLAW_TENANT=acme
NANOCLAW_AGENT=finance
NANOCLAW_RUN_MODE=live
NANOCLAW_RUNTIME_DIR=/runtime/tenants/acme/agents/finance/live
NANOCLAW_CWD=/workspace/tenants/acme/agents/finance
```

## Poll Loop

Pseudo-code:

```ts
while (!stopping) {
  const batch = claimPendingInbound(runtimeDir, maxMessages);
  if (batch.length === 0) {
    await sleep(pollInterval);
    continue;
  }

  const prompt = formatMessages(batch);
  const continuation = state.get(`continuation:${providerName}`);

  const query = provider.query({
    prompt,
    continuation,
    cwd,
    systemContext
  });

  for await (const event of query.events) {
    if (event.type === "init") state.set(`continuation:${providerName}`, event.continuation);
    if (event.type === "result" && event.text) writeOutbound(event.text);
    if (event.type === "error") writeOutboundError(event);
  }

  markInboundCompleted(batch);
}
```

## Follow-up Messages

While a provider query is active, the loop should keep checking for newly pending inbound messages.

Initial behavior:

- Claude provider: call `query.push(message)`.
- Providers without push semantics: mark follow-up pending and let the next loop process it after current query completes.

Later behavior:

- OpenCode provider may abort and restart internally.

## Idle and Stop

The runner exits when:

- mode is `isolated-task` and the first batch completes
- mode is `live` and idle timeout expires
- supervisor sends stop
- close marker is received through runtime DB state

Close should be represented in DB:

```sql
INSERT OR REPLACE INTO kv(key, value, updated_at)
VALUES ('control:stop', 'true', datetime('now'));
```

Legacy `_close` file can still be bridged to this state key.

## Isolated Task Semantics

An isolated task:

- uses the same tenant/agent identity
- runs as the same future Linux user
- uses a unique run directory
- does not read live chat history
- does not reuse live continuation
- exits after result or error

Directory:

```text
runs/<taskId>-<timestamp>/
  inbound.db
  outbound.db
  state.db
```

## Provider State

Continuations are per provider:

```text
continuation:claude
continuation:opencode
```

This prevents provider switches from reading incompatible session IDs.

## Output Formatting

During compatibility phase, agent-runner can write both:

- `outbound.db`
- stdout sentinel `OUTPUT_START/OUTPUT_END`

The host should prefer `outbound.db` when `NANOCLAW_RUNTIME_DIR` is present.

## Tests

- live mode processes inbound messages and idles.
- isolated-task mode processes exactly one run and exits.
- provider continuation is saved per provider.
- follow-up messages are not lost during active query.
- close control stops a live runner.
- legacy single-shot mode still passes existing tests.

## Acceptance Criteria

- Agent-runner can operate without stdin prompt input.
- Host can enqueue a message after process startup and receive an outbound response.
- isolated task does not modify live continuation.
- Existing Docker per-group runtime can launch either legacy or poll-loop mode.

