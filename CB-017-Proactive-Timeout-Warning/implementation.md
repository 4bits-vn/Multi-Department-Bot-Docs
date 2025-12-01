# CB-017: Proactive Timeout Warning - Implementation

## Summary

Implemented a proactive timeout warning feature that automatically sends a message to users before their conversation times out, informing them about the upcoming timeout and that subsequent messages will start a new conversation.

## Files Created

### `/botframework/src/Services/ProactiveTimeoutService.js`

New service to manage proactive timeout warning timers:

- **Timer Management**: In-memory Map storage for conversation timers
- **Conversation Info Cache**: Stores user info for proactive messaging
- **Warning Message Generation**: Uses LangFlow OTHER flow with fallback
- **Automatic Cleanup**: Periodic cleanup of stale entries
- **Startup Rehydration**: Restores timers from storage after server restart

Key methods:
- `resetTimer(conversationId, info)`: Reset/schedule warning timer
- `clearTimer(conversationId)`: Clear timer on restart/timeout
- `handleTimeoutWarning(conversationId)`: Send warning message
- `generateWarningMessage(info)`: Generate warning via OTHER flow
- `rehydrateFromStorage(userService)`: Restore timers from storage on startup
- `getStats()`: Get service statistics

## Files Modified

### `/botframework/src/config/index.js`

Added new lifecycle configuration:

```javascript
lifecycle: {
    // ... existing config
    enableTimeoutWarning: process.env.ENABLE_TIMEOUT_WARNING === "true",
    timeoutWarningMinutes: parseInt(process.env.TIMEOUT_WARNING_MINUTES) || 5,
}
```

### `/botframework/src/utils/lifecycleMessages.js`

Added new message type for proactive timeout warning:

```javascript
// New message type
TIMEOUT_WARNING_PROACTIVE: 'TIMEOUT_WARNING_PROACTIVE'

// Category mapping
[LifecycleMessageType.TIMEOUT_WARNING_PROACTIVE]: MessageCategory.CONVERSATIONAL

// Prompt for OTHER flow
[LifecycleMessageType.TIMEOUT_WARNING_PROACTIVE]:
    'Send a friendly reminder that the user has been inactive...'

// Fallback message
[LifecycleMessageType.TIMEOUT_WARNING_PROACTIVE]: "⏰ Just a friendly reminder: Your conversation session will expire in a few minutes due to inactivity..."
```

### `/botframework/src/controllers/bot.controller.js`

Integrated ProactiveTimeoutService:

1. **Import and Initialize**:
```javascript
const { createProactiveTimeoutService } = require('../Services/ProactiveTimeoutService');
const { otherFlow } = require('../Handlers/LangFlow/flows');

const proactiveTimeoutService = createProactiveTimeoutService({
    msTeamsService,
    otherFlow
});
```

2. **Reset Timer After Message Processing**:
```javascript
// In handleRouterResponse cleanup section
proactiveTimeoutService.resetTimer(conversationId, {
    email: userEmail,
    serviceUrl,
    channel
});
```

3. **Clear Timer on Restart**:
```javascript
// When bot command restart is detected
proactiveTimeoutService.clearTimer(conversationId);

// When lifecycle check clears session
proactiveTimeoutService.clearTimer(conversationId);
```

### `/botframework/src/Services/UserService.js`

Added method to query active conversations:

```javascript
async getActiveConversations(options = {})
```

Returns conversations that:
- Have valid `conversationId` and `serviceUrl`
- Have activity within the specified timeframe
- Are not in IDLE or COMPLETED lifecycle state

### `/botframework/src/server.js`

Added startup rehydration:

```javascript
// Rehydrate timeout warning timers from storage (CB-017)
setTimeout(() => {
    rehydrateTimeoutWarnings();
}, 2000);
```

Added cleanup on shutdown:
```javascript
// Cleanup proactive timeout service (CB-017)
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
| `CONVERSATION_TIMEOUT_MINUTES` | `30` | Total conversation timeout (existing) |

### Example Configuration

```bash
# Enable timeout warning
ENABLE_TIMEOUT_WARNING=true

# Send warning 5 minutes before 30-minute timeout
TIMEOUT_WARNING_MINUTES=5
CONVERSATION_TIMEOUT_MINUTES=30

# Warning will be sent at 25 minutes of inactivity
```

## Flow Diagram

### Normal Operation Flow
```
User sends message
        ↓
Message processed successfully
        ↓
proactiveTimeoutService.resetTimer()
        ↓
Timer scheduled for (30-5=25) minutes
        ↓
... user inactive for 25 minutes ...
        ↓
handleTimeoutWarning() triggered
        ↓
Generate warning via OTHER flow
        ↓
Send proactive message to Teams
        ↓
"⏰ Your session will expire in 5 minutes..."
        ↓
... 5 more minutes of inactivity ...
        ↓
Session expires (normal timeout handling)
```

### Startup Rehydration Flow
```
Server starts
        ↓
Wait 2 seconds (allow services to initialize)
        ↓
rehydrateTimeoutWarnings() called
        ↓
Query active conversations from Azure Table Storage
        ↓
For each conversation:
├── Already timed out? → Skip (will handle on next user message)
├── Warning time passed? → Schedule immediate warning (1-10s delay)
└── Warning time not yet? → Schedule timer for remaining time
        ↓
Timers rehydrated, warnings will be sent as scheduled
```

## Key Design Decisions

1. **In-Memory Timer Storage with Rehydration**: Using JavaScript `setTimeout` with `Map` for simple, efficient timer management. On server restart, timers are rehydrated from Azure Table Storage by checking the `lastActivityAt` timestamp of active conversations.

2. **OTHER Flow for Messages**: The warning message is generated by the LangFlow OTHER flow to maintain consistency with other bot messages and allow for LLM-generated, contextual responses.

3. **Single Warning Per Session**: Only one warning is sent per timeout period. The `warningSent` flag prevents duplicate warnings.

4. **Non-Blocking**: Timer operations don't block message processing. Proactive message failures are logged but don't affect user experience.

5. **Configurable Timing**: Both the timeout and warning time are configurable via environment variables.

## Testing

### Quick Test Configuration

```bash
# Short timeouts for testing
ENABLE_TIMEOUT_WARNING=true
CONVERSATION_TIMEOUT_MINUTES=3
TIMEOUT_WARNING_MINUTES=1

# Warning sent after 2 minutes of inactivity
# Timeout at 3 minutes
```

### Test Scenarios

| Scenario | Expected Result |
|----------|-----------------|
| Enable feature, wait 2 min | Warning message sent |
| Send message before warning | Timer resets, no warning |
| Type "restart" | Timer cleared |
| Send another message after warning | Normal processing continues |
| Feature disabled | No warnings sent |

## Dependencies

- `MSTeamsService`: For sending proactive messages
- `OtherFlow`: For generating warning messages via LangFlow
- `config/index.js`: For configuration settings

## Notes

- The feature is **disabled by default** (`ENABLE_TIMEOUT_WARNING=false`)
- Warning time must be less than conversation timeout time
- If warning time >= timeout time, feature is effectively disabled
- Timers are automatically cleaned up for stale conversations
