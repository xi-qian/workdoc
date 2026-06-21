# Monitor Web UI Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Create a Bootstrap 5 web interface for nanoclaw-monitor with login, dashboard, and instance detail pages.

**Architecture:** Express serves static files from `public/` directory. Frontend uses vanilla JavaScript with token-based auth stored in localStorage. API calls use existing endpoints with Bearer token header.

**Tech Stack:** Bootstrap 5, vanilla HTML/CSS/JavaScript, Express static file serving

---

## File Structure

```
packages/monitor/src/
  server.ts                    # Modify: add static file serving
  public/
    index.html                 # Create: Login page
    dashboard.html             # Create: Instance list
    instance.html              # Create: Instance detail
    css/
      style.css                # Create: Minimal custom styles
    js/
      auth.js                  # Create: Login/logout, token management
      api.js                   # Create: API request wrapper with auth header
      dashboard.js             # Create: Dashboard page logic
      instance.js              # Create: Instance detail page logic
```

---

### Task 1: Add Static File Serving to Express Server

**Files:**
- Modify: `packages/monitor/src/server.ts:9-44`

- [ ] **Step 1: Add static file middleware**

Modify server.ts to serve static files and add root route redirect.

```typescript
import express from 'express';
import { Router } from 'express';
import path from 'path';
import { MonitorConfig } from './config.js';
import { createAuthRouter } from './api/auth.js';
import { createInstancesRouter } from './api/instances.js';
import { createContainersRouter } from './api/containers.js';
import { createGroupsRouter } from './api/groups.js';

export function createHttpServer(config: MonitorConfig): express.Application {
  const app = express();

  // Middleware
  app.use(express.json());

  // Serve static files from public directory
  const publicDir = path.join(import.meta.dirname, 'public');
  app.use(express.static(publicDir));

  // Health check (no auth)
  app.get('/health', (req, res) => {
    res.json({ status: 'ok', timestamp: new Date().toISOString() });
  });

  // Redirect /dashboard to dashboard.html
  app.get('/dashboard', (req, res) => {
    res.sendFile(path.join(publicDir, 'dashboard.html'));
  });

  // Redirect /instance to instance.html
  app.get('/instance', (req, res) => {
    res.sendFile(path.join(publicDir, 'instance.html'));
  });

  // API routes
  const apiRouter = Router();

  // Auth routes (no auth required)
  apiRouter.use('/auth', createAuthRouter(config.auth));

  // Instance routes (auth required)
  apiRouter.use('/instances', createInstancesRouter());

  // Container routes (nested under instances)
  apiRouter.use('/instances/:instanceId/containers', createContainersRouter());

  // Group routes (nested under instances)
  apiRouter.use('/instances/:instanceId/groups', createGroupsRouter());

  app.use('/api', apiRouter);

  // Error handler
  app.use((err: Error, req: express.Request, res: express.Response, next: express.NextFunction) => {
    console.error('Error:', err);
    res.status(500).json({ success: false, error: err.message });
  });

  return app;
}
```

- [ ] **Step 2: Rebuild the monitor package**

Run: `cd packages/monitor && npm run build`
Expected: TypeScript compiles successfully

- [ ] **Step 3: Commit**

```bash
git add packages/monitor/src/server.ts
git commit -m "feat(monitor): add static file serving for web UI"
```

---

### Task 2: Create Login Page

**Files:**
- Create: `packages/monitor/src/public/index.html`

- [ ] **Step 1: Create public directory structure**

Run: `mkdir -p packages/monitor/src/public/css packages/monitor/src/public/js`
Expected: Directories created

- [ ] **Step 2: Write login page HTML**

Create `packages/monitor/src/public/index.html`:

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>NanoClaw Monitor - 登录</title>
  <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css" rel="stylesheet">
  <link href="/css/style.css" rel="stylesheet">
</head>
<body class="login-page">
  <div class="container">
    <div class="row justify-content-center align-items-center min-vh-100">
      <div class="col-md-4">
        <div class="card shadow">
          <div class="card-body">
            <h3 class="card-title text-center mb-4">NanoClaw Monitor</h3>
            <form id="loginForm">
              <div class="mb-3">
                <label for="username" class="form-label">用户名</label>
                <input type="text" class="form-control" id="username" value="admin" readonly>
              </div>
              <div class="mb-3">
                <label for="password" class="form-label">密码</label>
                <input type="password" class="form-control" id="password" required>
              </div>
              <div id="errorMessage" class="alert alert-danger d-none"></div>
              <button type="submit" class="btn btn-primary w-100">登录</button>
            </form>
          </div>
        </div>
      </div>
    </div>
  </div>
  <script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/js/bootstrap.bundle.min.js"></script>
  <script src="/js/auth.js"></script>
  <script>
    document.getElementById('loginForm').addEventListener('submit', async (e) => {
      e.preventDefault();
      const password = document.getElementById('password').value;
      const errorDiv = document.getElementById('errorMessage');

      try {
        const result = await auth.login(password);
        if (result.success) {
          window.location.href = '/dashboard';
        } else {
          errorDiv.textContent = result.error || '登录失败';
          errorDiv.classList.remove('d-none');
        }
      } catch (err) {
        errorDiv.textContent = '网络错误，请重试';
        errorDiv.classList.remove('d-none');
      }
    });
  </script>
</body>
</html>
```

- [ ] **Step 3: Commit**

```bash
git add packages/monitor/src/public/index.html
git commit -m "feat(monitor): add login page HTML"
```

---

### Task 3: Create Custom CSS Styles

**Files:**
- Create: `packages/monitor/src/public/css/style.css`

- [ ] **Step 1: Write CSS file**

Create `packages/monitor/src/public/css/style.css`:

```css
/* Login page */
.login-page {
  background-color: #f8f9fa;
}

/* Status badge colors */
.status-running {
  background-color: #198754;
}

.status-idle {
  background-color: #0d6efd;
}

.status-error {
  background-color: #dc3545;
}

.status-offline {
  background-color: #6c757d;
}

/* Instance card hover */
.instance-card {
  cursor: pointer;
  transition: transform 0.2s, box-shadow 0.2s;
}

.instance-card:hover {
  transform: translateY(-2px);
  box-shadow: 0 4px 12px rgba(0,0,0,0.15);
}

/* Connection badge */
.connection-badge {
  font-size: 0.75rem;
}

/* Channels list */
.channels-list {
  font-size: 0.85rem;
}

/* Accordion content pre */
.accordion-body pre {
  background-color: #f8f9fa;
  padding: 10px;
  border-radius: 4px;
  white-space: pre-wrap;
  word-wrap: break-word;
}

/* Memory file content */
.memory-content {
  max-height: 200px;
  overflow-y: auto;
}
```

- [ ] **Step 2: Commit**

```bash
git add packages/monitor/src/public/css/style.css
git commit -m "feat(monitor): add custom CSS styles"
```

---

### Task 4: Create Auth Module

**Files:**
- Create: `packages/monitor/src/public/js/auth.js`

- [ ] **Step 1: Write auth.js**

Create `packages/monitor/src/public/js/auth.js`:

```javascript
// Auth module - handles login/logout and token management
const auth = {
  // Get stored token
  getToken() {
    return localStorage.getItem('monitor_token');
  },

  // Store token
  setToken(token) {
    localStorage.setItem('monitor_token', token);
  },

  // Clear token
  clearToken() {
    localStorage.removeItem('monitor_token');
  },

  // Check if logged in
  isLoggedIn() {
    return this.getToken() !== null;
  },

  // Login with password (username is fixed as 'admin')
  async login(password) {
    const response = await fetch('/api/auth/login', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        username: 'admin',
        password: password,
      }),
    });
    const data = await response.json();
    if (data.success) {
      this.setToken(data.token);
    }
    return data;
  },

  // Logout
  async logout() {
    const token = this.getToken();
    if (token) {
      await fetch('/api/auth/logout', {
        method: 'POST',
        headers: {
          'Authorization': `Bearer ${token}`,
        },
      });
    }
    this.clearToken();
    window.location.href = '/';
  },

  // Verify token is still valid
  async verify() {
    const token = this.getToken();
    if (!token) return false;

    try {
      const response = await fetch('/api/auth/verify', {
        headers: {
          'Authorization': `Bearer ${token}`,
        },
      });
      const data = await response.json();
      return data.success;
    } catch {
      return false;
    }
  },

  // Redirect to login if not authenticated
  async requireAuth() {
    const valid = await this.verify();
    if (!valid) {
      this.clearToken();
      window.location.href = '/';
    }
    return valid;
  },
};
```

- [ ] **Step 2: Commit**

```bash
git add packages/monitor/src/public/js/auth.js
git commit -m "feat(monitor): add auth module for token management"
```

---

### Task 5: Create API Module

**Files:**
- Create: `packages/monitor/src/public/js/api.js`

- [ ] **Step 1: Write api.js**

Create `packages/monitor/src/public/js/api.js`:

```javascript
// API module - wraps API calls with auth header
const api = {
  // Make authenticated API request
  async request(url, options = {}) {
    const token = auth.getToken();
    if (!token) {
      throw new Error('Not authenticated');
    }

    const headers = {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`,
      ...options.headers,
    };

    const response = await fetch(url, {
      ...options,
      headers,
    });

    if (response.status === 401) {
      auth.clearToken();
      window.location.href = '/';
      throw new Error('Session expired');
    }

    return response.json();
  },

  // GET request
  async get(url) {
    return this.request(url, { method: 'GET' });
  },

  // POST request
  async post(url, body) {
    return this.request(url, {
      method: 'POST',
      body: JSON.stringify(body),
    });
  },

  // PUT request
  async put(url, body) {
    return this.request(url, {
      method: 'PUT',
      body: JSON.stringify(body),
    });
  },

  // DELETE request
  async delete(url) {
    return this.request(url, { method: 'DELETE' });
  },

  // Instance API
  async getInstances() {
    return this.get('/api/instances');
  },

  async getInstance(id) {
    return this.get(`/api/instances/${id}`);
  },

  // Container API
  async getContainers(instanceId) {
    return this.get(`/api/instances/${instanceId}/containers`);
  },

  async getContainerLogs(instanceId, containerId) {
    return this.get(`/api/instances/${instanceId}/containers/${containerId}/logs`);
  },

  // Group API
  async getGroups(instanceId) {
    return this.get(`/api/instances/${instanceId}/groups`);
  },

  async getSkills(instanceId, folder) {
    return this.get(`/api/instances/${instanceId}/groups/${folder}/skills`);
  },

  async getMemory(instanceId, folder) {
    return this.get(`/api/instances/${instanceId}/groups/${folder}/memory`);
  },
};
```

- [ ] **Step 2: Commit**

```bash
git add packages/monitor/src/public/js/api.js
git commit -m "feat(monitor): add API module with auth wrapper"
```

---

### Task 6: Create Dashboard Page

**Files:**
- Create: `packages/monitor/src/public/dashboard.html`

- [ ] **Step 1: Write dashboard.html**

Create `packages/monitor/src/public/dashboard.html`:

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>NanoClaw Monitor - Dashboard</title>
  <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css" rel="stylesheet">
  <link href="/css/style.css" rel="stylesheet">
</head>
<body>
  <nav class="navbar navbar-expand-lg navbar-dark bg-dark">
    <div class="container-fluid">
      <a class="navbar-brand" href="/dashboard">NanoClaw Monitor</a>
      <button class="btn btn-outline-light" id="logoutBtn">退出登录</button>
    </div>
  </nav>

  <div class="container mt-4">
    <h2>实例列表</h2>
    <div id="instancesGrid" class="row g-3">
      <div class="col-12 text-center">
        <div class="spinner-border" role="status">
          <span class="visually-hidden">加载中...</span>
        </div>
      </div>
    </div>
  </div>

  <script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/js/bootstrap.bundle.min.js"></script>
  <script src="/js/auth.js"></script>
  <script src="/js/api.js"></script>
  <script src="/js/dashboard.js"></script>
</body>
</html>
```

- [ ] **Step 2: Commit**

```bash
git add packages/monitor/src/public/dashboard.html
git commit -m "feat(monitor): add dashboard page HTML"
```

---

### Task 7: Create Dashboard Logic

**Files:**
- Create: `packages/monitor/src/public/js/dashboard.js`

- [ ] **Step 1: Write dashboard.js**

Create `packages/monitor/src/public/js/dashboard.js`:

```javascript
// Dashboard page logic
(async function initDashboard() {
  // Check authentication
  await auth.requireAuth();

  // Setup logout button
  document.getElementById('logoutBtn').addEventListener('click', () => {
    auth.logout();
  });

  // Load instances
  loadInstances();
})();

async function loadInstances() {
  const grid = document.getElementById('instancesGrid');

  try {
    const result = await api.getInstances();
    if (!result.success) {
      grid.innerHTML = `<div class="col-12"><div class="alert alert-danger">加载失败: ${result.error}</div></div>`;
      return;
    }

    const instances = result.instances;

    if (instances.length === 0) {
      grid.innerHTML = `<div class="col-12"><div class="alert alert-info">暂无连接的实例</div></div>`;
      return;
    }

    grid.innerHTML = instances.map(inst => createInstanceCard(inst)).join('');

    // Add click handlers
    grid.querySelectorAll('.instance-card').forEach(card => {
      card.addEventListener('click', () => {
        const id = card.dataset.instanceId;
        window.location.href = `/instance?id=${id}`;
      });
    });
  } catch (err) {
    grid.innerHTML = `<div class="col-12"><div class="alert alert-danger">加载失败: ${err.message}</div></div>`;
  }
}

function createInstanceCard(instance) {
  const statusClass = getStatusClass(instance.status);
  const connectionBadge = instance.connected
    ? '<span class="badge bg-success connection-badge">已连接</span>'
    : '<span class="badge bg-secondary connection-badge">离线</span>';

  const lastHeartbeat = instance.last_heartbeat
    ? formatTime(instance.last_heartbeat)
    : '未知';

  const channels = instance.channels || [];
  const channelsHtml = channels.length > 0
    ? `<div class="channels-list text-muted">频道: ${channels.join(', ')}</div>`
    : '';

  return `
    <div class="col-md-4 col-lg-3">
      <div class="card instance-card" data-instance-id="${instance.instance_id}">
        <div class="card-body">
          <div class="d-flex justify-content-between align-items-start mb-2">
            <h5 class="card-title">${instance.hostname || instance.instance_id}</h5>
            <span class="badge ${statusClass}">${instance.status || '未知'}</span>
          </div>
          <div class="mb-2">
            <small class="text-muted">ID: ${instance.instance_id}</small>
          </div>
          <div class="mb-2">
            ${connectionBadge}
          </div>
          <div class="mb-2">
            <small class="text-muted">最后心跳: ${lastHeartbeat}</small>
          </div>
          ${channelsHtml}
        </div>
      </div>
    </div>
  `;
}

function getStatusClass(status) {
  switch (status) {
    case 'running':
      return 'status-running';
    case 'idle':
      return 'status-idle';
    case 'error':
      return 'status-error';
    default:
      return 'status-offline';
  }
}

function formatTime(timestamp) {
  const date = new Date(timestamp);
  return date.toLocaleString('zh-CN');
}
```

- [ ] **Step 2: Commit**

```bash
git add packages/monitor/src/public/js/dashboard.js
git commit -m "feat(monitor): add dashboard page logic"
```

---

### Task 8: Create Instance Detail Page

**Files:**
- Create: `packages/monitor/src/public/instance.html`

- [ ] **Step 1: Write instance.html**

Create `packages/monitor/src/public/instance.html`:

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>NanoClaw Monitor - Instance Detail</title>
  <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css" rel="stylesheet">
  <link href="/css/style.css" rel="stylesheet">
</head>
<body>
  <nav class="navbar navbar-expand-lg navbar-dark bg-dark">
    <div class="container-fluid">
      <a class="navbar-brand" href="/dashboard">NanoClaw Monitor</a>
      <div class="d-flex">
        <a href="/dashboard" class="btn btn-outline-light me-2">返回列表</a>
        <button class="btn btn-outline-light" id="logoutBtn">退出登录</button>
      </div>
    </div>
  </nav>

  <div class="container mt-4">
    <!-- Instance header -->
    <div id="instanceHeader" class="mb-4">
      <div class="spinner-border" role="status">
        <span class="visually-hidden">加载中...</span>
      </div>
    </div>

    <!-- Accordion for sections -->
    <div class="accordion" id="instanceAccordion">
      <!-- Containers section -->
      <div class="accordion-item">
        <h2 class="accordion-header">
          <button class="accordion-button collapsed" type="button" data-bs-toggle="collapse" data-bs-target="#containersSection">
            容器列表
          </button>
        </h2>
        <div id="containersSection" class="accordion-collapse collapse" data-bs-parent="#instanceAccordion">
          <div class="accordion-body" id="containersBody">
            加载中...
          </div>
        </div>
      </div>

      <!-- Groups section -->
      <div class="accordion-item">
        <h2 class="accordion-header">
          <button class="accordion-button collapsed" type="button" data-bs-toggle="collapse" data-bs-target="#groupsSection">
            群组列表
          </button>
        </h2>
        <div id="groupsSection" class="accordion-collapse collapse" data-bs-parent="#instanceAccordion">
          <div class="accordion-body" id="groupsBody">
            加载中...
          </div>
        </div>
      </div>

      <!-- Skills section (loaded dynamically) -->
      <div class="accordion-item">
        <h2 class="accordion-header">
          <button class="accordion-button collapsed" type="button" data-bs-toggle="collapse" data-bs-target="#skillsSection">
            Skills
          </button>
        </h2>
        <div id="skillsSection" class="accordion-collapse collapse" data-bs-parent="#instanceAccordion">
          <div class="accordion-body" id="skillsBody">
            请先展开群组列表，点击群组查看 Skills
          </div>
        </div>
      </div>

      <!-- Memory section (loaded dynamically) -->
      <div class="accordion-item">
        <h2 class="accordion-header">
          <button class="accordion-button collapsed" type="button" data-bs-toggle="collapse" data-bs-target="#memorySection">
            Memory 文件
          </button>
        </h2>
        <div id="memorySection" class="accordion-collapse collapse" data-bs-parent="#instanceAccordion">
          <div class="accordion-body" id="memoryBody">
            请先展开群组列表，点击群组查看 Memory
          </div>
        </div>
      </div>
    </div>
  </div>

  <script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/js/bootstrap.bundle.min.js"></script>
  <script src="/js/auth.js"></script>
  <script src="/js/api.js"></script>
  <script src="/js/instance.js"></script>
</body>
</html>
```

- [ ] **Step 2: Commit**

```bash
git add packages/monitor/src/public/instance.html
git commit -m "feat(monitor): add instance detail page HTML"
```

---

### Task 9: Create Instance Detail Logic

**Files:**
- Create: `packages/monitor/src/public/js/instance.js`

- [ ] **Step 1: Write instance.js**

Create `packages/monitor/src/public/js/instance.js`:

```javascript
// Instance detail page logic
let currentInstanceId = null;

(async function initInstancePage() {
  // Check authentication
  await auth.requireAuth();

  // Setup logout button
  document.getElementById('logoutBtn').addEventListener('click', () => {
    auth.logout();
  });

  // Get instance ID from URL
  const params = new URLSearchParams(window.location.search);
  currentInstanceId = params.get('id');

  if (!currentInstanceId) {
    document.getElementById('instanceHeader').innerHTML =
      '<div class="alert alert-danger">缺少实例 ID 参数</div>';
    return;
  }

  // Load instance data
  loadInstanceHeader();
})();

async function loadInstanceHeader() {
  const headerDiv = document.getElementById('instanceHeader');

  try {
    const result = await api.getInstance(currentInstanceId);
    if (!result.success) {
      headerDiv.innerHTML = `<div class="alert alert-danger">加载失败: ${result.error}</div>`;
      return;
    }

    const inst = result.instance;
    const statusClass = getStatusClass(inst.status);
    const connectionBadge = inst.connected
      ? '<span class="badge bg-success">已连接</span>'
      : '<span class="badge bg-secondary">离线</span>';

    headerDiv.innerHTML = `
      <div class="card">
        <div class="card-body">
          <div class="row">
            <div class="col-md-6">
              <h4>${inst.hostname || inst.instance_id}</h4>
              <p class="text-muted">ID: ${inst.instance_id}</p>
            </div>
            <div class="col-md-6 text-end">
              <span class="badge ${statusClass}">${inst.status || '未知'}</span>
              ${connectionBadge}
            </div>
          </div>
          <div class="row mt-3">
            <div class="col-md-4">
              <small class="text-muted">版本: ${inst.version || '未知'}</small>
            </div>
            <div class="col-md-4">
              <small class="text-muted">启动时间: ${inst.start_time ? formatTime(inst.start_time) : '未知'}</small>
            </div>
            <div class="col-md-4">
              <small class="text-muted">最后心跳: ${inst.last_heartbeat ? formatTime(inst.last_heartbeat) : '未知'}</small>
            </div>
          </div>
        </div>
      </div>
    `;

    // Load containers and groups
    loadContainers();
    loadGroups();
  } catch (err) {
    headerDiv.innerHTML = `<div class="alert alert-danger">加载失败: ${err.message}</div>`;
  }
}

async function loadContainers() {
  const bodyDiv = document.getElementById('containersBody');

  try {
    const result = await api.getContainers(currentInstanceId);
    if (!result.success) {
      bodyDiv.innerHTML = `<div class="alert alert-warning">加载失败: ${result.error}</div>`;
      return;
    }

    const containers = result.containers;

    if (containers.length === 0) {
      bodyDiv.innerHTML = '<div class="text-muted">暂无容器</div>';
      return;
    }

    bodyDiv.innerHTML = `
      <table class="table table-sm">
        <thead>
          <tr>
            <th>ID</th>
            <th>名称</th>
            <th>状态</th>
            <th>群组</th>
          </tr>
        </thead>
        <tbody>
          ${containers.map(c => `
            <tr>
              <td><small>${c.container_id}</small></td>
              <td>${c.name || '-'}</td>
              <td><span class="badge ${getStatusClass(c.status)}">${c.status}</span></td>
              <td>${c.group_folder || '-'}</td>
            </tr>
          `).join('')}
        </tbody>
      </table>
    `;
  } catch (err) {
    bodyDiv.innerHTML = `<div class="alert alert-warning">加载失败: ${err.message}</div>`;
  }
}

async function loadGroups() {
  const bodyDiv = document.getElementById('groupsBody');

  try {
    const result = await api.getGroups(currentInstanceId);
    if (!result.success) {
      bodyDiv.innerHTML = `<div class="alert alert-warning">加载失败: ${result.error}</div>`;
      return;
    }

    const groups = result.groups;

    if (!groups || groups.length === 0) {
      bodyDiv.innerHTML = '<div class="text-muted">暂无群组</div>';
      return;
    }

    bodyDiv.innerHTML = `
      <table class="table table-sm">
        <thead>
          <tr>
            <th>名称</th>
            <th>文件夹</th>
            <th>主群组</th>
            <th>触发模式</th>
            <th>操作</th>
          </tr>
        </thead>
        <tbody>
          ${groups.map(g => `
            <tr>
              <td>${g.name}</td>
              <td>${g.folder}</td>
              <td>${g.is_main ? '<span class="badge bg-primary">主</span>' : ''}</td>
              <td><code>${g.trigger_pattern || '-'}</code></td>
              <td>
                <button class="btn btn-sm btn-outline-primary" onclick="loadGroupSkills('${g.folder}')">Skills</button>
                <button class="btn btn-sm btn-outline-secondary" onclick="loadGroupMemory('${g.folder}')">Memory</button>
              </td>
            </tr>
          `).join('')}
        </tbody>
      </table>
    `;
  } catch (err) {
    bodyDiv.innerHTML = `<div class="alert alert-warning">加载失败: ${err.message}</div>`;
  }
}

async function loadGroupSkills(folder) {
  const bodyDiv = document.getElementById('skillsBody');

  // Expand the skills section
  const skillsSection = document.getElementById('skillsSection');
  const bsCollapse = new bootstrap.Collapse(skillsSection, { toggle: true });

  bodyDiv.innerHTML = '<div class="spinner-border spinner-border-sm"></div> 加载中...';

  try {
    const result = await api.getSkills(currentInstanceId, folder);
    if (!result.success) {
      bodyDiv.innerHTML = `<div class="alert alert-warning">加载失败: ${result.error}</div>`;
      return;
    }

    const skills = result.skills;
    const cachedBadge = result.cached
      ? '<span class="badge bg-secondary ms-2">缓存</span>'
      : '<span class="badge bg-success ms-2">实时</span>';

    if (!skills || skills.length === 0) {
      bodyDiv.innerHTML = '<div class="text-muted">暂无 Skills</div>';
      return;
    }

    bodyDiv.innerHTML = `
      <h6>群组 ${folder} 的 Skills ${cachedBadge}</h6>
      <div class="accordion accordion-flush">
        ${skills.map((s, i) => `
          <div class="accordion-item">
            <h2 class="accordion-header">
              <button class="accordion-button collapsed" type="button" data-bs-toggle="collapse" data-bs-target="#skill-${i}">
                ${s.name}
              </button>
            </h2>
            <div id="skill-${i}" class="accordion-collapse collapse">
              <div class="accordion-body">
                <pre>${escapeHtml(s.content || '(空)')}</pre>
              </div>
            </div>
          </div>
        `).join('')}
      </div>
    `;
  } catch (err) {
    bodyDiv.innerHTML = `<div class="alert alert-warning">加载失败: ${err.message}</div>`;
  }
}

async function loadGroupMemory(folder) {
  const bodyDiv = document.getElementById('memoryBody');

  // Expand the memory section
  const memorySection = document.getElementById('memorySection');
  const bsCollapse = new bootstrap.Collapse(memorySection, { toggle: true });

  bodyDiv.innerHTML = '<div class="spinner-border spinner-border-sm"></div> 加载中...';

  try {
    const result = await api.getMemory(currentInstanceId, folder);
    if (!result.success) {
      bodyDiv.innerHTML = `<div class="alert alert-warning">加载失败: ${result.error}</div>`;
      return;
    }

    const memory = result.memory;
    const cachedBadge = result.cached
      ? '<span class="badge bg-secondary ms-2">缓存</span>'
      : '<span class="badge bg-success ms-2">实时</span>';

    if (!memory || memory.length === 0) {
      bodyDiv.innerHTML = '<div class="text-muted">暂无 Memory 文件</div>';
      return;
    }

    bodyDiv.innerHTML = `
      <h6>群组 ${folder} 的 Memory ${cachedBadge}</h6>
      <div class="accordion accordion-flush">
        ${memory.map((m, i) => `
          <div class="accordion-item">
            <h2 class="accordion-header">
              <button class="accordion-button collapsed" type="button" data-bs-toggle="collapse" data-bs-target="#memory-${i}">
                ${m.filename}
              </button>
            </h2>
            <div id="memory-${i}" class="accordion-collapse collapse">
              <div class="accordion-body memory-content">
                <pre>${escapeHtml(m.content || '(空)')}</pre>
              </div>
            </div>
          </div>
        `).join('')}
      </div>
    `;
  } catch (err) {
    bodyDiv.innerHTML = `<div class="alert alert-warning">加载失败: ${err.message}</div>`;
  }
}

function getStatusClass(status) {
  switch (status) {
    case 'running':
      return 'status-running';
    case 'idle':
      return 'status-idle';
    case 'error':
      return 'status-error';
    default:
      return 'status-offline';
  }
}

function formatTime(timestamp) {
  const date = new Date(timestamp);
  return date.toLocaleString('zh-CN');
}

function escapeHtml(text) {
  const div = document.createElement('div');
  div.textContent = text;
  return div.innerHTML;
}
```

- [ ] **Step 2: Commit**

```bash
git add packages/monitor/src/public/js/instance.js
git commit -m "feat(monitor): add instance detail page logic"
```

---

### Task 10: Rebuild and Test

**Files:**
- None (build and verification)

- [ ] **Step 1: Rebuild monitor package**

Run: `cd packages/monitor && npm run build`
Expected: TypeScript compiles successfully, no errors

- [ ] **Step 2: Restart monitor service**

Run: `cd packages/monitor && pm2 restart nanoclaw-monitor`
Expected: Monitor restarts with new static file serving

- [ ] **Step 3: Verify login page loads**

Run: `curl -s http://localhost:8090/ | head -20`
Expected: HTML content with "NanoClaw Monitor - 登录" title

- [ ] **Step 4: Final commit (squash if needed)**

If multiple commits, optionally squash to single commit:

```bash
git log --oneline HEAD~10..HEAD
# If too many small commits:
git reset --soft HEAD~10
git commit -m "feat(monitor): add Bootstrap 5 web UI for monitoring"
git push --force-with-lease
```

Otherwise, just push:

```bash
git push
```

---

## Spec Coverage Check

| Spec Section | Task |
|--------------|------|
| Login Page (`/`) | Task 2, 4 |
| Dashboard (`/dashboard`) | Task 6, 7 |
| Instance Detail (`/instance?id=`) | Task 8, 9 |
| CSS Styles | Task 3 |
| API Integration | Task 5 |
| Static File Serving | Task 1 |

All spec requirements covered.