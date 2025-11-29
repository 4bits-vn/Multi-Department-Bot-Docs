# CB-006: Implementation Plan - Conversation Lifecycle Management

> **Version**: 1.0
> **Created**: November 2024
> **Last Updated**: November 2024

---

## Table of Contents

1. [Technical Approach](#technical-approach)
2. [Conversation Lifecycle States](#conversation-lifecycle-states)
3. [Implementation Steps](#implementation-steps)
4. [Data Model Changes](#data-model-changes)
5. [Component Changes](#component-changes)
6. [Integration with LangFlow](#integration-with-langflow)
7. [Error Handling Strategy](#error-handling-strategy)
8. [Testing Strategy](#testing-strategy)

---

## Technical Approach

### Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    CONVERSATION LIFECYCLE MANAGEMENT                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌──────────────────┐    ┌──────────────────┐    ┌──────────────────────┐   │
│  │  User Message    │───►│  Lifecycle Check │───►│  Route to Handler    │   │
│  └──────────────────┘    │                  │    └──────────────────────┘   │
│                          │  1. Check timeout│                               │
│                          │  2. Check restart│                               │
│                          │  3. Check state  │                               │
│                          └──────────────────┘                               │
│                                   │                                          │
│                                   ▼                                          │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                    CONVERSATION STATE MACHINE                          │  │
│  │                                                                        │  │
│  │    ┌────────┐     ┌─────────┐     ┌─────────────┐     ┌───────────┐   │  │
│  │    │  IDLE  │────►│ ACTIVE  │────►│ AWAITING    │────►│ COMPLETED │   │  │
│  │    └────────┘     └─────────┘     │ _INPUT      │     └───────────┘   │  │
│  │         ▲              │          └─────────────┘           │         │  │
│  │         │              │                │                   │         │  │
│  │         │              ▼                ▼                   │         │  │
│  │         │         ┌─────────┐     ┌───────────┐             │         │  │
│  │         └─────────│ ERROR   │◄────│ TIMED_OUT │◄────────────┘         │  │
│  │                   └─────────┘     └───────────┘                       │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Key Components

| Component | Purpose |
|-----------|---------|
| `ConversationLifecycleService` | New service to manage conversation lifecycle |
| `ConversationStateService` | Enhanced with lifecycle states |
| `UserService` | Enhanced with timeout tracking |
| `RestartIntentDetector` | Utility to detect restart commands |
| `EndConversationDetector` | Utility to detect end-of-conversation signals |

---

## Conversation Lifecycle States

### State Definitions

```javascript
const ConversationLifecycleState = {
    IDLE: 'IDLE',                    // No active conversation
    ACTIVE: 'ACTIVE',                // Conversation in progress
    AWAITING_INPUT: 'AWAITING_INPUT', // Bot waiting for user response
    PROCESSING: 'PROCESSING',         // Bot processing request
    COMPLETED: 'COMPLETED',           // Conversation ended naturally
    TIMED_OUT: 'TIMED_OUT',          // Expired due to inactivity
    ERROR: 'ERROR'                    // System error occurred
};
```

### State Transition Rules

```
IDLE:
  - User sends message → ACTIVE

ACTIVE:
  - LangFlow processing → PROCESSING
  - Error occurs → ERROR
  - Timeout reached → TIMED_OUT
  - User restarts → IDLE

PROCESSING:
  - Response received, action=GO → ACTIVE
  - Response received, action=COMPLETED → AWAITING_INPUT (ask if more help needed)
  - Error from LangFlow → ERROR
  - Timeout → ERROR

AWAITING_INPUT:
  - User says "yes/more help" → ACTIVE
  - User says "no/that's all" → COMPLETED
  - User asks new question → ACTIVE (new topic)
  - Timeout reached → TIMED_OUT

COMPLETED:
  - User sends new message → ACTIVE (new session)
  - Cleanup performed automatically

TIMED_OUT:
  - User sends message → IDLE (cleared) → ACTIVE

ERROR:
  - User restarts → IDLE
  - Auto-recovery after 30s → IDLE
```

### Timeout Configuration

```javascript
const LIFECYCLE_TIMEOUTS = {
    // Full conversation timeout (30 minutes)
    CONVERSATION_TIMEOUT_MS: 30 * 60 * 1000,

    // Waiting for user response (5 minutes)
    RESPONSE_TIMEOUT_MS: 5 * 60 * 1000,

    // LangFlow processing (2 minutes)
    PROCESSING_TIMEOUT_MS: 2 * 60 * 1000,

    // Error recovery (30 seconds)
    ERROR_RECOVERY_MS: 30 * 1000
};
```

---

## Implementation Steps

### Step 1: Create ConversationLifecycleService

**File**: `src/Services/ConversationLifecycleService.js`

Core lifecycle management service with:

- State machine implementation
- Timeout detection
- Restart intent detection
- End conversation detection
- Integration with Azure Table Storage

### Step 2: Create Intent Detectors

**File**: `src/utils/intentDetectors.js`

Utilities for detecting:

```javascript
// Bot commands (from Teams command menu)
const BOT_COMMANDS = {
    RESTART: 'restart',
    HELP: 'help'
};

function detectBotCommand(text) {
    // Returns { command, isCommand, response? } or null
}

// Restart patterns (natural language)
const RESTART_PATTERNS = [
    /\b(restart|start\s*over|new\s*conversation)\b/i,
    /\b(cancel|nevermind|never\s*mind|forget\s*it)\b/i,
    /\b(clear|reset|begin\s*again)\b/i
];

// End conversation patterns
const END_CONVERSATION_PATTERNS = [
    /\b(no\s*(thanks?|thank\s*you)?|that'?s?\s*all|goodbye|bye)\b/i,
    /\b(nothing\s*(else)?|i'?m?\s*good|all\s*set)\b/i
];

// More help patterns (continues conversation)
const MORE_HELP_PATTERNS = [
    /\b(yes|yeah|yep|sure|please|help)\b/i,
    /\b(another\s*question|more\s*help)\b/i
];
```

### Step 2.1: Update Teams Manifest

**File**: `teams-packages/manifest.json`

Add command menu for restart and help:

```json
"commandLists": [{
    "scopes": ["personal", "team", "groupChat"],
    "commands": [
        { "title": "restart", "description": "Clear conversation and start fresh" },
        { "title": "help", "description": "Show available commands" }
    ]
}]
```

### Step 3: Enhance ConversationStateService

**File**: `src/Services/ConversationStateService.js`

Add lifecycle tracking:

```javascript
// New fields in state data
{
    lifecycleState: 'ACTIVE',
    lastInteractionAt: Date.now(),
    awaitingInputSince: null,
    questionAsked: null  // Track what we're waiting for
}
```

### Step 4: Enhance UserService

**File**: `src/Services/UserService.js`

Add new methods:

```javascript
// Get conversation lifecycle state
async getConversationLifecycle(email, channel)

// Update lifecycle state with timestamp
async updateLifecycleState(email, channel, state, metadata)

// Clear conversation completely (for restart/timeout)
async clearConversation(email, channel, reason)
```

### Step 5: Update Bot Controller

**File**: `src/controllers/bot.controller.js`

Add lifecycle checks before processing:

```javascript
async function handleTeamsMessage(activity) {
    // 1. Get user info
    // 2. Check lifecycle state FIRST
    const lifecycleCheck = await lifecycleService.checkAndHandle({
        email: userEmail,
        channel,
        messageText,
        conversationId,
        serviceUrl
    });

    if (lifecycleCheck.handled) {
        // Lifecycle action was taken (restart, timeout recovery, etc.)
        return;
    }

    // 3. Continue with normal processing...
}
```

### Step 6: Update LangFlowHandler

**File**: `src/Handlers/LangFlowHandler.js`

Add error detection and handling:

```javascript
// Detect system errors in LangFlow responses
const LANGFLOW_ERROR_PATTERNS = [
    /Agent stopped due to max iterations/i,
    /maximum recursion depth/i,
    /timeout exceeded/i
];

// Enhanced router response handling
parseRouterResponse(response) {
    // Check for system errors
    if (this.isSystemError(response)) {
        return {
            success: false,
            isSystemError: true,
            errorType: 'MAX_ITERATIONS',
            ...
        };
    }
}
```

### Step 7: Update handleRouterResponse

**File**: `src/controllers/bot.controller.js`

Handle end-of-conversation flow:

```javascript
// When action is COMPLETED and it's a resolution
if (action === ROUTER_ACTIONS.COMPLETED) {
    // Check if this is a resolution message
    if (this.isResolutionResponse(message)) {
        // Update lifecycle state to AWAITING_INPUT
        await lifecycleService.setAwaitingInput(userEmail, channel, {
            question: 'need_more_help',
            askedAt: new Date().toISOString()
        });

        // Send message with "need more help?" question
        await msTeamsService.sendProactiveMessage({
            serviceUrl,
            conversationId,
            message: message + "\n\nIs there anything else I can help you with?"
        });
        return;
    }
}
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
│  CONVERSATION LIFECYCLE (NEW)                                    │
│  ────────────────────────                                        │
│  • lifecycle_state       - Current lifecycle state               │
│  • lifecycle_updated_at  - Last state change timestamp           │
│  • awaiting_input_since  - When bot started waiting for response │
│  • awaiting_input_type   - What type of input (need_more_help)   │
│  • conversation_started_at - When current conversation began     │
│  • conversation_ended_at - When conversation was completed       │
│  • last_error           - Last error message (if any)            │
│  • last_error_at        - When last error occurred               │
└─────────────────────────────────────────────────────────────────┘
```

### Migration

No migration needed - new fields added via upsert with Merge mode.

---

## Component Changes

### New Files to Create

| File | Description |
|------|-------------|
| `src/Services/ConversationLifecycleService.js` | Main lifecycle management |
| `src/utils/intentDetectors.js` | Restart/end intent detection |
| `src/utils/lifecycleMessages.js` | User-facing messages |

### Files to Modify

| File | Changes |
|------|---------|
| `src/controllers/bot.controller.js` | Add lifecycle checks |
| `src/Services/ConversationStateService.js` | Add lifecycle states |
| `src/Services/UserService.js` | Add lifecycle persistence |
| `src/Handlers/LangFlowHandler.js` | Add error detection |
| `src/config/index.js` | Add timeout configuration |

---

## Integration with LangFlow

### Router Response Handling

The ROUTER flow should return:

```json
{
    "route": "...",
    "message": "Here's the solution...",
    "action": "COMPLETED",
    "summary": "...",
    "resolution_type": "ANSWERED"  // NEW: indicates resolution type
}
```

Resolution types:
- `ANSWERED` - Question answered, should ask if more help needed
- `ESCALATED` - Transferred to human agent
- `CREATED` - Ticket/item created
- `NO_ANSWER` - Could not help

### Error Response Detection

LangFlow may return errors in various formats:

```javascript
// 1. In response text
"Agent stopped due to max iterations"

// 2. In structured response
{
    "error": true,
    "error_type": "MAX_ITERATIONS",
    "message": "I apologize, but I'm having trouble..."
}

// 3. As HTTP error
// Status 500 with error body
```

---

## Error Handling Strategy

### Error Categories

| Category | Examples | Handling |
|----------|----------|----------|
| **Recoverable** | Network timeout, 503 | Retry with backoff |
| **User Error** | Invalid input | Ask for clarification |
| **System Error** | Max iterations, crash | Reset session, apologize |
| **Config Error** | Missing flow ID | Log, use fallback |

### Error Messages

```javascript
const LIFECYCLE_ERROR_MESSAGES = {
    MAX_ITERATIONS: "I apologize, but I've hit a processing limit. Let me start fresh - could you please rephrase your question?",

    TIMEOUT: "Your session has expired due to inactivity. I've cleared our previous conversation. How can I help you today?",

    RESTART_CONFIRMED: "I've cleared our conversation history. How can I help you with a fresh start?",

    GENERIC_ERROR: "I'm sorry, I encountered an unexpected issue. Please try again or start a new conversation by typing 'restart'."
};
```

---

## Testing Strategy

### Unit Tests

```javascript
describe('ConversationLifecycleService', () => {
    describe('checkTimeout', () => {
        it('should detect conversation timeout after 30 minutes');
        it('should detect response timeout after 5 minutes');
        it('should not timeout during active processing');
    });

    describe('detectRestartIntent', () => {
        it('should detect "restart" command');
        it('should detect "start over" command');
        it('should not false-positive on normal messages');
    });

    describe('detectEndConversation', () => {
        it('should detect "no thanks" as end');
        it('should detect "yes" as continue');
        it('should handle "no, actually..." as continue');
    });
});
```

### Integration Tests

```javascript
describe('Conversation Lifecycle Flow', () => {
    it('should complete conversation naturally');
    it('should handle user restart mid-conversation');
    it('should recover from timeout');
    it('should handle LangFlow errors gracefully');
});
```

### Manual Testing Checklist

- [ ] Test "restart" command at various stages
- [ ] Test timeout after 30+ minutes inactivity
- [ ] Test "no thanks" after resolution
- [ ] Test continued conversation after resolution
- [ ] Test error recovery from max iterations
- [ ] Test rapid messages during processing

---

## Implementation Order

1. **Phase 1: Core Infrastructure**
   - [ ] Create `ConversationLifecycleService`
   - [ ] Create `intentDetectors.js`
   - [ ] Create `lifecycleMessages.js`
   - [ ] Add new fields to UserService

2. **Phase 2: Timeout Handling**
   - [ ] Implement conversation timeout check
   - [ ] Implement response timeout check
   - [ ] Add timeout recovery in bot controller

3. **Phase 3: Restart Command**
   - [ ] Implement restart detection
   - [ ] Add restart handler in bot controller
   - [ ] Clear session on restart

4. **Phase 4: End Conversation**
   - [ ] Implement "need more help?" flow
   - [ ] Detect end-of-conversation responses
   - [ ] Handle conversation completion

5. **Phase 5: Error Handling**
   - [ ] Detect LangFlow system errors
   - [ ] Implement error recovery
   - [ ] Add user-friendly error messages

6. **Phase 6: Testing & Polish**
   - [ ] Unit tests
   - [ ] Integration tests
   - [ ] Manual testing on Teams

---

## Environment Variables

```bash
# Conversation Lifecycle Configuration
CONVERSATION_TIMEOUT_MINUTES=30
RESPONSE_TIMEOUT_MINUTES=5
PROCESSING_TIMEOUT_MINUTES=2
ERROR_RECOVERY_SECONDS=30

# Feature Flags
ENABLE_AUTO_ASK_MORE_HELP=true
ENABLE_TIMEOUT_WARNING=false  # Future: send warning before timeout
```

---

## References

- [Bot Framework State Management](https://learn.microsoft.com/en-us/azure/bot-service/bot-builder-concept-state)
- [LangFlow API Reference](https://docs.langflow.org/api-reference-api-examples)
- [Azure Table Storage](https://learn.microsoft.com/en-us/azure/cosmos-db/table/how-to-use-nodejs)
- [ARCHITECTURE.md](../ARCHITECTURE.md)
