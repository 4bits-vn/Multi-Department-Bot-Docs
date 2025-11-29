# CB-003: Express.js Project Structure Restructure

## Ticket Information

| Field | Value |
|-------|-------|
| Ticket | CB-003 |
| Type | Chore / Refactoring |
| Priority | Medium |
| Status | In Progress |

## Overview

Restructure the `botframework/src/server.js` file (808 lines) following Express.js best practices for project organization. The current monolithic server file contains all routes, controllers, middleware, and configuration in a single file, making it difficult to maintain and scale.

## Current State Analysis

### Current File Structure
```
botframework/src/
├── server.js           # 808 lines - Contains everything
├── Handlers/
│   └── LangFlowHandler.js
├── Services/
│   ├── Az.AppInsights.js
│   ├── Az.StorageAccount.js
│   ├── Az.StorageAccount.Table.js
│   ├── ConversationStateService.js
│   ├── Logger.js
│   ├── MSTeamsService.js
│   ├── ServiceNowService.js
│   └── UserService.js
└── utils/
    ├── DACoreUtils.js
    ├── index.js
    └── uuid.js
```

### Issues with Current Structure
1. **Monolithic server.js**: All logic in one 808-line file
2. **Mixed concerns**: Configuration, routes, controllers, middleware all together
3. **Difficult to test**: Hard to unit test individual components
4. **Poor maintainability**: Changes require navigating a large file
5. **No separation of concerns**: Business logic mixed with routing

## Functional Requirements

### FR-001: Separation of Concerns
- Configuration should be in dedicated config module
- Middleware should be in separate files
- Routes should be organized by domain
- Controllers should handle request/response logic
- Business logic should remain in Services

### FR-002: Maintain All Existing Functionality
- All existing endpoints must work identically
- Health checks: `/`, `/health`, `/ready`
- Bot messages: `/api/messages`, `/Messages`
- Webhooks: `/api/webhook/langflow`, `/api/webhook/langflow/routing`
- Proactive: `/api/proactive`
- Admin: `/api/admin/stats`, `/api/admin/clear-state`, `/api/admin/state/:id`

### FR-003: Follow Express Best Practices
- Use Router for route organization
- Centralized error handling
- Middleware composition
- Environment-based configuration

## Non-Functional Requirements

### NFR-001: Maintainability
- Each file should have a single responsibility
- Files should be under 200 lines where possible
- Clear naming conventions

### NFR-002: Testability
- Controllers should be easily testable
- Middleware should be testable in isolation
- No hidden dependencies

### NFR-003: Scalability
- Structure should support adding new routes easily
- New middleware can be added without modifying existing code

## Target Structure

```
botframework/src/
├── app.js                      # Express app configuration
├── server.js                   # Server startup only (minimal)
├── config/
│   └── index.js               # Centralized configuration
├── middleware/
│   ├── index.js               # Middleware exports
│   ├── requestLogger.js       # Request logging middleware
│   ├── errorHandler.js        # Error handling middleware
│   └── auth.js                # API key verification
├── routes/
│   ├── index.js               # Route aggregator
│   ├── health.routes.js       # Health endpoints
│   ├── bot.routes.js          # Bot message routes
│   ├── webhook.routes.js      # LangFlow webhook routes
│   ├── proactive.routes.js    # Proactive message routes
│   └── admin.routes.js        # Admin routes
├── controllers/
│   ├── bot.controller.js      # Bot message handlers
│   ├── webhook.controller.js  # Webhook handlers
│   ├── proactive.controller.js # Proactive message handlers
│   └── admin.controller.js    # Admin handlers
├── Handlers/                  # Existing (unchanged)
├── Services/                  # Existing (unchanged)
└── utils/                     # Existing (unchanged)
```

## Acceptance Criteria

- [ ] All existing endpoints return identical responses
- [ ] No breaking changes to API contracts
- [ ] Each new file has clear single responsibility
- [ ] server.js is under 50 lines
- [ ] app.js handles Express configuration only
- [ ] Routes are organized by domain
- [ ] Controllers handle request/response only
- [ ] Middleware is reusable and composable
- [ ] Application starts and runs correctly
- [ ] Graceful shutdown still works

## Dependencies

- Express.js v5.1.0 (current)
- No new dependencies required

## References

- [Express.js Best Practices](https://expressjs.com/en/advanced/best-practice-performance.html)
- [Express.js Router](https://expressjs.com/en/guide/routing.html)
- [Node.js Project Structure Best Practices](https://github.com/goldbergyoni/nodebestpractices)
