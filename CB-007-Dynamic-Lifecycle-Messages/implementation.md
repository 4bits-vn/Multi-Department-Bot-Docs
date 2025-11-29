# CB-007: Dynamic Lifecycle Messages - Implementation

## Summary

Replaced hardcoded lifecycle messages with dynamically generated messages via LangFlow flows. Conversational messages (greetings, goodbyes, follow-ups) are routed to the CHITCHAT flow, while error messages are routed to the EXCEPTION flow. Static fallback messages are used when LangFlow is unavailable.

## Files Created

### `botframework/src/Services/LifecycleMessageService.js`

New service that provides a unified API for getting lifecycle messages:

```javascript
const { lifecycleMessageService } = require('./Services/LifecycleMessageService');

// Get a simple message
const message = await lifecycleMessageService.getMessage(
  LifecycleMessageType.GOODBYE,
  { userEmail: 'user@example.com', userName: 'John' }
);

// Get resolution with follow-up question
const messageWithFollowUp = await lifecycleMessageService.getResolutionWithFollowUp(
  resolutionMessage,
  { includeHint: false, userEmail }
);

// Get error message
const errorMessage = await lifecycleMessageService.getErrorMessage('MAX_ITERATIONS', {
  userEmail,
  errorDetails: 'Processing limit reached'
});
```

Key features:
- Routes to appropriate LangFlow flow based on message type
- Timeout protection (5 second default)
- Optional caching
- Graceful fallback to static messages

## Files Modified

### `botframework/src/utils/lifecycleMessages.js`

Refactored to support dynamic message generation:

**New exports:**
- `LifecycleMessageType` - Enum of all message types
- `MessageCategory` - CONVERSATIONAL or ERROR
- `MESSAGE_TYPE_CATEGORY` - Mapping of types to categories
- `CHITCHAT_PROMPTS` - Prompts for CHITCHAT flow
- `EXCEPTION_PROMPTS` - Prompts for EXCEPTION flow
- `FALLBACK_MESSAGES` - Static fallback messages
- Helper functions: `getFallbackMessage`, `getMessageCategory`, `getChitChatPrompt`, `getExceptionPrompt`, etc.

**Legacy exports preserved** for backward compatibility:
- `LIFECYCLE_MESSAGES` (alias for FALLBACK_MESSAGES)
- `getMessage`
- `buildResolutionWithFollowUp`
- `buildErrorMessage`

### `botframework/src/Handlers/LangFlowHandler.js`

Added lifecycle message generation methods:

```javascript
// Main entry point
async generateLifecycleMessage({ messageType, sessionId, userEmail, context })

// Conversational messages via CHITCHAT flow
async generateConversationalMessage(messageType, sessionId, userEmail, context)

// Error messages via EXCEPTION flow
async generateErrorMessage(messageType, userEmail, context)
```

### `botframework/src/Services/ConversationLifecycleService.js`

Updated to use LifecycleMessageService:

- Added `lifecycleMessageService` dependency
- Added `getLifecycleMessage()` helper method
- Updated all handler methods to use dynamic messages:
  - `handleTimeout()` → `LifecycleMessageType.SESSION_EXPIRED`
  - `handleErrorRecovery()` → `LifecycleMessageType.ERROR_RECOVERY`
  - `handleRestartIntent()` → `LifecycleMessageType.RESTART_CONFIRMED`
  - `handleNeedMoreHelpResponse()` → `LifecycleMessageType.GOODBYE` / `HOW_CAN_I_HELP`
  - `handleSystemError()` → Error-specific message types

### `botframework/src/controllers/bot.controller.js`

Updated to initialize and use services:

```javascript
// Initialize lifecycle message service
const lifecycleMessageService = createLifecycleMessageService({
    langFlowHandler
});

// Pass to lifecycle service
const lifecycleService = createConversationLifecycleService({
    userService,
    conversationStateService,
    lifecycleMessageService
});
```

Updated message usages:
- Bot command restart → `lifecycleMessageService.getMessage(LifecycleMessageType.RESTART_CONFIRMED)`
- Still processing → `lifecycleMessageService.getMessage(LifecycleMessageType.STILL_PROCESSING)`
- Resolution follow-up → `lifecycleMessageService.getResolutionWithFollowUp()`
- Generic error → `lifecycleMessageService.getErrorMessage('GENERIC')`

## Message Flow Diagram

```
User Action
    │
    ▼
┌─────────────────┐
│ bot.controller  │
│ needs message   │
└────────┬────────┘
         │
         ▼
┌─────────────────────────────┐
│ LifecycleMessageService     │
│ getMessage(type, options)   │
└────────┬────────────────────┘
         │
         ├─── Dynamic enabled? ─── No ──▶ Return FALLBACK_MESSAGES[type]
         │
         ▼ Yes
         │
         ├─── isConversationalMessage(type)?
         │    │
         │    ▼ Yes
         │    LangFlowHandler.generateConversationalMessage()
         │         │
         │         ▼
         │    ┌────────────────┐
         │    │  CHITCHAT Flow │ ──▶ Generate response
         │    └────────────────┘
         │
         ├─── isErrorMessage(type)?
         │    │
         │    ▼ Yes
         │    LangFlowHandler.generateErrorMessage()
         │         │
         │         ▼
         │    ┌─────────────────┐
         │    │ EXCEPTION Flow  │ ──▶ Generate response
         │    └─────────────────┘
         │
         ▼
┌─────────────────────────────┐
│ Return message to caller    │
│ (dynamic or fallback)       │
└─────────────────────────────┘
```

## Configuration

| Environment Variable | Default | Description |
|---------------------|---------|-------------|
| `ENABLE_DYNAMIC_LIFECYCLE_MESSAGES` | `true` | Enable dynamic message generation |
| `LIFECYCLE_MESSAGE_TIMEOUT` | `5000` | Timeout in milliseconds |
| `LIFECYCLE_MESSAGE_CACHE_TTL` | `0` | Cache TTL (0 = disabled) |

## Testing

### Manual Testing

1. **Restart command**: Type "restart" and verify message is generated via CHITCHAT flow
2. **Goodbye**: After solution, say "no" to more help - verify goodbye message
3. **Error scenario**: Force a system error - verify error message via EXCEPTION flow
4. **Fallback**: Disable CHITCHAT flow and verify fallback messages are used

### Logging

Watch for log entries:
- `"Generating conversational message via CHITCHAT"` - Dynamic generation
- `"Successfully generated lifecycle message"` - Success
- `"Using fallback message"` - Fallback used
- `"Error generating conversational message"` - Error (fallback will be used)

## Known Limitations

1. **No caching by default**: Each message is generated fresh (can enable via env var)
2. **No personalization persistence**: Context is per-request only
3. **Session ID for lifecycle messages**: Uses temporary session ID, doesn't persist conversation

## Future Improvements

1. **Localization support**: Detect user language and generate localized messages
2. **Message variants**: Store multiple fallback variants for variety
3. **Analytics**: Track message generation success rates
4. **A/B testing**: Compare dynamic vs static message effectiveness

## Related Documentation

- [CB-006 Conversation Lifecycle](../CB-006-Conversation-Lifecycle/implementation.md)
- [CHITCHAT Flow README](../../flows/CHITCHAT/README.md)
- [EXCEPTION Flow README](../../flows/EXCEPTION/README.md)
