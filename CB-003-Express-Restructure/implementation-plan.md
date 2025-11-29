# CB-003: Implementation Plan - Express.js Restructure

## Technical Approach

### Architecture Overview
Transform the monolithic `server.js` into a layered architecture:

```
Request → Routes → Controllers → Services → Response
              ↓
          Middleware (cross-cutting concerns)
```

### Layer Responsibilities

| Layer | Responsibility |
|-------|---------------|
| **config/** | Environment variables, constants |
| **middleware/** | Cross-cutting concerns (auth, logging, errors) |
| **routes/** | URL mapping, route definitions |
| **controllers/** | Request handling, response formatting |
| **Services/** | Business logic (existing) |
| **Handlers/** | External integrations (existing) |

## Implementation Steps

### Step 1: Create Configuration Module
**File**: `src/config/index.js`

Extract from server.js:
- Environment variables
- Port configuration
- Feature flags

### Step 2: Create Middleware
**Files**:
- `src/middleware/requestLogger.js` - Request logging
- `src/middleware/errorHandler.js` - Error handling (404, 500)
- `src/middleware/auth.js` - API key verification
- `src/middleware/index.js` - Export aggregator

### Step 3: Create Controllers
**Files**:
- `src/controllers/bot.controller.js` - Bot message handling
- `src/controllers/webhook.controller.js` - LangFlow webhooks
- `src/controllers/proactive.controller.js` - Proactive messages
- `src/controllers/admin.controller.js` - Admin endpoints

### Step 4: Create Routes
**Files**:
- `src/routes/health.routes.js` - `/`, `/health`, `/ready`
- `src/routes/bot.routes.js` - `/api/messages`, `/Messages`
- `src/routes/webhook.routes.js` - `/api/webhook/*`
- `src/routes/proactive.routes.js` - `/api/proactive`
- `src/routes/admin.routes.js` - `/api/admin/*`
- `src/routes/index.js` - Route aggregator

### Step 5: Create App Configuration
**File**: `src/app.js`

- Express app initialization
- Middleware registration
- Route registration
- Error handlers

### Step 6: Refactor Server Entry Point
**File**: `src/server.js`

- Import app
- Start server
- Graceful shutdown
- Startup logging

## File Mapping

| Current Location | New Location |
|-----------------|--------------|
| `server.js` lines 24-36 | `config/index.js` |
| `server.js` lines 45-54 | `middleware/requestLogger.js` |
| `server.js` lines 711-724 | `middleware/auth.js` |
| `server.js` lines 729-742 | `middleware/errorHandler.js` |
| `server.js` lines 60-76 | `routes/health.routes.js` |
| `server.js` lines 86-88 | `routes/bot.routes.js` |
| `server.js` lines 92-402 | `controllers/bot.controller.js` |
| `server.js` lines 424-617 | `controllers/webhook.controller.js` |
| `server.js` lines 629-661 | `controllers/proactive.controller.js` |
| `server.js` lines 670-702 | `controllers/admin.controller.js` |

## Testing Strategy

### Manual Testing Checklist
- [ ] `GET /` returns health status
- [ ] `GET /health` returns healthy
- [ ] `GET /ready` returns ready
- [ ] `POST /api/messages` accepts bot messages
- [ ] `POST /Messages` accepts bot messages (legacy)
- [ ] `POST /api/webhook/langflow` processes webhooks
- [ ] `POST /api/webhook/langflow/routing` processes routing
- [ ] `POST /api/proactive` sends proactive messages
- [ ] `GET /api/admin/stats` returns stats
- [ ] `POST /api/admin/clear-state` clears state
- [ ] `GET /api/admin/state/:id` returns state
- [ ] 404 handler works for unknown routes
- [ ] Error handler catches errors
- [ ] Graceful shutdown works (SIGTERM, SIGINT)

## Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| Breaking API contracts | High | Test all endpoints before/after |
| Import path issues | Medium | Use relative paths consistently |
| Middleware order issues | Medium | Document middleware order |
| Missing error handling | Medium | Test error scenarios |

## Rollback Plan

If issues arise:
1. The original `server.js` can be restored from git
2. No database or external service changes required
3. No dependency changes required
