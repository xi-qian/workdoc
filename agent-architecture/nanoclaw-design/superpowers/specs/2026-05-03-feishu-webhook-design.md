# Feishu Webhook 回调模式设计

## 概述

当前 Feishu 通道仅支持 WebSocket 长连接接收事件。本次新增 Webhook 回调模式作为可选替代，两种模式共享消息处理、发送、API 调用等逻辑，仅连接层不同。

## 配置

### credentials.json 新增字段

`store/auth/feishu/credentials.json` 新增 `mode` 和 `webhook` 字段：

```json
{
  "appId": "...",
  "appSecret": "...",
  "mode": "websocket",
  "webhook": {
    "host": "127.0.0.1",
    "port": 8080,
    "path": "/feishu/webhook",
    "encryptKey": "",
    "verificationToken": ""
  }
}
```

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `mode` | `"websocket" \| "webhook"` | `"websocket"` | 连接模式 |
| `webhook.host` | string | `"127.0.0.1"` | HTTP Server 监听地址 |
| `webhook.port` | number | `8080` | HTTP Server 监听端口 |
| `webhook.path` | string | `"/feishu/webhook"` | 回调 URL 路径 |
| `webhook.encryptKey` | string | `""` | 加密密钥，为空则明文模式 |
| `webhook.verificationToken` | string | `""` | URL 验证 Token |

### 环境变量降级

为方便容器化部署，这些字段也支持环境变量：

- `FEISHU_MODE=webhook` — 等价于 `mode: "webhook"`
- `FEISHU_WEBHOOK_HOST` — `webhook.host`
- `FEISHU_WEBHOOK_PORT` — `webhook.port`
- `FEISHU_WEBHOOK_PATH` — `webhook.path`
- `FEISHU_WEBHOOK_ENCRYPT_KEY` — `webhook.encryptKey`
- `FEISHU_WEBHOOK_VERIFICATION_TOKEN` — `webhook.verificationToken`

优先级：环境变量 > credentials.json

## 类型变更

### `FeishuCredentials`（`src/feishu/types.ts` 和 `src/feishu/auth.ts`）

```typescript
export interface FeishuCredentials {
  appId: string;
  appSecret: string;
  accessToken?: string;
  refreshToken?: string;
  expiresAt?: string;
  tenantKey?: string;
  // 新增
  mode?: 'websocket' | 'webhook';
  webhook?: {
    host?: string;
    port?: number;
    path?: string;
    encryptKey?: string;
    verificationToken?: string;
  };
}
```

## HTTP Server 职责（webhook 模式）

### 1. URL 验证

飞书配置事件订阅时发送 `POST` 请求，body：`{"type": "url_verification", "challenge": "...", "token": "..."}`。验证 token 匹配后返回 `{"challenge": "..."}`。token 为空则不校验。

### 2. 明文事件接收

`POST {path}`，Content-Type: `application/json`。飞书 webhook 推送的事件 body 格式为：

```json
{
  "schema": "2.0",
  "header": {
    "event_id": "xxx",
    "event_type": "im.message.receive_v1",
    "token": "..."
  },
  "event": {
    "sender": {...},
    "message": {...}
  }
}
```

事件类型从 `header.event_type` 提取，事件体从 `event` 字段提取。对 `FeishuChannel` 已有的消息处理回调，构造的 `FeishuEvent` 结构保持一致：`{ type: header.event_type, event: body.event }`，使得 `setupEventHandlers()` 中已有的解析逻辑完全复用。

### 3. 加密事件处理

如果 `encryptKey` 非空，HTTP body 为：`{"encrypt": "base64..."}`。解密流程：

```
JSON.parse(body).encrypt → Base64 decode → AES-256-CBC decrypt → UTF-8 → JSON.parse → 明文事件对象
```

解密参数：
- key: `SHA256(encryptKey)` 的前 16 字节
- iv: 密文数据（Base64 解码后）的前 16 字节
- 密文: 密文数据从第 17 字节开始

解密后得到的就是上面的明文事件 JSON，后续处理与明文路径一致。

### 4. 事件分发

提取 `header.event_type` 后，构造 `{ type: eventType, event: body.event }`，调用 `emit(eventType, feishuEvent)`。支持的事件类型与 WebSocket 一致：

| header.event_type | emit eventType |
|-------------------|----------------|
| `im.message.receive_v1` | `im.message.receive_v1` |
| `im.message.message_read_v1` | `im.message.message_read_v1` |
| `im.message.reaction.created_v1` | `im.message.reaction.created_v1` |
| `card.action.trigger` | `card.action.trigger` |

未知事件类型仅记录日志，不做分发。

## 代码改动

### `src/feishu/client.ts`

1. `connect()` 方法分支：
   - `mode === 'webhook'` → `this.startWebhookServer()`
   - 否则 → 现有 WebSocket 逻辑（不变）

2. 新增私有方法：
   - `startWebhookServer()` — 创建 `http.createServer`，注册路由
   - `handleWebhookRequest(req, res)` — 请求处理入口
   - `handleUrlVerification(req, res)` — URL 验证
   - `handleEventCallback(req, res)` — 事件回调处理
   - `decryptEvent(encryptedBase64)` — AES 解密
   - `parseEvents(plainJson)` — 解析事件 JSON 数组/单个对象

3. `isConnected()` — webhook 模式返回 `server?.listening ?? false`

4. `disconnect()` — webhook 模式调 `server.close()`

### `src/channels/feishu.ts`

`FeishuChannel` 无需改动。`setupEventHandlers()` 中注册的 `client.on(...)` 在两种模式下都通过 `emit()` 触发。

### `src/feishu/auth.ts`

`loadCredentials()` 在返回前合并环境变量：

```typescript
// webhook 配置：环境变量优先
if (process.env.FEISHU_MODE) {
  credentials.mode = process.env.FEISHU_MODE as 'websocket' | 'webhook';
}
if (process.env.FEISHU_WEBHOOK_HOST || process.env.FEISHU_WEBHOOK_PORT || ...) {
  credentials.webhook = {
    host: process.env.FEISHU_WEBHOOK_HOST || credentials.webhook?.host || '127.0.0.1',
    port: parseInt(process.env.FEISHU_WEBHOOK_PORT || '') || credentials.webhook?.port || 8080,
    path: process.env.FEISHU_WEBHOOK_PATH || credentials.webhook?.path || '/feishu/webhook',
    encryptKey: process.env.FEISHU_WEBHOOK_ENCRYPT_KEY || credentials.webhook?.encryptKey || '',
    verificationToken: process.env.FEISHU_WEBHOOK_VERIFICATION_TOKEN || credentials.webhook?.verificationToken || '',
  };
}
```

### 不涉及的文件

- `src/channels/registry.ts`
- `src/types.ts`（`Channel` 接口不变）
- `src/index.ts`

## 注意事项

- Webhook 模式下无长连接，`typing indicator` 仍可用（它通过飞书 API 调用，不依赖 WebSocket）
- `FeishuChannel` 的 `setupEventHandlers()` 在构造函数中调用，发生在 `connect()` 之前，WebSocket/Webhook 模式都能正常注册处理器
- HTTP Server 仅监听内网地址，公网可达性由反向代理或内网穿透工具负责
