# Feishu Webhook 回调模式 实现计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 为 Feishu 通道新增 Webhook 回调模式，与现有 WebSocket 模式并列可选，通过 credentials.json 或环境变量切换。

**Architecture:** 在 `FeishuClient` 中新增 HTTP Server（Node.js 内置 `http` 模块），根据 `mode` 配置分支：`webhook` 启动 HTTP Server 接收飞书事件回调，`websocket` 走现有 WSClient。事件经过解密/解析后统一通过 `emit()` 分发，`FeishuChannel` 层完全不变。

**Tech Stack:** Node.js `http` 模块, Node.js `crypto` 模块（SHA256 + AES-256-CBC）

---

### Task 1: 更新类型定义

**Files:**
- Modify: `src/feishu/types.ts`

- [ ] **Step 1: 更新 `FeishuCredentials` 接口**

在 `src/feishu/types.ts` 中为 `FeishuCredentials` 新增 `mode` 和 `webhook` 字段：

```typescript
export interface FeishuCredentials {
  appId: string;
  appSecret: string;
  accessToken?: string;
  refreshToken?: string;
  expiresAt?: string;
  tenantKey?: string;
  /** 连接模式：websocket（默认）或 webhook */
  mode?: 'websocket' | 'webhook';
  /** webhook 模式配置 */
  webhook?: {
    /** HTTP Server 监听地址，默认 127.0.0.1 */
    host?: string;
    /** HTTP Server 监听端口，默认 8080 */
    port?: number;
    /** 回调 URL 路径，默认 /feishu/webhook */
    path?: string;
    /** 加密密钥，为空则明文模式 */
    encryptKey?: string;
    /** URL 验证 Token */
    verificationToken?: string;
  };
}
```

- [ ] **Step 2: 编译检查**

```bash
cd /home/yeats/claude_project/nanoclaw-fork/nanoclaw && npx tsc --noEmit src/feishu/types.ts
```

- [ ] **Step 3: Commit**

```bash
git add src/feishu/types.ts
git commit -m "feat: add mode and webhook fields to FeishuCredentials type

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>"
```

---

### Task 2: 更新 auth.ts（类型 + 环境变量合并）

**Files:**
- Modify: `src/feishu/auth.ts`

- [ ] **Step 1: 更新 `loadCredentials` 中合并环境变量**

在 `src/feishu/auth.ts` 中，`loadCredentials()` 函数 return 前插入环境变量合并逻辑。找到 `return credentials;`（约在第 63 行），在其前插入：

```typescript
// --- webhook 配置：环境变量优先 ---
if (process.env.FEISHU_MODE) {
  credentials.mode = process.env.FEISHU_MODE as 'websocket' | 'webhook';
}

const envHost = process.env.FEISHU_WEBHOOK_HOST;
const envPort = process.env.FEISHU_WEBHOOK_PORT;
const envPath = process.env.FEISHU_WEBHOOK_PATH;
const envEncryptKey = process.env.FEISHU_WEBHOOK_ENCRYPT_KEY;
const envVerificationToken = process.env.FEISHU_WEBHOOK_VERIFICATION_TOKEN;

if (envHost || envPort || envPath || envEncryptKey || envVerificationToken) {
  credentials.webhook = {
    host: envHost || credentials.webhook?.host || '127.0.0.1',
    port: envPort ? parseInt(envPort, 10) : (credentials.webhook?.port || 8080),
    path: envPath || credentials.webhook?.path || '/feishu/webhook',
    encryptKey: envEncryptKey !== undefined ? envEncryptKey : (credentials.webhook?.encryptKey || ''),
    verificationToken: envVerificationToken !== undefined ? envVerificationToken : (credentials.webhook?.verificationToken || ''),
  };
}
```

注意：`auth.ts` 中已经有接口 `FeishuCredentials`（第 29 行），与 `types.ts` 中的定义重复。需要更新 `auth.ts` 中的定义使其与 `types.ts` 一致：

```typescript
export interface FeishuCredentials {
  appId: string;
  appSecret: string;
  accessToken?: string;
  refreshToken?: string;
  expiresAt?: string;
  tenantKey?: string;
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

- [ ] **Step 2: 编译检查**

```bash
cd /home/yeats/claude_project/nanoclaw-fork/nanoclaw && npx tsc --noEmit src/feishu/auth.ts
```

- [ ] **Step 3: Commit**

```bash
git add src/feishu/auth.ts
git commit -m "feat: merge webhook env vars into FeishuCredentials on load

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>"
```

---

### Task 3: 添加 AES 解密方法到 FeishuClient

**Files:**
- Modify: `src/feishu/client.ts`

- [ ] **Step 1: 导入 `crypto` 模块**

在 `src/feishu/client.ts` 顶部已有 import 区域添加：

```typescript
import crypto from 'crypto';
```

- [ ] **Step 2: 添加 `decryptWebhookEvent` 和 `parseWebhookBody` 方法**

在 `FeishuClient` 类的末尾（最后一个 `}` 之前），`uploadAndSendFile` 方法之后，添加：

```typescript
/**
 * 解密飞书 webhook 加密事件体
 * 加密算法：AES-256-CBC
 * - key: SHA256(encryptKey) 的前 16 字节
 * - iv: 密文（base64 解码后）的前 16 字节
 * - 密文: 从第 17 字节开始
 */
private decryptWebhookEvent(encryptedBase64: string): string {
  const encryptKey = this.credentials.webhook?.encryptKey || '';
  if (!encryptKey) {
    throw new Error('Webhook encryptKey not configured');
  }

  const encrypted = Buffer.from(encryptedBase64, 'base64');
  const iv = encrypted.subarray(0, 16);
  const ciphertext = encrypted.subarray(16);

  const hash = crypto.createHash('sha256').update(encryptKey).digest();
  const key = hash.subarray(0, 16);

  const decipher = crypto.createDecipheriv('aes-256-cbc', key, iv);
  let decrypted = decipher.update(ciphertext);
  decrypted = Buffer.concat([decrypted, decipher.final()]);

  return decrypted.toString('utf-8');
}

/**
 * 解析 webhook 请求体，生成 { type, event } 结构
 * 支持明文和加密两种格式：
 * - 加密：{"encrypt": "base64..."}
 * - 明文：{"schema": "2.0", "header": {"event_type": "..."}, "event": {...}}
 */
private parseWebhookBody(
  rawBody: string,
): { type: string; event: any } | null {
  let body: any;
  try {
    body = JSON.parse(rawBody);
  } catch {
    log.error({ rawBody: rawBody.slice(0, 200) }, 'Failed to parse webhook body as JSON');
    return null;
  }

  // 1. 处理加密格式
  if (body.encrypt) {
    try {
      const decrypted = this.decryptWebhookEvent(body.encrypt);
      body = JSON.parse(decrypted);
    } catch (err) {
      log.error({ err }, 'Failed to decrypt webhook event');
      return null;
    }
  }

  // 2. URL 验证请求（type: "url_verification"）不在事件分发中处理
  if (body.type === 'url_verification') {
    return null;
  }

  // 3. 提取 schema 2.0 格式的事件
  if (body.schema && body.header?.event_type && body.event) {
    return { type: body.header.event_type, event: body.event };
  }

  // 4. 旧版 V1 格式（直接嵌套在 body 中）
  if (body.event?.type) {
    return { type: body.event.type, event: body.event };
  }

  log.warn({ body: JSON.stringify(body).slice(0, 200) }, 'Unknown webhook body format');
  return null;
}
```

- [ ] **Step 3: 编译检查**

```bash
cd /home/yeats/claude_project/nanoclaw-fork/nanoclaw && npx tsc --noEmit src/feishu/client.ts
```

- [ ] **Step 4: Commit**

```bash
git add src/feishu/client.ts
git commit -m "feat: add AES decryption and webhook body parsing to FeishuClient

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>"
```

---

### Task 4: 添加 webhook HTTP Server 到 FeishuClient

**Files:**
- Modify: `src/feishu/client.ts`

- [ ] **Step 1: 导入 `http` 模块**

在 `src/feishu/client.ts` 顶部已有 import 区域添加：

```typescript
import http from 'http';
```

- [ ] **Step 2: 添加 `webhookServer` 字段**

在 `FeishuClient` 类的字段声明区域（在 `private eventHandlers` 约第 132 行之后）添加：

```typescript
private webhookServer: http.Server | null = null;
```

- [ ] **Step 3: 添加 webhook server 相关方法**

在 `FeishuClient` 类的末尾，上一步添加的 `parseWebhookBody` 方法之后，添加：

```typescript
/**
 * 启动 webhook HTTP Server
 */
private startWebhookServer(): Promise<void> {
  return new Promise((resolve, reject) => {
    const cfg = this.credentials.webhook || {};
    const host = cfg.host || '127.0.0.1';
    const port = cfg.port || 8080;
    const path = cfg.path || '/feishu/webhook';
    const verificationToken = cfg.verificationToken || '';

    this.webhookServer = http.createServer((req, res) => {
      this.handleWebhookRequest(req, res, path, verificationToken);
    });

    this.webhookServer.on('error', (err) => {
      log.error({ err }, 'Webhook HTTP server error');
    });

    this.webhookServer.listen(port, host, () => {
      log.info({ host, port, path }, 'Webhook HTTP server started');
      resolve();
    });

    this.webhookServer.once('error', reject);
  });
}

/**
 * 处理 webhook HTTP 请求
 */
private handleWebhookRequest(
  req: http.IncomingMessage,
  res: http.ServerResponse,
  path: string,
  verificationToken: string,
): void {
  // 仅接受 POST 请求到配置的路径
  if (req.method !== 'POST' || req.url !== path) {
    res.writeHead(404);
    res.end('Not Found');
    return;
  }

  let body = '';
  req.on('data', (chunk) => {
    body += chunk;
  });

  req.on('end', () => {
    try {
      const jsonBody = JSON.parse(body);

      // 1. URL 验证请求
      if (jsonBody.type === 'url_verification') {
        // 如果配置了 verificationToken，进行校验
        if (verificationToken && jsonBody.token !== verificationToken) {
          log.warn({ token: jsonBody.token }, 'URL verification token mismatch');
          res.writeHead(403);
          res.end(JSON.stringify({ error: 'token mismatch' }));
          return;
        }
        log.info({ challenge: jsonBody.challenge }, 'URL verification succeeded');
        res.writeHead(200, { 'Content-Type': 'application/json' });
        res.end(JSON.stringify({ challenge: jsonBody.challenge }));
        return;
      }

      // 2. 事件回调
      this.handleEventCallback(body);
      res.writeHead(200);
      res.end('ok');
    } catch (err) {
      log.error({ err }, 'Failed to process webhook request');
      res.writeHead(400);
      res.end('Bad Request');
    }
  });
}

/**
 * 处理事件回调 —— 解析事件体并分发给已注册的 handler
 */
private handleEventCallback(rawBody: string): void {
  const parsed = this.parseWebhookBody(rawBody);
  if (!parsed) return; // URL 验证或未知格式，跳过

  const { type, event } = parsed;

  // 构造与 WebSocket EventDispatcher 一致的 FeishuEvent 结构
  const feishuEvent = { type, event };

  log.info({ type }, 'Webhook event received');

  // 通过现有的 emit 机制分发事件（与 WebSocket 路径完全一致）
  this.emit(type, feishuEvent);
}
```

- [ ] **Step 4: 编译检查**

```bash
cd /home/yeats/claude_project/nanoclaw-fork/nanoclaw && npx tsc --noEmit src/feishu/client.ts
```

- [ ] **Step 5: Commit**

```bash
git add src/feishu/client.ts
git commit -m "feat: add webhook HTTP server to FeishuClient

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>"
```

---

### Task 5: 分支 connect / isConnected / disconnect

**Files:**
- Modify: `src/feishu/client.ts`

- [ ] **Step 1: 修改 `connect()` 方法**

将 `connect()` 方法改为按 `mode` 分支：

```typescript
async connect(): Promise<void> {
  const mode = this.credentials.mode || 'websocket';

  if (mode === 'webhook') {
    await this.startWebhookServer();
    return;
  }

  // --- 以下为原有 WebSocket 连接逻辑，完全不变 ---
  try {
    log.info('Starting WebSocket connection...');

    this.wsClient = new Lark.WSClient({
      appId: this.credentials.appId,
      appSecret: this.credentials.appSecret,
      domain: BRAND_TO_DOMAIN[this.brand],
      loggerLevel: Lark.LoggerLevel.debug,
    });

    const self = this;
    const eventDispatcher = new Lark.EventDispatcher({
      loggerLevel: Lark.LoggerLevel.debug,
    }).register({
      'im.message.receive_v1': async (data: any) => {
        log.info(
          { data: JSON.stringify(data) },
          'Event received from EventDispatcher',
        );
        self.emit('im.message.receive_v1', {
          type: 'im.message.receive_v1',
          event: data,
        });
      },
      'card.action.trigger': async (data: any) => {
        log.info(
          { data: JSON.stringify(data) },
          'Card action event received from EventDispatcher',
        );
        self.emit('card.action.trigger', {
          type: 'card.action.trigger',
          event: data,
        });
      },
    });
    log.info(
      'EventDispatcher registered for im.message.receive_v1 and card.action.trigger',
    );

    const startParams = { eventDispatcher };
    await this.wsClient.start(startParams);

    log.info('Feishu WebSocket client connected');
  } catch (error) {
    log.error(
      { error: error instanceof Error ? error.message : String(error) },
      'Failed to connect WebSocket',
    );

    if (error instanceof Error) {
      log.error('WebSocket connection failed. Please ensure:');
      log.error(
        '1. The app is configured for persistent connection in Feishu console',
      );
      log.error('2. Network connectivity');
      log.error('3. Correct App ID and App Secret');
    }
    throw error;
  }
}
```

- [ ] **Step 2: 修改 `isConnected()` 方法**

```typescript
isConnected(): boolean {
  const mode = this.credentials.mode || 'websocket';

  if (mode === 'webhook') {
    return this.webhookServer?.listening ?? false;
  }

  return this.wsClient !== null && this.credentials.accessToken !== undefined;
}
```

- [ ] **Step 3: 修改 `disconnect()` 方法**

```typescript
async disconnect(): Promise<void> {
  const mode = this.credentials.mode || 'websocket';

  if (mode === 'webhook') {
    if (this.webhookServer) {
      await new Promise<void>((resolve, reject) => {
        this.webhookServer!.close((err) => {
          if (err) reject(err);
          else resolve();
        });
      });
      this.webhookServer = null;
      log.info('Feishu Webhook HTTP server stopped');
    }
    return;
  }

  if (this.wsClient) {
    try {
      await this.wsClient.close();
      this.wsClient = null;
      log.info('Feishu WebSocket client disconnected');
    } catch (error) {
      log.error(
        { error: error instanceof Error ? error.message : String(error) },
        'Failed to stop WebSocket',
      );
    }
  }
}
```

- [ ] **Step 4: 编译检查**

```bash
cd /home/yeats/claude_project/nanoclaw-fork/nanoclaw && npx tsc --noEmit src/feishu/client.ts
```

- [ ] **Step 5: 运行现有测试确保 WebSocket 模式未受影响**

```bash
cd /home/yeats/claude_project/nanoclaw-fork/nanoclaw && npm test
```

- [ ] **Step 6: Commit**

```bash
git add src/feishu/client.ts
git commit -m "feat: branch connect/isConnected/disconnect on webhook vs websocket mode

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>"
```

---

### Task 6: 验证与清理

**Files:**
- No code changes. Validation only.

- [ ] **Step 1: 整体编译检查**

```bash
cd /home/yeats/claude_project/nanoclaw-fork/nanoclaw && npx tsc --noEmit
```

- [ ] **Step 2: 运行全部测试**

```bash
cd /home/yeats/claude_project/nanoclaw-fork/nanoclaw && npm test
```

- [ ] **Step 3: 验证 `FeishuChannel` 层未受影响**

```bash
cd /home/yeats/claude_project/nanoclaw-fork/nanoclaw && git diff main -- src/channels/feishu.ts
```

预期：无 diff（或仅有 prettier 格式化变更）。

- [ ] **Step 4: 最终 commit（如有格式化变更）**

```bash
git add -A && git diff --cached --quiet || git commit -m "chore: prettier formatting for feishu webhook changes

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>"
```
