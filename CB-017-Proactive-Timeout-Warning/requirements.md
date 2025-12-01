# CB-017: Proactive Timeout Warning & Expiration

## Overview

Implement a proactive timeout system that:
1. **Warning**: Sends an automated warning message before conversation timeout
2. **Expiration**: Sends an expiration message AND closes the conversation when timeout occurs
3. **Clean Return**: User returns to a fresh conversation without any reactive timeout messages

## Problem Statement

Previously, users were only informed about conversation timeout when they sent a new message after the session had already expired. This led to:
- Confusing "session expired" messages when users returned
- Loss of context without any warning
- Poor user experience with redundant messages

## Requirements

### Functional Requirements

1. **Proactive Warning Message**
   - Send an automated warning message X minutes before conversation timeout (configurable)
   - Warning should inform users that the conversation will timeout soon
   - Message should mention that messages after timeout will start a new conversation
   - Use LangFlow OTHER flow for dynamic message generation

2. **Proactive Expiration Message**
   - When timeout actually occurs, send an expiration message proactively
   - Close the conversation (clear session, set lifecycle_state to TIMED_OUT)
   - User should NOT receive any reactive "session expired" message when they return

3. **Timer Management**
   - Track last activity time for each active conversation
   - Schedule warning timer based on: `conversationTimeout - warningTime`
   - Schedule expiration timer after warning is sent
   - Reset/reschedule timers on each user interaction
   - Clear scheduled timers when conversation ends or is restarted

4. **Clean User Return**
   - When user returns after proactive expiration, start fresh conversation
   - NO "session expired" or "fresh conversation" message - just respond normally
   - Session already cleared proactively, so reactive cleanup is minimal

5. **Channel Support**
   - Support MS Teams channel (proactive messaging)
   - Maintain conversation reference (serviceUrl, conversationId) for proactive messaging

### Non-Functional Requirements

1. **Performance**
   - Efficient in-memory timer management
   - Minimal memory footprint per conversation
   - Automatic cleanup of stale timers

2. **Configurability**
   - Enable/disable feature via environment variable
   - Configurable warning time before timeout
   - Configurable conversation timeout (existing)

3. **Reliability**
   - Rehydrate timers from storage on server restart
   - Handle proactive message failures gracefully
   - Don't send warnings/expirations if user becomes active
   - Prevent duplicate messages

## Configuration

### Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `ENABLE_TIMEOUT_WARNING` | `false` | Enable/disable timeout warning & expiration feature |
| `TIMEOUT_WARNING_MINUTES` | `5` | Minutes before timeout to send warning |
| `CONVERSATION_TIMEOUT_MINUTES` | `30` | Total conversation timeout |

### Example Configuration

```bash
# Enable timeout warning & expiration feature
ENABLE_TIMEOUT_WARNING=true

# Send warning 5 minutes before 30-minute timeout
TIMEOUT_WARNING_MINUTES=5
CONVERSATION_TIMEOUT_MINUTES=30

# Timeline:
# - Warning sent at 25 minutes of inactivity
# - Expiration sent at 30 minutes of inactivity
# - Conversation closed at 30 minutes
```

## User Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                    USER SENDS MESSAGE                           │
│                           ↓                                     │
│              Reset timeout warning timer                        │
│              Schedule warning at (timeout - warning) minutes    │
│                           ↓                                     │
│              ... user inactive for X minutes ...                │
│                           ↓                                     │
│              ┌─────────────────────────────────┐                │
│              │  WARNING TIMER FIRES            │                │
│              │  (at timeout - warning time)    │                │
│              │                                 │                │
│              │  1. Generate warning via OTHER  │                │
│              │  2. Send proactive message      │                │
│              │  3. Schedule expiration timer   │                │
│              └─────────────────────────────────┘                │
│                           ↓                                     │
│              ... additional Y minutes pass ...                  │
│                           ↓                                     │
│              ┌─────────────────────────────────┐                │
│              │  EXPIRATION TIMER FIRES         │                │
│              │  (at timeout time)              │                │
│              │                                 │                │
│              │  1. Generate expiration message │                │
│              │  2. Send proactive message      │                │
│              │  3. Clear session               │                │
│              │  4. Set lifecycle_state=TIMED_OUT│               │
│              │  5. Conversation is CLOSED      │                │
│              └─────────────────────────────────┘                │
│                           ↓                                     │
│              ... some time later ...                            │
│                           ↓                                     │
│              ┌─────────────────────────────────┐                │
│              │  USER RETURNS WITH MESSAGE      │                │
│              │                                 │                │
│              │  - NO "session expired" message │                │
│              │  - Start fresh conversation     │                │
│              │  - Respond normally to message  │                │
│              └─────────────────────────────────┘                │
└─────────────────────────────────────────────────────────────────┘
```

## Acceptance Criteria

1. ✅ Warning is sent proactively before conversation expires
2. ✅ Expiration message is sent when conversation actually times out
3. ✅ Conversation is closed (session cleared, lifecycle_state=TIMED_OUT) at expiration
4. ✅ User returns to fresh conversation WITHOUT any reactive timeout message
5. ✅ Warning/expiration times are configurable via environment variables
6. ✅ Feature can be enabled/disabled via environment variable
7. ✅ Messages are generated by LangFlow OTHER flow
8. ✅ Timers reset when user sends a new message
9. ✅ No warning/expiration sent if user becomes active
10. ✅ Timers rehydrated from storage on server restart
11. ✅ MS Teams proactive messaging works correctly

## Dependencies

- `MSTeamsService` - For sending proactive messages
- `UserService` - For session management and conversation queries
- `OtherFlow` (LangFlow) - For generating warning/expiration messages
- `config/index.js` - For configuration
- `ConversationLifecycleService` - For timeout settings (reactive handler now returns no message)

## Related Documentation

- [CB-006-Conversation-Lifecycle](../CB-006-Conversation-Lifecycle/) - Original lifecycle implementation
- [flows/MISCELLANEOUS/README.md](../../flows/MISCELLANEOUS/README.md) - OTHER flow documentation
