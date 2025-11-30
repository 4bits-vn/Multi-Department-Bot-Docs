# CB-016: Implementation Plan - Conversation Routing Refactor

> **Version**: 1.0
> **Created**: November 2024
> **Last Updated**: November 2024

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Route Lock State Machine](#route-lock-state-machine)
3. [Implementation Steps](#implementation-steps)
4. [Data Model Changes](#data-model-changes)
5. [Component Changes](#component-changes)
6. [API Integration](#api-integration)
7. [Testing Strategy](#testing-strategy)

---

## Architecture Overview

### Routing Flow

```
User Message
    │
    ▼
┌─────────────────────────────┐
│ ConversationRoutingService  │
│                             │
│  1. Check route lock state  │
│  2. Handle user commands    │
│  3. Detect topic switch     │
│  4. Classify if needed      │
└─────────────────────────────┘
    │
    ├─── LOCKED ────────────► Continue in locked route
    │                         (skip classification)
    │
    ├─── AWAITING_SWITCH ───► Show switch confirmation
    │
    └─── UNLOCKED ──────────► Call Classification API
                              │
                              ├── useLocalClassification=true
                              │   └── ClassificationClient
                              │       ├── useHttpClient=false → Direct call
                              │       └── useHttpClient=true  → HTTP call
                              │
                              └── useLocalClassification=false
                                  └── LangFlow Router (existing)
```

### Key Components

| Component | Purpose |
|-----------|---------|
| `ClassificationClient` | Abstraction for classification calls (direct or HTTP) |
| `ConversationRoutingService` | Route lock management and routing decisions |
| `UserService` (enhanced) | Persist route lock state to Azure Table |
| `LangFlowHandler` (modified) | Use ConversationRoutingService for routing |

---

## Route Lock State Machine

### State Definitions

```javascript
const RouteLockState = {
    UNLOCKED: 'UNLOCKED',           // Fresh routing needed
    LOCKED: 'LOCKED',               // Multi-turn flow active
    AWAITING_SWITCH: 'AWAITING_SWITCH', // Topic change detected
    PENDING_RELEASE: 'PENDING_RELEASE'  // Will release on next turn
};
```

### State Diagram

```
                         ┌─────────────────────────────────────────────────────┐
                         │                                                     │
                         │  "restart" / "cancel" / timeout                     │
                         │                                                     │
                         ▼                                                     │
    ┌──────────┐    Enter Flow    ┌────────┐    Task Done    ┌─────────────────┐
    │ UNLOCKED │ ───────────────► │ LOCKED │ ──────────────► │ PENDING_RELEASE │
    └──────────┘                  └────────┘                 └─────────────────┘
         ▲                             │                            │
         │                             │                            │
         │                             ▼                            │
         │                    ┌──────────────────┐                  │
         │                    │ AWAITING_SWITCH  │                  │
         │                    └──────────────────┘                  │
         │                        │         │                       │
         │           "continue"   │         │  "switch"             │
         │              ┌─────────┘         └─────────┐             │
         │              ▼                             ▼             │
         │         ┌────────┐                   ┌──────────┐        │
         │         │ LOCKED │                   │ UNLOCKED │        │
         │         └────────┘                   └──────────┘        │
         │                                           │              │
         └───────────────────────────────────────────┴──────────────┘
```

### State Transition Rules

```javascript
const STATE_TRANSITIONS = {
    UNLOCKED: {
        onEnterLockableRoute: 'LOCKED',
        onSingleTurnRoute: 'UNLOCKED'
    },
    LOCKED: {
        onContinueInFlow: 'LOCKED',
        onTaskCompleted: 'PENDING_RELEASE',
        onTopicChangeDetected: 'AWAITING_SWITCH',
        onUserCancel: 'UNLOCKED',
        onTimeout: 'UNLOCKED'
    },
    AWAITING_SWITCH: {
        onUserContinues: 'LOCKED',
        onUserSwitches: 'UNLOCKED',
        onTimeout: 'UNLOCKED'
    },
    PENDING_RELEASE: {
        onNoMoreHelp: 'UNLOCKED',
        onFollowUp: 'LOCKED',
        onNewQuestion: 'UNLOCKED'
    }
};
```

---

## Implementation Steps

### Step 1: Update Configuration

**File**: `src/config/index.js`

Add new configuration options:

```javascript
classification: {
    // ... existing options ...

    // CB-016: Routing Configuration
    routing: {
        // Use local classification instead of LangFlow Router
        useLocalClassification: process.env.USE_LOCAL_CLASSIFICATION !== 'false',
        // Use HTTP client (vs direct import) for classification
        useHttpClient: process.env.CLASSIFICATION_USE_HTTP === 'true',
        // HTTP client timeout
        httpTimeout: parseInt(process.env.CLASSIFICATION_HTTP_TIMEOUT) || 5000,
    },

    // Route Lock Configuration
    routeLock: {
        // Lock timeout in minutes (default: 30)
        timeoutMinutes: parseInt(process.env.ROUTE_LOCK_TIMEOUT_MINUTES) || 30,
        // Topic switch detection threshold
        topicSwitchThreshold: parseFloat(process.env.TOPIC_SWITCH_THRESHOLD) || 0.7,
        // Routes that should lock
        lockableRoutes: ['IT_TICKET_MGMT', 'IT_HELP_DESK'],
    },
}
```

### Step 2: Add Route Lock Fields to UserService

**File**: `src/Services/UserService.js`

Add new fields to user record:

```javascript
// Route Lock (CB-016)
route_lock_state: entity.route_lock_state || 'UNLOCKED',
locked_route: entity.locked_route || null,
locked_at: entity.locked_at || null,
lock_expires_at: entity.lock_expires_at || null,
route_context: entity.route_context || null,  // JSON string
```

Add new methods:

```javascript
async getRouteLock(email, channel)
async setRouteLock(email, channel, route, context)
async releaseRouteLock(email, channel, reason)
async updateRouteContext(email, channel, context)
```

### Step 3: Create ClassificationClient

**File**: `src/Services/ClassificationClient.js`

```javascript
class ClassificationClient {
    constructor() {
        this.useHttp = config.classification.routing.useHttpClient;
        this.timeout = config.classification.routing.httpTimeout;
        this.baseUrl = `http://localhost:${config.port}`;
    }

    async classifyToRoute(text, options = {}) {
        if (this.useHttp) {
            return this._httpClassify(text, options);
        }
        return classifyRoute(text, options);
    }

    async _httpClassify(text, options) {
        const response = await axios.post(
            `${this.baseUrl}/api/classification/route`,
            { text, ...options },
            {
                timeout: this.timeout,
                headers: { 'X-Internal-Request': 'true' }
            }
        );
        return response.data;
    }
}
```

### Step 4: Create ConversationRoutingService

**File**: `src/Services/ConversationRoutingService.js`

Core service with methods:

```javascript
class ConversationRoutingService {
    // Main routing entry point
    async routeMessage({ email, channel, message, conversationId })

    // Route lock management
    async getRouteLock(email, channel)
    async lockRoute(email, channel, route, context)
    async releaseRoute(email, channel, reason)
    async updateRouteContext(email, channel, contextUpdate)

    // User control
    async handleUserCommand(email, channel, message)

    // Topic switch detection
    async detectTopicSwitch(email, channel, message, currentRoute)
    async handleTopicSwitchResponse(email, channel, response)

    // Classification
    async classifyMessage(message, options)

    // Timeout handling
    isLockExpired(lockExpiresAt)
    async handleExpiredLock(email, channel)
}
```

### Step 5: Update LangFlowHandler

**File**: `src/Handlers/LangFlow/index.js`

Modify `sendToRouter` to use ConversationRoutingService:

```javascript
async sendToRouter(params) {
    const { message, sessionId, userEmail, channel } = params;

    // Use ConversationRoutingService for routing decision
    const routingResult = await this.routingService.routeMessage({
        email: userEmail,
        channel,
        message,
        sessionId
    });

    // Handle special cases (topic switch, user command)
    if (routingResult.handled) {
        return routingResult;
    }

    // Return classification result
    return {
        success: true,
        route: routingResult.route,
        message: routingResult.message,
        action: routingResult.isLocked ? 'GO' : routingResult.action,
        ...
    };
}
```

### Step 6: Update Flow Handlers

**Files**:
- `src/Handlers/LangFlow/flows/TicketingFlow.js`
- `src/Handlers/LangFlow/flows/ITHelpDeskFlow.js`

Add lock/release calls:

```javascript
// On flow entry (in execute method)
await routingService.lockRoute(userEmail, channel, 'IT_TICKET_MGMT', {
    currentStep: 'list_tickets'
});

// On task completion
await routingService.releaseRoute(userEmail, channel, 'task_completed');
```

---

## Data Model Changes

### BotUsers Table - New Fields

```
┌─────────────────────────────────────────────────────────────────┐
│                    BotUsers TABLE (Enhanced)                     │
├─────────────────────────────────────────────────────────────────┤
│  ... existing fields ...                                         │
├─────────────────────────────────────────────────────────────────┤
│  ROUTE LOCK (CB-016)                                             │
│  ────────────────────                                            │
│  • route_lock_state    - UNLOCKED | LOCKED | AWAITING_SWITCH |  │
│                          PENDING_RELEASE                         │
│  • locked_route        - Current locked route name               │
│  • locked_at           - When route was locked                   │
│  • lock_expires_at     - Lock expiration timestamp               │
│  • route_context       - JSON string with flow context           │
└─────────────────────────────────────────────────────────────────┘
```

### Route Context Schema

```javascript
{
    // Flow progress
    currentStep: 'select_action',
    completedSteps: ['list_tickets', 'select_ticket'],

    // Context data (varies by route)
    selectedTicketId: 'INC0012345',
    selectedAction: null,

    // Topic switch
    pendingSwitchTo: null,  // Route to switch to if user confirms
    pendingMessage: null,   // Original message that triggered switch

    // Tracking
    turnCount: 3,
    lastActivityAt: '2024-11-30T10:15:00Z'
}
```

---

## Component Changes

### New Files to Create

| File | Description |
|------|-------------|
| `src/Services/ClassificationClient.js` | Classification abstraction (HTTP/direct) |
| `src/Services/ConversationRoutingService.js` | Route lock and routing management |

### Files to Modify

| File | Changes |
|------|---------|
| `src/config/index.js` | Add routing and route lock configuration |
| `src/Services/UserService.js` | Add route lock fields and methods |
| `src/Handlers/LangFlow/index.js` | Use ConversationRoutingService |
| `src/Handlers/LangFlow/flows/TicketingFlow.js` | Add lock/release calls |
| `src/Handlers/LangFlow/flows/ITHelpDeskFlow.js` | Add lock/release calls |

---

## API Integration

### ClassificationClient Methods

```javascript
// Classify message to route
const result = await classificationClient.classifyToRoute(text, {
    useEnsemble: true  // Optional: use ensemble classification
});

// Response format
{
    success: true,
    route: 'IT_KB_SEARCH',
    score: 0.89,
    confidence: 'high',
    classifiedBy: 'ensemble'
}
```

### ConversationRoutingService Methods

```javascript
// Route a message
const result = await routingService.routeMessage({
    email: 'user@example.com',
    channel: 'msteams',
    message: 'check my ticket',
    conversationId: 'conv-123'
});

// Response format
{
    route: 'IT_TICKET_MGMT',
    isLocked: true,           // Was locked before or just locked
    wasClassified: true,      // Whether classification was called
    action: 'GO',
    message: null,            // Optional acknowledgment message
    handled: false,           // True if special case handled (command, switch)
    metadata: { ... }
}
```

---

## Testing Strategy

### Unit Tests

```javascript
describe('ConversationRoutingService', () => {
    describe('routeMessage', () => {
        it('should classify when UNLOCKED');
        it('should skip classification when LOCKED');
        it('should detect topic switch');
        it('should handle restart command');
    });

    describe('lockRoute', () => {
        it('should lock route with context');
        it('should set expiration time');
    });

    describe('releaseRoute', () => {
        it('should clear lock state');
        it('should preserve context for logging');
    });

    describe('timeout handling', () => {
        it('should detect expired lock');
        it('should release and notify on return');
    });
});
```

### Integration Tests

```javascript
describe('Conversation Flow', () => {
    it('should handle single-turn queries without locking');
    it('should lock on IT_TICKET_MGMT entry');
    it('should stay locked through multi-turn flow');
    it('should release on task completion');
    it('should handle topic switch confirmation');
    it('should handle restart mid-flow');
});
```

### Manual Testing Checklist

- [ ] Single-turn query (KB search) - no lock
- [ ] Multi-turn flow (ticket management) - locks
- [ ] Continue in locked flow - stays locked
- [ ] Complete task - releases lock
- [ ] Topic switch detection - shows prompt
- [ ] User continues after switch prompt
- [ ] User switches after switch prompt
- [ ] Restart command - releases immediately
- [ ] Timeout recovery - notifies user
- [ ] Server restart - lock persisted

---

## Environment Variables

```bash
# CB-016: Conversation Routing Configuration

# Use local classification instead of LangFlow Router (default: true)
USE_LOCAL_CLASSIFICATION=true

# Use HTTP client for classification (default: false = direct call)
CLASSIFICATION_USE_HTTP=false

# HTTP client timeout in ms (default: 5000)
CLASSIFICATION_HTTP_TIMEOUT=5000

# Route lock timeout in minutes (default: 30)
ROUTE_LOCK_TIMEOUT_MINUTES=30

# Topic switch detection threshold (default: 0.7)
TOPIC_SWITCH_THRESHOLD=0.7
```

---

## Implementation Order

1. **Phase 1: Configuration & Data Model**
   - [ ] Update `src/config/index.js`
   - [ ] Add route lock fields to `UserService.js`

2. **Phase 2: Classification Client**
   - [ ] Create `ClassificationClient.js`
   - [ ] Test HTTP and direct modes

3. **Phase 3: Routing Service**
   - [ ] Create `ConversationRoutingService.js`
   - [ ] Implement lock/unlock/classify logic
   - [ ] Add user command handling
   - [ ] Add topic switch detection

4. **Phase 4: Integration**
   - [ ] Update `LangFlowHandler`
   - [ ] Update `TicketingFlow`
   - [ ] Update `ITHelpDeskFlow`

5. **Phase 5: Testing**
   - [ ] Unit tests
   - [ ] Integration tests
   - [ ] Manual testing on Teams

---

## References

- [CB-014-Zero-Shot-Classification](../CB-014-Zero-Shot-Classification/)
- [CB-015-Intent-Classification-Quality](../CB-015-Intent-Classification-Quality/)
- [CB-006-Conversation-Lifecycle](../CB-006-Conversation-Lifecycle/)
- [IntentClassificationService](../../botframework/src/Services/IntentClassificationService.js)
