# Webhook Task Trigger Design

## Overview

Add a generic webhook endpoint to NanoClaw that triggers task execution via HTTP. The endpoint receives a URL with a task name, reads task configuration from a JSON file, interpolates URL query parameters into the prompt template, and executes the task through the existing scheduler infrastructure.

## Architecture

The feature extends the existing Feishu webhook HTTP server with additional routes. No new HTTP server is needed.

```
POST /webhook/task/{name}?param1=value1&param2=value2
  → Feishu HTTP server receives request
  → Path doesn't match /feishu/webhook → delegate to webhook task handler
  → Load webhook-tasks.json, find task by name
  → Interpolate URL query params into prompt template
  → Create a one-shot ScheduledTask in the database
  → Task picked up by scheduler loop, executed via runTask()
  → Return 200 with { taskId }
```

## Configuration File

Location: `webhook-tasks.json` in project root.

```json
{
  "tasks": [
    {
      "name": "daily-report",
      "path": "/webhook/task/daily-report",
      "group_folder": "oc_da0cdb996931bf98d2ba38a92ce34979",
      "prompt": "请生成 {date} 的周报摘要",
      "context_mode": "group"
    },
    {
      "name": "cleanup",
      "path": "/webhook/task/cleanup",
      "group_folder": "oc_da0cdb996931bf98d2ba38a92ce34979",
      "prompt": "清理过期文件",
      "context_mode": "isolated"
    }
  ]
}
```

Fields:

| Field | Required | Description |
|-------|----------|-------------|
| `name` | yes | Task identifier, used in URL path |
| `path` | yes | URL path to match (e.g., `/webhook/task/daily-report`) |
| `group_folder` | no | Target group folder. Defaults to the main group (isMain=true) if not specified |
| `prompt` | yes | Prompt template. `{paramName}` placeholders are replaced with URL query parameters |
| `context_mode` | no | `group` (default) or `isolated` |

## Parameter Interpolation

- URL query parameters are extracted from the request URL
- Each `{paramName}` in the prompt template is replaced with the corresponding query parameter value
- Unprovided parameters remain as `{paramName}` literal
- Extra parameters not in the template are ignored
- Example: `POST /webhook/task/daily-report?date=2024-01-01` → prompt becomes `"请生成 2024-01-01 的周报摘要"`

## HTTP API

### Trigger a task

```
POST /webhook/task/{name}?param1=value1
```

**Response (200):**
```json
{
  "ok": true,
  "taskId": "webhook_daily-report_1716000000000",
  "message": "Task triggered"
}
```

**Response (404) — task not found:**
```json
{
  "ok": false,
  "error": "Task not found: unknown-task"
}
```

**Response (500) — internal error:**
```json
{
  "ok": false,
  "error": "Failed to trigger task"
}
```

## Changes

### New file: `src/webhook-tasks.ts`

- `loadWebhookTasks()`: Read and parse `webhook-tasks.json`
- `matchWebhookTask(url)`: Match request URL to configured task path
- `interpolatePrompt(template, params)`: Replace `{param}` placeholders with query params
- `handleWebhookTaskRequest(req, res, deps)`: Main handler — match task, interpolate, create ScheduledTask, respond

### Modified: `src/feishu/client.ts`

- In `handleWebhookRequest()`: when request doesn't match Feishu webhook path, delegate to webhook task handler
- The webhook task handler needs access to `SchedulerDependencies` — passed during FeishuClient initialization or via a callback

### Modified: `src/config.ts`

- Add `WEBHOOK_TASKS_CONFIG` path constant (defaults to `webhook-tasks.json` in project root)

### New file: `webhook-tasks.json`

- Default empty configuration: `{ "tasks": [] }`

### Modified: `src/index.ts`

- Pass scheduler dependencies to FeishuClient so the webhook task handler can create and trigger tasks
