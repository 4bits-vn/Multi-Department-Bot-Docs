# CB-002: Implementation Plan

## Overview

Refactor `ConversationStateService` to follow Bot Framework best practices with simplified state management.

## Technical Approach

### Architecture Change

```
BEFORE (Complex):
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  server.js ──► ConversationStateService ──► UserService ──► Azure Table
│      │                                                          │
│      └──► RequestDebounceService (redundant)                    │
│                                                                 │
│  States: IDLE, WAITING_FOR_LANGFLOW, IT_TICKET_MGMT,           │
│          IT_KB_SEARCH, IT_HELP_DESK, MED_KB_SEARCH,            │
│          MED_DRUG_SEARCH, CHIT_CHAT, OTHER, COMPLETED, ERROR   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

AFTER (Simple):
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  server.js ──► ConversationStateService ──► In-Memory Store     │
│                                              (Redis-ready)      │
│                                                                 │
│  States: IDLE, PROCESSING, ERROR                               │
│                                                                 │
│  Features:                                                      │
│  - Auto-recovery from stuck states                             │
│  - Conversation-scoped (not user-scoped)                       │
│  - Built-in debounce (no separate service)                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Key Changes

| Component | Change |
|-----------|--------|
| `ConversationStateService` | Complete rewrite with simplified states |
| `RequestDebounceService` | Remove (functionality merged into state service) |
| `server.js` | Update to use new API |
| `UserService` | No changes (still handles user data, not state) |

## Implementation Steps

### Step 1: Create New ConversationStateService (✅ Done)

File: `src/Services/ConversationStateService.v2.js`

Features:
- Simple 3-state machine: IDLE → PROCESSING → IDLE/ERROR
- Conversation-scoped (`conversationId` as key)
- Auto-recovery with configurable timeouts
- Built-in statistics
- Periodic cleanup of stale entries

### Step 2: Update server.js

Replace usage of old services:

```javascript
// BEFORE
const { createConversationStateService, STATES } = require('./Services/ConversationStateService');
const { requestDebounceService } = require('./Services/RequestDebounceService');
const conversationStateService = createConversationStateService(userService);

// AFTER
const { conversationStateService, ProcessingState } = require('./Services/ConversationStateService.v2');
// RequestDebounceService is no longer needed
```

Update `handleTeamsMessage`:

```javascript
// BEFORE
const debounceCheck = requestDebounceService.canProcessRequest(userEmail, channel);
if (!debounceCheck.canProcess) { ... }
requestDebounceService.startProcessing(userEmail, channel, activity.text);
await conversationStateService.transitionTo(userEmail, channel, STATES.WAITING_FOR_LANGFLOW);

// AFTER
const conversationId = activity.conversation?.id;
const processCheck = conversationStateService.canProcess(conversationId);
if (!processCheck.canProcess) { ... }
conversationStateService.startProcessing({
    conversationId,
    sessionId,
    userEmail,
    channel
});
```

### Step 3: Update Webhook Endpoints

Update webhook handlers to use `conversationId`:

```javascript
// BEFORE
await conversationStateService.transitionTo(userEmail, channel, STATES.COMPLETED);
requestDebounceService.endProcessing(userEmail, channel);

// AFTER
await conversationStateService.completeProcessing(conversationId);
```

### Step 4: Remove Redundant Code

- Delete `RequestDebounceService.js`
- Rename `ConversationStateService.v2.js` to `ConversationStateService.js`
- Update all imports

### Step 5: Add Redis Support (Optional, Future)

For distributed deployments, replace `ConversationStateStore` with Redis:

```javascript
// Future enhancement
class RedisConversationStateStore {
    constructor(redisClient) {
        this.redis = redisClient;
        this.prefix = 'conv_state:';
    }

    async get(conversationId) {
        const data = await this.redis.get(this.prefix + conversationId);
        return data ? JSON.parse(data) : null;
    }

    async set(conversationId, data) {
        await this.redis.setex(
            this.prefix + conversationId,
            86400, // 24 hour TTL
            JSON.stringify(data)
        );
    }
}
```

## API Changes

### Old API (v1)

```javascript
// Get state
const state = await conversationStateService.getCurrentState(email, channel);

// Transition
await conversationStateService.transitionTo(email, channel, STATES.WAITING_FOR_LANGFLOW);

// Complete
await conversationStateService.markCompleted(email, channel);

// Check processing
const isProcessing = await conversationStateService.isProcessing(email, channel);
```

### New API (v2)

```javascript
// Check if can process
const { canProcess, reason } = conversationStateService.canProcess(conversationId);

// Start processing
conversationStateService.startProcessing({
    conversationId,
    sessionId,
    userEmail,
    channel
});

// Complete processing
conversationStateService.completeProcessing(conversationId);

// Mark error
conversationStateService.markError(conversationId, 'Error message');

// Check processing
const isProcessing = conversationStateService.isProcessing(conversationId);

// Get session ID
const sessionId = conversationStateService.getSessionId(conversationId);
```

## Testing Strategy

### Unit Tests

```javascript
describe('ConversationStateService v2', () => {
    it('should allow processing when IDLE', () => {
        const result = service.canProcess('conv-123');
        expect(result.canProcess).toBe(true);
    });

    it('should block processing when already PROCESSING', () => {
        service.startProcessing({ conversationId: 'conv-123', sessionId: 'sess-1' });
        const result = service.canProcess('conv-123');
        expect(result.canProcess).toBe(false);
    });

    it('should auto-recover from stuck PROCESSING after timeout', async () => {
        service.startProcessing({ conversationId: 'conv-123', sessionId: 'sess-1' });
        // Simulate timeout (mock Date.now())
        await advanceTime(2 * 60 * 1000 + 1);
        const result = service.canProcess('conv-123');
        expect(result.canProcess).toBe(true);
    });
});
```

### Integration Tests

- Test full message flow with new state service
- Test webhook callbacks update state correctly
- Test auto-recovery in real scenarios

## Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| State lost on restart | Users might see duplicate responses | Accept for now; add Redis for production |
| Race condition | Concurrent messages might slip through | In-memory Map is synchronous; safe in single-node |
| Breaking change | Existing flows might break | Gradual rollout; keep old service as fallback |

## Rollback Plan

1. Keep old `ConversationStateService.js` as backup
2. Feature flag to switch between old/new
3. If issues, revert imports in `server.js`

## Timeline

| Task | Time |
|------|------|
| Create new service | ✅ Done |
| Update server.js | 1 hour |
| Testing | 1 hour |
| Documentation | 30 min |
| Code review | 30 min |
| **Total** | ~3 hours |

## Checklist

- [x] New `ConversationStateService.js` created (was v2, now promoted)
- [x] Update `server.js` to use new service
- [x] Remove `RequestDebounceService` dependency
- [x] Update webhook endpoints
- [x] Update admin endpoints
- [x] Update shutdown handlers
- [ ] Add unit tests
- [ ] Test manually on Teams
- [x] Update documentation
- [x] Old service files backed up (*.old.js)
