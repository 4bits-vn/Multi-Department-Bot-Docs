# CB-017: Proactive Timeout Warning & Expiration - Implementation Plan

## Overview

This document outlines the implementation plan for the proactive timeout warning and expiration feature.

## Technical Approach

### Architecture

```
┌───────────────────────────────────────────────────────────────────┐
│                        Bot Controller                              │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │  handleTeamsMessage()                                        │  │
│  │    ├─ Process message                                        │  │
│  │    └─ Call proactiveTimeoutService.resetTimer()             │  │
│  └─────────────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────────────┘
                               │
                               ↓
┌───────────────────────────────────────────────────────────────────┐
│                   ProactiveTimeoutService                          │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │  Timer Storage:                                              │  │
│  │  - timers: Map (conversationId → warningTimeoutId)          │  │
│  │  - expirationTimers: Map (conversationId → expirationId)    │  │
│  │  - conversationInfo: Map (conversationId → {email, etc.})   │  │
│  │  - warningSent: Map (conversationId → timestamp)            │  │
│  │  - sessionExpired: Map (conversationId → timestamp)         │  │
│  │                                                              │  │
│  │  Methods:                                                    │  │
│  │  - resetTimer(conversationId, info)                         │  │
│  │  - clearTimer(conversationId)                               │  │
│  │  - handleTimeoutWarning(conversationId)                     │  │
│  │  - scheduleExpirationTimer(conversationId, info)            │  │
│  │  - handleSessionExpiration(conversationId)                  │  │
│  └─────────────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────────────┘
                               │
                               ↓
┌───────────────────────────────────────────────────────────────────┐
│  On Warning Timer Fire:                                           │
│  1. Get conversation info from cache                              │
│  2. Generate warning message via OtherFlow                        │
│  3. Send proactive message via MSTeamsService                     │
│  4. Schedule expiration timer (warningMinutes later)              │
│  5. Mark warning as sent                                          │
└───────────────────────────────────────────────────────────────────┘
                               │
                               ↓
┌───────────────────────────────────────────────────────────────────┐
│  On Expiration Timer Fire:                                        │
│  1. Check if user became active after warning                     │
│  2. If still inactive:                                            │
│     a. Generate expiration message via OtherFlow                  │
│     b. Send proactive message via MSTeamsService                  │
│     c. Clear session via UserService                              │
│     d. Set lifecycle_state = TIMED_OUT                            │
│     e. Mark session as expired                                    │
│  3. Clean up timers and caches                                    │
└───────────────────────────────────────────────────────────────────┘
```

### Components

#### 1. Configuration (`config/index.js`)

Lifecycle configuration:
- `timeoutWarningMinutes`: Minutes before timeout to send warning (default: 5)
- `enableTimeoutWarning`: Feature flag
- `conversationTimeoutMinutes`: Total conversation timeout (existing)

#### 2. ProactiveTimeoutService (`Services/ProactiveTimeoutService.js`)

Service to manage timeout warning and expiration timers:

```javascript
class ProactiveTimeoutService {
    constructor(deps) {
        this.msTeamsService = deps.msTeamsService;
        this.otherFlow = deps.otherFlow;
        this.userService = deps.userService;

        this.timers = new Map();           // conversationId → warningTimeoutId
        this.expirationTimers = new Map(); // conversationId → expirationTimeoutId
        this.conversationInfo = new Map(); // conversationId → info
        this.warningSent = new Map();      // conversationId → timestamp
        this.sessionExpired = new Map();   // conversationId → timestamp
    }

    resetTimer(conversationId, info) { /* ... */ }
    clearTimer(conversationId) { /* ... */ }
    handleTimeoutWarning(conversationId) { /* ... */ }
    scheduleExpirationTimer(conversationId, info) { /* ... */ }
    handleSessionExpiration(conversationId) { /* ... */ }
    generateWarningMessage(info) { /* ... */ }
    generateExpirationMessage(info) { /* ... */ }
    rehydrateFromStorage(userService) { /* ... */ }
}
```

#### 3. Lifecycle Messages (`utils/lifecycleMessages.js`)

Message types for timeout handling:
- `TIMEOUT_WARNING_PROACTIVE`: Prompt for warning message
- `SESSION_EXPIRED`: Prompt for expiration message (used by proactive service)

#### 4. ConversationLifecycleService Changes

- `handleTimeout()` returns `message: null` - no reactive message needed
- Session clearing and state update handled by ProactiveTimeoutService

#### 5. ConversationRoutingService Changes

- `handleExpiredLock()` returns `action: 'GO'` - no message
- Route lock expiration handled silently (conversation already expired proactively)

## Implementation Steps

### Step 1: Update Configuration

File: `botframework/src/config/index.js`

```javascript
lifecycle: {
    conversationTimeoutMinutes: parseInt(process.env.CONVERSATION_TIMEOUT_MINUTES) || 30,
    enableTimeoutWarning: process.env.ENABLE_TIMEOUT_WARNING === "true",
    timeoutWarningMinutes: parseInt(process.env.TIMEOUT_WARNING_MINUTES) || 5,
}
```

### Step 2: Update Lifecycle Messages

File: `botframework/src/utils/lifecycleMessages.js`

Add:
- `TIMEOUT_WARNING_PROACTIVE` message type
- Prompts for OTHER flow (warning and expiration)
- Fallback messages

### Step 3: Create ProactiveTimeoutService

File: `botframework/src/Services/ProactiveTimeoutService.js`

Features:
- Warning timer management
- Expiration timer management
- Warning message generation via OTHER flow
- Expiration message generation via OTHER flow
- Session closure on expiration
- Proactive message sending via MSTeamsService
- Automatic cleanup of stale entries
- Startup rehydration from storage

### Step 4: Integrate with Bot Controller

File: `botframework/src/controllers/bot.controller.js`

- Import and initialize ProactiveTimeoutService with userService
- Call `resetTimer()` on successful message processing
- Call `clearTimer()` on conversation restart

### Step 5: Update ConversationLifecycleService

File: `botframework/src/Services/ConversationLifecycleService.js`

- `handleTimeout()` returns `message: null`
- No reactive timeout message (handled proactively)

### Step 6: Update ConversationRoutingService

File: `botframework/src/Services/ConversationRoutingService.js`

- Remove `SESSION_EXPIRED` message
- `handleExpiredLock()` returns `action: 'GO'` (no message)

### Step 7: Add Startup Rehydration

File: `botframework/src/server.js`

- Rehydrate timers from Azure Table Storage on startup
- Schedule warnings based on lastActivityAt timestamps

### Step 8: Add Graceful Shutdown

File: `botframework/src/server.js`

- Clear all timers on shutdown
- Clean up resources

## Testing

### Manual Testing Checklist

| Scenario | Expected Result |
|----------|-----------------|
| Enable feature, wait for warning time | Warning message sent |
| Wait for expiration time | Expiration message sent, session closed |
| Send message before warning | Timer resets, no warning |
| Send message after warning, before expiration | Timer resets, no expiration |
| Return after expiration | Fresh conversation, NO "session expired" message |
| Type "restart" | All timers cleared |
| Disable feature | No warnings or expirations sent |
| Server restart with active conversations | Timers rehydrated |

### Test Configuration

```bash
# Quick testing with short timeouts
ENABLE_TIMEOUT_WARNING=true
CONVERSATION_TIMEOUT_MINUTES=3
TIMEOUT_WARNING_MINUTES=1

# Timeline:
# - Warning sent at 2 minutes of inactivity
# - Expiration sent at 3 minutes of inactivity
```

## Rollout Plan

1. **Phase 1**: Deploy with feature disabled (default)
2. **Phase 2**: Enable in development environment for testing
3. **Phase 3**: Enable in staging with real users
4. **Phase 4**: Enable in production

## Configuration Recommendations

### Production Settings

```bash
ENABLE_TIMEOUT_WARNING=true
CONVERSATION_TIMEOUT_MINUTES=30
TIMEOUT_WARNING_MINUTES=5
```

This sends:
- Warning at 25 minutes of inactivity
- Expiration at 30 minutes of inactivity

## External Dependencies

### LangFlow OTHER Flow

The warning and expiration messages are generated by the OTHER flow:
- Warning prompt: Friendly reminder about upcoming timeout
- Expiration prompt: Brief notification that session has ended

### MS Teams Proactive Messaging

Uses `MSTeamsService.sendProactiveMessage()`:
- Requires `serviceUrl` and `conversationId`
- Handles authentication automatically
- Supports message text

### UserService

Used for:
- Clearing session on expiration
- Updating lifecycle_state to TIMED_OUT
- Querying active conversations for rehydration

## Files Changed

| File | Changes |
|------|---------|
| `config/index.js` | Add `timeoutWarningMinutes` config |
| `utils/lifecycleMessages.js` | Add `TIMEOUT_WARNING_PROACTIVE` type and prompts |
| `Services/ProactiveTimeoutService.js` | New service (warning + expiration) |
| `Services/UserService.js` | Add `getActiveConversations()` method |
| `Services/ConversationLifecycleService.js` | `handleTimeout()` returns no message |
| `Services/ConversationRoutingService.js` | Remove SESSION_EXPIRED, update handleExpiredLock |
| `controllers/bot.controller.js` | Integrate timeout service |
| `server.js` | Add rehydration and shutdown handling |

## Risks and Mitigations

| Risk | Mitigation |
|------|------------|
| Timer memory leaks | Automatic cleanup of stale timers, graceful shutdown |
| Service restart loses timers | Rehydrate from Azure Table Storage |
| LangFlow down during warning/expiration | Use fallback messages |
| User active after warning | Check lastActivity before expiration, cancel if active |
| Duplicate messages | Track warningSent and sessionExpired timestamps |
| Message rate limits | Single warning and single expiration per timeout period |
