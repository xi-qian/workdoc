# NanoClaw Monitor Web UI Design

**Date:** 2026-04-11
**Status:** Draft

## Overview

Create a simple, user-friendly web interface for nanoclaw-monitor that displays instance monitoring information. Built with Bootstrap 5 and vanilla JavaScript, embedded directly in the Express server.

## Target Users

Non-developer users who need to monitor nanoclaw instances - requires a friendly, intuitive interface.

## Technology Stack

- **Frontend:** Bootstrap 5 + vanilla HTML/CSS/JavaScript
- **No build step:** Static files served directly by Express
- **Authentication:** Password-based (existing API)

## Pages

### 1. Login Page (`/`)

- Simple password login form
- Username: admin (fixed)
- Password: from environment variable
- Stores session token in localStorage
- Redirects to dashboard on success

### 2. Dashboard (`/dashboard`)

- Instance cards in grid layout
- Each card shows:
  - Instance ID
  - Hostname
  - Status indicator (running/idle/error/offline) with color coding
  - Last heartbeat time
  - Connection status badge
  - Channels list
- Click card → navigate to instance detail page
- Logout button in header

### 3. Instance Detail (`/instance?id=<instanceId>`)

- Header with instance info (hostname, version, start time)
- Collapsible sections using Bootstrap accordion:
  - **Containers** - list with status, name, group folder
  - **Groups** - list with name, main group indicator, trigger pattern
  - **Skills** - grouped by folder, expandable to show content
  - **Memory Files** - list with expandable content preview
- "Back to Dashboard" button
- Logout button in header

## Visual Design

- Bootstrap 5 default theme
- Color coding for status:
  - Running: green badge
  - Idle: blue badge
  - Error: red badge
  - Offline: gray badge
- Responsive layout (works on mobile)

## File Structure

```
packages/monitor/
  src/
    public/
      index.html        # Login page
      dashboard.html    # Instance list
      instance.html     # Instance detail
      css/
        style.css       # Minimal custom styles
      js/
        auth.js         # Login/logout, token management
        api.js           # API request wrapper with auth header
        dashboard.js    # Dashboard page logic
        instance.js     # Instance detail page logic
```

## API Integration

All existing API endpoints are used:

| Endpoint | Used In |
|----------|---------|
| `POST /api/auth/login` | Login page |
| `GET /api/instances` | Dashboard |
| `GET /api/instances/:id` | Instance detail header |
| `GET /api/instances/:id/containers` | Containers section |
| `GET /api/instances/:id/groups` | Groups section |
| `GET /api/instances/:id/groups/:folder/skills` | Skills section |
| `GET /api/instances/:id/groups/:folder/memory` | Memory section |

## Future Extensions (Not in This Phase)

- Restart instance button
- Stop container button
- Edit skill/memory content
- Real-time updates via WebSocket
- Multi-instance comparison view