# CB-006: Conversation Lifecycle Management

> **Ticket**: CB-006-Conversation-Lifecycle
> **Type**: Feature Enhancement
> **Priority**: High
> **Status**: Planning

---

## Overview

Implement comprehensive conversation lifecycle management for the DXC ChatBot Platform. This feature handles conversation endings, timeouts, restarts, and error recovery to provide a seamless user experience.

---

## Problem Statement

Currently, the system lacks proper handling for:

1. **Conversation endings** - No clear mechanism to detect when a user has finished their conversation
2. **Session timeouts** - Users who leave mid-conversation have stale sessions
3. **Conversation restarts** - No way for users to restart a conversation without manual intervention
4. **System errors** - LangFlow errors (e.g., "Agent stopped due to max iterations") are not gracefully handled

---

## Use Cases

### UC-1: Natural Conversation End
**Scenario**: User completes their request successfully
- User starts a chat with bot
- Bot provides a helpful solution
- Bot asks "Is there anything else I can help you with?"
- User confirms "No, that's all" or similar
- **Expected**: Conversation marked as COMPLETED, session cleaned up, ready for new topic

### UC-2: Conversation Timeout
**Scenario**: User abandons conversation
- User starts a chat with bot
- User leaves for an extended period (e.g., 30 minutes)
- User returns and sends a new message
- **Expected**: Old conversation context is cleared, fresh session starts

### UC-3: User-Initiated Restart
**Scenario**: User wants to start over mid-conversation
- User is in the middle of a conversation (e.g., ticket creation)
- User says "restart", "start over", "new conversation", etc.
- **Expected**: Current conversation cleared, fresh session starts with acknowledgment

### UC-4: LangFlow System Error
**Scenario**: LangFlow returns a system error
- User sends a message
- LangFlow returns error: "Agent stopped due to max iterations"
- **Expected**: User receives friendly error message, session is preserved or reset based on error type

### UC-5: Inactivity During Processing
**Scenario**: User is inactive while bot is asking follow-up questions
- Bot asks for more information (e.g., "What is your priority level?")
- User doesn't respond for extended period
- **Expected**: Conversation times out with friendly message on next interaction

---

## Functional Requirements

### FR-1: Conversation State Machine

Implement a proper conversation state machine with the following states:

```
STATES:
├── IDLE          - No active conversation
├── ACTIVE        - Conversation in progress
├── AWAITING_INPUT - Bot is waiting for user response
├── PROCESSING    - Bot is processing a request
├── COMPLETED     - Conversation ended naturally
├── TIMED_OUT     - Conversation expired due to inactivity
└── ERROR         - System error occurred
```

### FR-2: Conversation End Detection

The system MUST detect conversation completion from:

1. **Router Response**: When router returns `action: COMPLETED` with follow-up questions
2. **User Confirmation**: When user indicates no more help needed
3. **Explicit End Phrases**: Patterns like "no thanks", "that's all", "goodbye"

### FR-3: Session Timeout

The system MUST implement configurable timeouts:

| Timeout Type | Default | Description |
|-------------|---------|-------------|
| Conversation Timeout | 30 minutes | Full conversation inactivity |
| Response Timeout | 5 minutes | User response to bot question |
| Processing Timeout | 2 minutes | LangFlow processing time |

### FR-4: Restart Commands

The system MUST support restart via:

**1. Teams Command Menu**:
- Bot command: `restart` - accessible from Teams command menu
- Bot command: `help` - shows available commands

**2. Natural Language Patterns**:
- "restart", "start over", "new conversation"
- "cancel", "nevermind", "forget it"
- "clear", "reset"

**Teams Manifest Configuration**:
```json
"commandLists": [{
    "scopes": ["personal", "team", "groupChat"],
    "commands": [
        { "title": "restart", "description": "Clear conversation and start fresh" },
        { "title": "help", "description": "Show available commands" }
    ]
}]
```

### FR-5: Error Handling

The system MUST handle LangFlow errors gracefully:

| Error Type | Handling |
|-----------|----------|
| Max iterations reached | Reset session, offer restart |
| API timeout | Retry once, then error message |
| Invalid response format | Log error, fallback message |
| Network error | Retry with backoff |

### FR-6: User Feedback Messages

The system MUST provide clear feedback for lifecycle events:

| Event | Message Template |
|-------|-----------------|
| Conversation End | "Is there anything else I can help you with?" |
| Timeout Warning | (Optional) "Are you still there?" |
| Session Expired | "Your previous session has expired. Let me start fresh." |
| Restart Confirmed | "I've cleared our conversation. How can I help you today?" |
| Error Recovery | "I apologize for the inconvenience. Let's start over." |

---

## Non-Functional Requirements

### NFR-1: Performance
- State checks must complete within 50ms
- Session operations must complete within 200ms
- No impact on message processing latency

### NFR-2: Reliability
- State must be persisted to Azure Table (not just in-memory)
- Recovery from server restarts without losing conversation context

### NFR-3: Scalability
- Support for 10,000+ concurrent conversations
- Efficient cleanup of stale sessions

### NFR-4: Observability
- Log all state transitions
- Track timeout occurrences
- Monitor error rates by type

---

## Acceptance Criteria

### AC-1: Natural End
- [ ] Router returns COMPLETED with "need more help?" question
- [ ] User says "no" → Session cleared, conversation ends
- [ ] User says "yes" → Conversation continues
- [ ] User asks new question → New topic started

### AC-2: Timeout
- [ ] Conversation inactive for 30+ minutes → Auto-cleared on next message
- [ ] User notified that session was reset
- [ ] Previous context not carried over

### AC-3: Restart
- [ ] User says "restart" → Session cleared immediately
- [ ] Confirmation message sent
- [ ] New session created for next message

### AC-4: Error Recovery
- [ ] LangFlow max iterations → Friendly error + offer restart
- [ ] Session cleared after error
- [ ] Errors logged with context

---

## Dependencies

- LangFlow Router must support returning `action: COMPLETED` correctly
- Azure Table Storage for persistent state
- Current ConversationStateService (to be enhanced)
- Current UserService (to be enhanced)

---

## References

- [Bot Framework State Management](https://learn.microsoft.com/en-us/azure/bot-service/bot-builder-concept-state)
- [LangFlow API Reference](https://docs.langflow.org/api-reference-api-examples)
- [ARCHITECTURE.md](../ARCHITECTURE.md)
- [CB-003-Consolidated-Storage](../CB-003-Consolidated-Storage/)
