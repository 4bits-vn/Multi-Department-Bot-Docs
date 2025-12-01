# CB-017: Proactive Timeout Warning - Implementation Plan

## Overview

This document outlines the implementation plan for the proactive timeout warning feature.

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
│  │  - In-memory timer map (conversationId → timeoutId)         │  │
│  │  - Conversation info cache (conversationId → {email, etc.}) │  │
│  │  - resetTimer(conversationId, conversationInfo)             │  │
│  │  - clearTimer(conversationId)                               │  │
│  │  - handleTimeoutWarning(conversationId)                     │  │
│  └─────────────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────────────┘
                               │
                               ↓
┌───────────────────────────────────────────────────────────────────┐
│  On Timeout Warning:                                               │
│  1. Get conversation info from cache                              │
│  2. Call OtherFlow to generate warning message                    │
│  3. Send proactive message via MSTeamsService                     │
│  4. Log warning sent                                              │
└───────────────────────────────────────────────────────────────────┘
```

### Components

#### 1. Configuration (`config/index.js`)

Add new lifecycle configuration:
- `timeoutWarningMinutes`: Minutes before timeout to send warning (default: 5)
- `enableTimeoutWarning`: Feature flag (existing, change default to true when ready)

#### 2. ProactiveTimeoutService (`Services/ProactiveTimeoutService.js`)

New service to manage timeout warning timers:

```javascript
class ProactiveTimeoutService {
    constructor(deps) {
        this.msTeamsService = deps.msTeamsService;
        this.langFlowHandler = deps.langFlowHandler;
        this.timers = new Map();         // conversationId → timeoutId
        this.conversationInfo = new Map(); // conversationId → info
    }

    resetTimer(conversationId, info) { /* ... */ }
    clearTimer(conversationId) { /* ... */ }
    handleTimeoutWarning(conversationId) { /* ... */ }
}
```

#### 3. Lifecycle Messages (`utils/lifecycleMessages.js`)

Add prompt for timeout warning:
- `TIMEOUT_WARNING_PROACTIVE`: Prompt for OTHER flow
- Fallback message for when LangFlow unavailable

#### 4. Bot Controller Integration

On each message:
```javascript
// After validating user and processing message
proactiveTimeoutService.resetTimer(conversationId, {
    email: userEmail,
    serviceUrl,
    conversationId,
    channel
});
```

## Implementation Steps

### Step 1: Update Configuration

File: `botframework/src/config/index.js`

```javascript
lifecycle: {
    // ... existing config
    timeoutWarningMinutes: parseInt(process.env.TIMEOUT_WARNING_MINUTES) || 5,
    enableTimeoutWarning: process.env.ENABLE_TIMEOUT_WARNING === 'true',
}
```

### Step 2: Update Lifecycle Messages

File: `botframework/src/utils/lifecycleMessages.js`

Add:
- `TIMEOUT_WARNING_PROACTIVE` message type
- Prompt for OTHER flow
- Fallback message

### Step 3: Create ProactiveTimeoutService

File: `botframework/src/Services/ProactiveTimeoutService.js`

Features:
- Timer management with Map
- Conversation info caching
- Warning message generation via OTHER flow
- Proactive message sending via MSTeamsService
- Automatic cleanup of stale entries
- Configuration-based enable/disable

### Step 4: Integrate with Bot Controller

File: `botframework/src/controllers/bot.controller.js`

- Import and initialize ProactiveTimeoutService
- Call `resetTimer()` on successful message processing
- Call `clearTimer()` on conversation restart/end

### Step 5: Handle Edge Cases

- Clear timer on conversation restart
- Clear timer on conversation timeout (already handled)
- Handle proactive message failures gracefully
- Don't send warning if conversation already ended

## Testing

### Manual Testing Checklist

| Scenario | Expected Result |
|----------|-----------------|
| Enable feature, wait for warning time | Warning message sent |
| Send message before warning | Timer resets, no warning |
| Restart conversation | Timer cleared |
| Disable feature | No warnings sent |
| Warning time > timeout time | Feature effectively disabled |
| LangFlow unavailable | Fallback message sent |

### Test Configuration

```bash
# Quick testing with short timeouts
ENABLE_TIMEOUT_WARNING=true
CONVERSATION_TIMEOUT_MINUTES=2
TIMEOUT_WARNING_MINUTES=1
# Warning sent after 1 minute, timeout at 2 minutes
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

This sends a warning at 25 minutes of inactivity, with timeout at 30 minutes.

## External Dependencies

### LangFlow OTHER Flow

The warning message is generated by calling the OTHER flow with a specific prompt about timeout warning. The flow should:
- Receive a prompt describing the timeout situation
- Generate a friendly, helpful warning message
- Include timing information if provided

### MS Teams Proactive Messaging

Uses existing `MSTeamsService.sendProactiveMessage()` method which:
- Requires `serviceUrl` and `conversationId`
- Handles authentication automatically
- Supports message text

## Files Changed

| File | Changes |
|------|---------|
| `config/index.js` | Add `timeoutWarningMinutes` config |
| `utils/lifecycleMessages.js` | Add `TIMEOUT_WARNING_PROACTIVE` type |
| `Services/ProactiveTimeoutService.js` | New service (create) |
| `controllers/bot.controller.js` | Integrate timeout service |

## Risks and Mitigations

| Risk | Mitigation |
|------|------------|
| Timer memory leaks | Automatic cleanup of stale timers |
| Service restart loses timers | Accept data loss, timers recreated on next message |
| LangFlow down during warning | Use fallback message |
| Message rate limits | Single warning per timeout period |
