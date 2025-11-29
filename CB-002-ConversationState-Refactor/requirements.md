# CB-002: Conversation State Service Refactor

## Overview

Simplify the `ConversationStateService` to follow Bot Framework best practices and better align with a LangFlow-delegated architecture.

## Problem Statement

The current `ConversationStateService` has several issues:

### 1. Overly Complex State Machine
- **10+ states** when only 3 are needed (IDLE, PROCESSING, ERROR)
- States like `IT_TICKET_MGMT`, `IT_KB_SEARCH`, `MED_KB_SEARCH` are **routing labels** from LangFlow, not conversation states
- Complex state transition validation provides little value since LangFlow handles routing logic

### 2. Incorrect State Scope
- Using `email + channel` as state key
- Should use `conversationId` (a user can have multiple conversations)
- Current design can't distinguish between personal chat and group chat with same user

### 3. Tight Coupling
- State stored inside user records (`next_action`, `next_action_updated_at` fields)
- Mixes user profile data with transient conversation state
- Makes it harder to scale and maintain

### 4. Missing Auto-Recovery
- No timeout handling for stuck states
- If LangFlow never calls back, state stays in `WAITING_FOR_LANGFLOW` forever
- Requires manual intervention to reset

### 5. Not Using Bot Framework Patterns
- Custom implementation instead of SDK patterns
- Not using `StatePropertyAccessor` pattern
- Not using built-in middleware for state management

## Solution

### New Design Principles

1. **Simple States** - Only track processing status, not routing decisions
2. **Conversation-Scoped** - Use `conversationId` as state key
3. **Auto-Recovery** - Timeout-based recovery from stuck states
4. **Separation of Concerns** - State separate from user data
5. **In-Memory + Redis** - Fast in-memory with optional Redis for distributed

### State Machine (Simplified)

```
┌─────────────────────────────────────────────────┐
│                                                 │
│    ┌──────┐   startProcessing   ┌────────────┐ │
│    │ IDLE │ ─────────────────► │ PROCESSING │ │
│    └──────┘                     └────────────┘ │
│        ▲                              │        │
│        │   completeProcessing         │        │
│        └──────────────────────────────┘        │
│        ▲                              │        │
│        │   auto-recovery              │ error  │
│        │   (30s)                      ▼        │
│    ┌───────┐                    ┌─────────┐   │
│    │       │ ◄───────────────── │  ERROR  │   │
│    └───────┘    auto-recovery   └─────────┘   │
│                   (2min)                       │
└─────────────────────────────────────────────────┘
```

### Comparison: Old vs New

| Aspect | Old (v1) | New (v2) |
|--------|----------|----------|
| States | 10+ (including routing labels) | 3 (IDLE, PROCESSING, ERROR) |
| Key | `email + channel` | `conversationId` |
| Storage | Azure Table (via UserService) | In-memory (Redis-ready) |
| Coupling | Tightly coupled to UserService | Independent |
| Auto-recovery | None | Yes (timeout-based) |
| Routing tracking | Yes (stores routing result) | No (LangFlow handles it) |

## Functional Requirements

### FR-1: Simple State Tracking
- Track only processing status: IDLE, PROCESSING, ERROR
- Do NOT track routing decisions (LangFlow's responsibility)

### FR-2: Conversation-Scoped State
- Use `conversationId` as primary key
- Support multiple conversations per user

### FR-3: Processing Lock
- Prevent duplicate message processing
- Return clear reason when message is blocked

### FR-4: Auto-Recovery
- Auto-recover from PROCESSING after 2 minutes
- Auto-recover from ERROR after 30 seconds
- Configurable timeouts

### FR-5: Session Management
- Store LangFlow session ID per conversation
- Support session ID updates

### FR-6: Cleanup
- Periodic cleanup of stale entries
- Configurable max age (default: 24 hours)

## Non-Functional Requirements

### NFR-1: Performance
- O(1) state lookup
- No database calls for state checks
- < 1ms for state operations

### NFR-2: Memory
- Efficient in-memory storage
- Automatic cleanup of old entries

### NFR-3: Observability
- Debug logging for state transitions
- Statistics endpoint for monitoring

### NFR-4: Scalability Ready
- Interface compatible with Redis
- No code changes needed for distributed deployment

## Acceptance Criteria

- [ ] State service uses `conversationId` as key (not email+channel)
- [ ] Only 3 states: IDLE, PROCESSING, ERROR
- [ ] Auto-recovery from stuck states works
- [ ] Debounce service removed (redundant with new design)
- [ ] All existing tests pass
- [ ] No breaking changes to webhook endpoints

## Dependencies

### Official Documentation
- [Bot Framework State Management](https://learn.microsoft.com/en-us/azure/bot-service/bot-builder-concept-state)
- [Conversation Reference](https://learn.microsoft.com/en-us/azure/bot-service/rest-api/bot-framework-rest-connector-api-reference)
- [Bot Builder JS - ConversationState](https://github.com/microsoft/botbuilder-js)

### Internal Dependencies
- Logger service
- UserService (for session ID lookup only)

## Migration Path

1. **Phase 1**: Create new `ConversationStateService.v2.js` alongside existing
2. **Phase 2**: Update `server.js` to use new service
3. **Phase 3**: Remove old service and `RequestDebounceService` (now redundant)
4. **Phase 4**: Update tests

## Timeline

- Implementation: 2-4 hours
- Testing: 1-2 hours
- Deployment: Can be deployed incrementally
