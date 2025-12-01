# CB-017: Proactive Timeout Warning & Expiration - Implementation

## Summary

Implemented a proactive timeout system that:
1. Sends a warning message before conversation timeout
2. Sends an expiration message AND closes the conversation when timeout occurs
3. Allows users to return to a fresh conversation without any reactive "session expired" messages

## Files Created

### `/botframework/src/Services/ProactiveTimeoutService.js`

New service to manage proactive timeout warning and expiration:

**Timer Storage:**
- `timers`: Warning timer map (conversationId → timeoutId)
- `expirationTimers`: Expiration timer map (conversationId → timeoutId)
- `conversationInfo`: Cache for user info (conversationId → {email, serviceUrl, etc.})
- `warningSent`: Track warnings sent (conversationId → timestamp)
- `sessionExpired`: Track sessions expired proactively (conversationId → timestamp)

**Key Methods:**
- `resetTimer(conversationId, info)`: Reset/schedule warning timer
- `clearTimer(conversationId)`: Clear all timers (warning + expiration)
- `handleTimeoutWarning(conversationId)`: Send warning message, schedule expiration
- `scheduleExpirationTimer(conversationId, info)`: Schedule expiration after warning
- `handleSessionExpiration(conversationId)`: Send expiration message, close session
- `generateWarningMessage(info)`: Generate warning via OTHER flow
- `generateExpirationMessage(info)`: Generate expiration via OTHER flow
- `rehydrateFromStorage(userService)`: Restore timers from storage on startup
- `getStats()`: Get service statistics
- `shutdown()`: Clean up all timers

## Files Modified

### `/botframework/src/config/index.js`

Added lifecycle configuration:

```javascript
lifecycle: {
    conversationTimeoutMinutes: parseInt(process.env.CONVERSATION_TIMEOUT_MINUTES) || 30,
    enableTimeoutWarning: process.env.ENABLE_TIMEOUT_WARNING === "true",
    timeoutWarningMinutes: parseInt(process.env.TIMEOUT_WARNING_MINUTES) || 5,
}
```

### `/botframework/src/utils/lifecycleMessages.js`

Added message type for proactive timeout warning:

```javascript
// New message type
TIMEOUT_WARNING_PROACTIVE: 'TIMEOUT_WARNING_PROACTIVE'

// Category mapping
[LifecycleMessageType.TIMEOUT_WARNING_PROACTIVE]: MessageCategory.CONVERSATIONAL

// Prompt for OTHER flow (warning)
[LifecycleMessageType.TIMEOUT_WARNING_PROACTIVE]:
    'Send a friendly reminder in 2 sentences, that the user has been inactive...'

// Fallback message (warning)
[LifecycleMessageType.TIMEOUT_WARNING_PROACTIVE]: "⏰ Just a friendly reminder..."
```

Updated SESSION_EXPIRED prompt (used for expiration message):

```javascript
// Prompt for OTHER flow (expiration - used by ProactiveTimeoutService)
[LifecycleMessageType.SESSION_EXPIRED]:
    'Start a fresh conversation with the user. Keep it brief...'
```

### `/botframework/src/controllers/bot.controller.js`

Integrated ProactiveTimeoutService:

```javascript
const { createProactiveTimeoutService } = require('../Services/ProactiveTimeoutService');
const { otherFlow } = require('../Handlers/LangFlow/flows');

const proactiveTimeoutService = createProactiveTimeoutService({
    msTeamsService,
    otherFlow,
    userService
});
```

Reset timer after message processing:
```javascript
proactiveTimeoutService.resetTimer(conversationId, {
    email: userEmail,
    serviceUrl,
    channel
});
```

Clear timer on restart:
```javascript
proactiveTimeoutService.clearTimer(conversationId);
```

### `/botframework/src/Services/ConversationLifecycleService.js`

Updated `handleTimeout()` to return no message:

```javascript
async handleTimeout(params, timeoutCheck) {
    // Clear session and reset state
    await this.clearConversation(email, channel, EndReason.TIMEOUT);

    // CB-017: No message needed - expiration is handled proactively
    return {
        handled: true,
        action: 'timeout_recovery',
        message: null, // No reactive message
        shouldContinue: true,
        metadata: { timeoutType, sessionCleared: true }
    };
}
```

### `/botframework/src/Services/ConversationRoutingService.js`

Removed SESSION_EXPIRED message and updated `handleExpiredLock()`:

```javascript
async handleExpiredLock(email, channel, message, startTime) {
    await this.releaseRoute(email, channel, 'timeout');
    const classification = await this.classifyMessage(message);
    // ...
    return {
        success: true,
        route: classification.route,
        handled: false, // Don't mark as handled - let flow continue
        action: 'GO',   // No message, just continue
        routingTimeMs: Date.now() - startTime
    };
}
```

### `/botframework/src/Services/UserService.js`

Added method to query active conversations:

```javascript
async getActiveConversations(options = {}) {
    const { maxAgeMs, channel } = options;
    // Query conversations with valid conversationId, serviceUrl
    // Filter by lastActivityAt within timeframe
    // Return for timeout rehydration
}
```

### `/botframework/src/server.js`

Added startup rehydration:

```javascript
setTimeout(() => {
    rehydrateTimeoutWarnings();
}, 2000);
```

Added shutdown cleanup:

```javascript
const proactiveTimeoutService = getProactiveTimeoutService();
if (proactiveTimeoutService) {
    proactiveTimeoutService.shutdown();
}
```

## Configuration

### Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `ENABLE_TIMEOUT_WARNING` | `false` | Enable/disable the feature |
| `TIMEOUT_WARNING_MINUTES` | `5` | Minutes before timeout to send warning |
| `CONVERSATION_TIMEOUT_MINUTES` | `30` | Total conversation timeout |

### Example Configuration

```bash
# Enable timeout warning & expiration
ENABLE_TIMEOUT_WARNING=true

# Send warning 5 minutes before 30-minute timeout
TIMEOUT_WARNING_MINUTES=5
CONVERSATION_TIMEOUT_MINUTES=30

# Timeline:
# - Warning sent at 25 minutes of inactivity
# - Expiration sent at 30 minutes of inactivity
# - Conversation closed at 30 minutes
```

## Flow Diagram

### Complete Flow

```
User sends message
        ↓
proactiveTimeoutService.resetTimer()
        ↓
Warning timer scheduled for (timeout - warning) minutes
        ↓
... user inactive for (timeout - warning) minutes ...
        ↓
handleTimeoutWarning() fires
        ↓
├── Generate warning via OTHER flow
├── Send proactive message to Teams
├── Schedule expiration timer (warning minutes later)
└── Mark warning as sent
        ↓
"⏰ Your session will expire in X minutes..."
        ↓
... user continues to be inactive ...
        ↓
handleSessionExpiration() fires
        ↓
├── Check if user became active after warning
├── If still inactive:
│   ├── Generate expiration via OTHER flow
│   ├── Send proactive message to Teams
│   ├── userService.clearSession()
│   ├── Set lifecycle_state = TIMED_OUT
│   └── Mark session as expired
└── Clean up timers
        ↓
"⏰ Your session has expired..."
Conversation CLOSED
        ↓
... some time later ...
        ↓
User sends new message
        ↓
ConversationLifecycleService.handleTimeout()
├── Clears session (if needed)
├── Returns message: null (no reactive message!)
└── shouldContinue: true
        ↓
Bot responds normally to user's message
(NO "session expired" message - just fresh start!)
```

### Startup Rehydration Flow

```
Server starts
        ↓
Wait 2 seconds (services initialize)
        ↓
rehydrateTimeoutWarnings()
        ↓
Query active conversations from Azure Table
        ↓
For each conversation:
├── Already timed out? → Skip
├── Warning time passed? → Schedule immediate warning (1-10s delay)
└── Warning time not yet? → Schedule future warning timer
        ↓
Timers rehydrated
```

## Key Design Decisions

1. **Two-Phase Timeout**: Warning timer fires first, then schedules expiration timer. This allows users to become active after warning and cancel expiration.

2. **Proactive Session Closure**: When expiration timer fires, the session is closed proactively (clearSession + TIMED_OUT state). No need for reactive handling.

3. **No Reactive Messages**: `ConversationLifecycleService.handleTimeout()` returns `message: null`. Users returning after expiration see no "session expired" message.

4. **Activity Check Before Expiration**: Before sending expiration message, check if user became active after warning. If so, cancel expiration.

5. **OTHER Flow for Messages**: Both warning and expiration messages generated by LangFlow OTHER flow for consistency and natural language.

6. **Rehydration on Restart**: Query Azure Table Storage for active conversations and re-establish timers based on lastActivityAt timestamps.

7. **Clean Timer Management**: Both warning and expiration timers cleared together. Automatic cleanup of stale entries via periodic interval.

## Testing

### Quick Test Configuration

```bash
ENABLE_TIMEOUT_WARNING=true
CONVERSATION_TIMEOUT_MINUTES=3
TIMEOUT_WARNING_MINUTES=1

# Timeline:
# - Warning at 2 minutes of inactivity
# - Expiration at 3 minutes of inactivity
```

### Test Scenarios

| Scenario | Expected Result |
|----------|-----------------|
| Wait for warning time | Warning message sent |
| Wait for expiration time | Expiration message sent, session closed |
| Send message before warning | Timer resets, no warning |
| Send message after warning | Timer resets, no expiration |
| Return after expiration | Fresh conversation, NO extra message |
| Type "restart" | All timers cleared |
| Feature disabled | No warnings or expirations |
| Server restart | Timers rehydrated from storage |

## Dependencies

- `MSTeamsService`: For sending proactive messages
- `OtherFlow`: For generating warning/expiration messages
- `UserService`: For session management and conversation queries
- `config/index.js`: For configuration settings

## Notes

- Feature is **disabled by default** (`ENABLE_TIMEOUT_WARNING=false`)
- Warning time must be less than conversation timeout time
- If user becomes active after warning, expiration is cancelled
- Session is closed proactively, so reactive handling just clears any remaining state
- Timers are in-memory but rehydrated from storage on restart
