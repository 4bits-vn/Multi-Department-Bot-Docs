# CB-016: Implementation - Conversation Routing Refactor

## Summary

Implemented a new conversation routing system that uses the local Classification API (CB-014/CB-015) instead of LangFlow Router, with route locking for multi-turn conversation flows.

## Changes Implemented

### 1. Configuration Updates

**File**: `src/config/index.js`

Added new configuration sections:

```javascript
classification: {
    // CB-016: Routing Configuration
    routing: {
        useLocalClassification: true,   // Use local classification instead of LangFlow
        useHttpClient: false,           // Direct call vs HTTP
        httpTimeout: 5000,              // HTTP timeout
    },

    // CB-016: Route Lock Configuration
    routeLock: {
        timeoutMinutes: 30,             // Lock timeout
        topicSwitchThreshold: 0.7,      // Topic switch detection
        lockableRoutes: ['IT_TICKET_MGMT', 'IT_HELP_DESK'],
    },
}
```

### 2. UserService Enhancements

**File**: `src/Services/UserService.js`

Added new fields for route lock state:
- `route_lock_state` - UNLOCKED | LOCKED | AWAITING_SWITCH | PENDING_RELEASE
- `locked_route` - Current locked route name
- `locked_at` - When route was locked
- `lock_expires_at` - Lock expiration timestamp
- `route_context` - JSON string with flow context

Added new methods:
- `getRouteLock(email, channel)` - Get current lock state
- `setRouteLock(email, channel, route, context, timeout)` - Lock a route
- `releaseRouteLock(email, channel, reason)` - Release lock
- `updateRouteLockState(email, channel, newState, contextUpdate)` - Update state
- `updateRouteContext(email, channel, contextUpdate)` - Update context only

### 3. ClassificationClient Service

**File**: `src/Services/ClassificationClient.js` (NEW)

Abstraction layer for calling the Classification API with two modes:
- **Direct mode** (default): Imports and calls `classifyRoute()` directly
- **HTTP mode**: Makes HTTP request to `/api/classification/route`

Methods:
- `classifyToRoute(text, options)` - Classify message to route
- `classifyTicket(text)` - Classify ticket action/type
- `isAvailable()` - Check if service is available
- `getStatus()` - Get service status

### 4. ConversationRoutingService

**File**: `src/Services/ConversationRoutingService.js` (NEW)

Core routing service with route lock management:

**Route Lock States**:
```javascript
const RouteLockState = {
    UNLOCKED: 'UNLOCKED',           // Fresh routing needed
    LOCKED: 'LOCKED',               // Multi-turn flow active
    AWAITING_SWITCH: 'AWAITING_SWITCH', // Topic change detected
    PENDING_RELEASE: 'PENDING_RELEASE'  // Task done, will release
};
```

**Key Methods**:
- `routeMessage({ email, channel, message, sessionId })` - Main entry point
- `handleUserCommand(email, channel, message)` - Handle restart/cancel/status
- `lockRoute(email, channel, route, context)` - Lock for multi-turn flow
- `releaseRoute(email, channel, reason)` - Release lock
- `setPendingRelease(email, channel, context)` - Set to pending release
- `detectTopicSwitch(message, currentRoute)` - Detect topic change

**User Commands Supported**:
- `restart` / `start over` - Release lock, clear session
- `cancel` / `nevermind` - Release lock
- `status` - Show current context
- `switch` / `continue` - Response to topic switch prompt

### 5. LangFlowHandler Integration

**File**: `src/Handlers/LangFlow/index.js`

Modified `sendToRouter()` to use ConversationRoutingService:
- When `useLocalClassification=true` (default): Uses local Classification API via ConversationRoutingService
- When `useLocalClassification=false`: Falls back to LangFlow Router
- Handles special cases (user commands, topic switch)
- Preserves backward compatibility

Added exports:
- `conversationRoutingService`
- `RouteLockState`

### 6. TicketingFlow Updates

**File**: `src/Handlers/LangFlow/flows/TicketingFlow.js`

Integrated with ConversationRoutingService:
- Updates route context with current step during flow
- Detects completion actions (create, close, escalate, cancel)
- Sets to PENDING_RELEASE on completion
- Returns `action: 'COMPLETED'` when flow completes

### 7. ITHelpDeskFlow Updates

**File**: `src/Handlers/LangFlow/flows/ITHelpDeskFlow.js`

Integrated with ConversationRoutingService:
- Updates route context during handover
- Releases route lock on handover completion
- Handles handover errors gracefully

## Route Lock Rules

| Route | Locks? | Release Condition |
|-------|--------|-------------------|
| IT_TICKET_MGMT | Yes | After ticket action completed |
| IT_HELP_DESK | Yes | After handover to agent |
| IT_KB_SEARCH | No | Single-turn query |
| MED_KB_SEARCH | No | Single-turn query |
| MED_DRUG_SEARCH | No | Single-turn query |
| CHIT_CHAT | No | Single-turn response |
| OTHER | No | Single-turn response |

## Conversation Flow Example

```
1. User: "what's the weather today?"
   → UNLOCKED → Classify → CHIT_CHAT → Response
   → Stays UNLOCKED

2. User: "check my ticket"
   → UNLOCKED → Classify → IT_TICKET_MGMT → LOCK

3. User: "select ticket #2"
   → LOCKED → Skip classify → Continue in IT_TICKET_MGMT

4. User: "escalate it"
   → LOCKED → Process escalation → PENDING_RELEASE
   → Bot: "Ticket escalated. Anything else?"

5. User: "nothing else"
   → PENDING_RELEASE → Classify → CHIT_CHAT → Response
   → UNLOCKED

6. User: "what's the weather?" (during LOCKED)
   → LOCKED → Detect topic switch → AWAITING_SWITCH
   → Bot: "Continue with ticket or switch?"

7. User: "switch"
   → AWAITING_SWITCH → UNLOCKED → Classify → CHIT_CHAT
```

## Environment Variables

```bash
# Use local classification (default: true)
USE_LOCAL_CLASSIFICATION=true

# Use HTTP client for classification (default: false)
CLASSIFICATION_USE_HTTP=false

# HTTP timeout in ms (default: 5000)
CLASSIFICATION_HTTP_TIMEOUT=5000

# Route lock timeout in minutes (default: 30)
ROUTE_LOCK_TIMEOUT_MINUTES=30

# Topic switch detection threshold (default: 0.7)
TOPIC_SWITCH_THRESHOLD=0.7

# Lockable routes (default: IT_TICKET_MGMT,IT_HELP_DESK)
LOCKABLE_ROUTES=IT_TICKET_MGMT,IT_HELP_DESK
```

## Files Created/Modified

| File | Status | Description |
|------|--------|-------------|
| `docs/CB-016-Conversation-Routing/requirements.md` | NEW | Requirements document |
| `docs/CB-016-Conversation-Routing/implementation-plan.md` | NEW | Implementation plan |
| `docs/CB-016-Conversation-Routing/implementation.md` | NEW | This file |
| `src/config/index.js` | MODIFIED | Added routing and route lock config |
| `src/Services/UserService.js` | MODIFIED | Added route lock fields and methods |
| `src/Services/ClassificationClient.js` | NEW | Classification abstraction layer |
| `src/Services/ConversationRoutingService.js` | NEW | Core routing with lock management |
| `src/Handlers/LangFlow/index.js` | MODIFIED | Integrated ConversationRoutingService |
| `src/Handlers/LangFlow/flows/TicketingFlow.js` | MODIFIED | Added lock context management |
| `src/Handlers/LangFlow/flows/ITHelpDeskFlow.js` | MODIFIED | Added lock release on handover |

## Testing Checklist

### Manual Testing
- [ ] Single-turn queries route correctly without locking
- [ ] IT_TICKET_MGMT locks on entry
- [ ] Locked route skips re-classification
- [ ] Topic switch detected and prompts user
- [ ] User can continue or switch topics
- [ ] `restart` command releases lock
- [ ] `cancel` command releases lock
- [ ] `status` command shows current context
- [ ] Lock expires after timeout
- [ ] User notified on return after timeout
- [ ] IT_HELP_DESK releases on handover

### Integration Tests
- [ ] Classification API available and working
- [ ] Route lock persisted to Azure Table
- [ ] Lock survives server restart
- [ ] Fallback to LangFlow Router works

## Dependencies

- CB-014: Zero-Shot Classification API
- CB-015: Intent Classification Quality (Ensemble)
- CB-006: Conversation Lifecycle Service
- Azure Table Storage (UserService)

## Known Limitations

1. Route context is stored as JSON string in Azure Table (max 64KB)
2. Lock timeout is checked on message receipt, not proactively
3. Topic switch detection requires Classification API to be available

## Future Enhancements

- [ ] Proactive lock expiration notifications
- [ ] Soft lock support for IT_KB_SEARCH
- [ ] Route context compression for long flows
- [ ] Analytics for lock usage patterns
