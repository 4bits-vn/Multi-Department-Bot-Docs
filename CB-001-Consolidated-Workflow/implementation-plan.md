# CB-001: Implementation Plan - Consolidated Bot Framework Workflow

## Overview

This document outlines the implementation plan for the consolidated workflow in the Bot Framework server.

## Technical Approach

### Architecture Changes

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         Bot Framework Server                             │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                          server.js                                │   │
│  │  POST /api/messages ──────────────────────────────────────────┐  │   │
│  │  POST /api/webhook/langflow ───────────────────────────────┐  │  │   │
│  │  POST /api/webhook/langflow/routing ────────────────────┐  │  │  │   │
│  └─────────────────────────────────────────────────────────┼──┼──┼──┘   │
│                                                            │  │  │      │
│  ┌─────────────────────────────────────────────────────────┼──┼──┼──┐   │
│  │                         Handlers                         │  │  │  │   │
│  │  ┌──────────────────────────────────────────────────────┼──┼──┼┐ │   │
│  │  │ LangFlowHandler                                      │  │  ││ │   │
│  │  │  - sendToRouter()     ───────────────────────────────┘  │  ││ │   │
│  │  │  - sendToExceptionFlow() ───────────────────────────────┘  ││ │   │
│  │  │  - handleRoutingResult()  ─────────────────────────────────┘│ │   │
│  │  └─────────────────────────────────────────────────────────────┘ │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │                         Services                                   │   │
│  │  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐      │   │
│  │  │ UserService    │  │ MSTeamsService │  │ ServiceNow     │      │   │
│  │  │ (+ session)    │  │                │  │ Service        │      │   │
│  │  └────────────────┘  └────────────────┘  └────────────────┘      │   │
│  │  ┌────────────────┐  ┌────────────────┐                          │   │
│  │  │ ConversationState│ │ RequestDebounce│                          │   │
│  │  │ Service (NEW)   │  │ Service        │                          │   │
│  │  └────────────────┘  └────────────────┘                          │   │
│  └──────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────┘
```

### Components to Create/Modify

| Component | Action | Description |
|-----------|--------|-------------|
| `UserService.js` | Modify | Add session fields (chat_session_id, current_input, next_action) |
| `LangFlowHandler.js` | Modify | Add ROUTER and EXCEPTION_FLOW methods |
| `ConversationStateService.js` | Create | Session lifecycle management |
| `server.js` | Modify | Implement consolidated workflow |
| `utils/uuid.js` | Create | UUID v7 generation utility |
| `package.json` | Modify | Add `uuid` dependency |

## Implementation Steps

### Step 1: Add UUID v7 Dependency

```bash
pnpm add uuid
```

Create UUID utility:

```javascript
// src/utils/uuid.js
const { v7: uuidv7 } = require('uuid');

/**
 * Generate UUID v7 (time-ordered)
 * @returns {string} UUID v7 string
 */
function generateSessionId() {
    return uuidv7();
}

module.exports = { generateSessionId };
```

### Step 2: Update UserService for Session Management

Add new fields to user entity:

```javascript
// Additional fields for user entity
const userEntity = {
    // ... existing fields
    chat_session_id: string,      // UUID v7 session ID
    current_input: string,        // Last user message
    next_action: string,          // Processing state
    next_action_updated_at: string // State timestamp
};

// New methods
class UserService {
    // Get or create session ID for user
    async getOrCreateSession(email, channel) { }

    // Update conversation state
    async updateConversationState(email, channel, state) { }

    // Clear session
    async clearSession(email, channel) { }
}
```

### Step 3: Update LangFlowHandler

Add methods for ROUTER and EXCEPTION flows:

```javascript
class LangFlowHandler {
    constructor() {
        this.routerFlow = process.env.ROUTER_FLOW;
        this.exceptionFlow = process.env.EXCEPTION_FLOW;
    }

    /**
     * Send message to ROUTER flow for intent classification
     */
    async sendToRouter(params) {
        const { message, sessionId, userEmail, channel, webhookUrl } = params;
        // POST to ROUTER_FLOW with callback URL
    }

    /**
     * Send to EXCEPTION flow for error message generation
     */
    async sendToExceptionFlow(params) {
        const { reason, userEmail, channel } = params;
        // POST to EXCEPTION_FLOW for error handling
    }

    /**
     * Process routing result from webhook
     */
    async handleRoutingResult(params) {
        const { routingResult, sessionId, userEmail, originalInput } = params;
        // Based on routingResult, call appropriate flow or return response
    }
}
```

### Step 4: Create ConversationStateService

```javascript
/**
 * ConversationStateService
 *
 * Manages conversation state transitions:
 * - IDLE → WAITING_FOR_LANGFLOW
 * - WAITING_FOR_LANGFLOW → IT_KNOWLEDGE_SEARCH | IT_CREATE_TICKET | RESPONSE
 * - IT_KNOWLEDGE_SEARCH → COMPLETED | ERROR
 * - IT_CREATE_TICKET → COMPLETED | ERROR
 * - RESPONSE → COMPLETED
 */
class ConversationStateService {
    constructor(userService) {
        this.userService = userService;
    }

    // State constants
    static STATES = {
        IDLE: 'IDLE',
        WAITING_FOR_LANGFLOW: 'WAITING_FOR_LANGFLOW',
        IT_KNOWLEDGE_SEARCH: 'IT_KNOWLEDGE_SEARCH',
        IT_CREATE_TICKET: 'IT_CREATE_TICKET',
        RESPONSE: 'RESPONSE',
        COMPLETED: 'COMPLETED',
        ERROR: 'ERROR'
    };

    async transitionTo(email, channel, newState, metadata = {}) { }
    async getCurrentState(email, channel) { }
    async isValidTransition(currentState, newState) { }
}
```

### Step 5: Update server.js - Main Workflow

```javascript
async function handleTeamsMessage(activity) {
    const enrichedActivity = await msTeamsService.getUserInfo(activity);
    const userEmail = enrichedActivity.from?.email;
    const channel = 'msteams';

    // 1. SEND TYPING INDICATOR
    await msTeamsService.sendTypingIndicator(
        enrichedActivity.serviceUrl,
        enrichedActivity.conversation?.id
    );

    // 2. CHECK DEBOUNCE
    const debounceCheck = requestDebounceService.canProcessRequest(userEmail, channel);
    if (!debounceCheck.canProcess) {
        // Handle debounce
        return;
    }
    requestDebounceService.startProcessing(userEmail, channel, activity.text);

    // 3. VALIDATE USER (Azure Table → ServiceNow → Exception)
    const validation = await validateUserWithFallback(userEmail, channel, enrichedActivity);
    if (!validation.valid) {
        // Call EXCEPTION_FLOW and send error message
        await handleInvalidUser(enrichedActivity, validation.reason);
        requestDebounceService.endProcessing(userEmail, channel);
        return;
    }

    // 4. SAVE CONVERSATION INFO
    await saveConversationInfo(enrichedActivity, userEmail);

    // 5. GET OR CREATE SESSION ID (UUID v7)
    const sessionId = await userService.getOrCreateSession(userEmail, channel);

    // 6. SEND TYPING INDICATOR
    await msTeamsService.sendTypingIndicator(
        enrichedActivity.serviceUrl,
        enrichedActivity.conversation?.id
    );

    // 7. UPDATE STATE: WAITING_FOR_LANGFLOW
    await conversationStateService.transitionTo(userEmail, channel, 'WAITING_FOR_LANGFLOW', {
        current_input: activity.text
    });

    // 8. CALL LANGFLOW ROUTER (async - webhook callback)
    const result = await langFlowHandler.sendToRouter({
        message: activity.text,
        sessionId,
        userEmail,
        channel,
        webhookUrl: `${WEBHOOK_BASE_URL}/api/webhook/langflow/routing`
    });

    // 9. Note: Don't end processing here - webhook will handle completion
}
```

### Step 6: Update Webhook Endpoint for Routing

```javascript
/**
 * Webhook for LangFlow routing results
 * POST /api/webhook/langflow/routing
 */
app.post('/api/webhook/langflow/routing', verifyApiKey, async (req, res) => {
    const { session_id, user_email, channel, routing_result, response_text, metadata } = req.body;

    // 1. Validate session
    const userState = await userService.getUser(user_email, channel);
    if (!userState || userState.chat_session_id !== session_id) {
        return res.status(400).json({ error: 'Invalid session' });
    }

    // 2. Get conversation info
    const conversation = await userService.getConversationByEmail(user_email, channel);
    if (!conversation) {
        return res.status(404).json({ error: 'Conversation not found' });
    }

    // 3. Send typing indicator
    await msTeamsService.sendTypingIndicator(
        conversation.serviceUrl,
        conversation.conversationId
    );

    // 4. Process based on routing result
    let finalResponse;
    switch (routing_result) {
        case 'IT_KNOWLEDGE_SEARCH':
            finalResponse = await langFlowHandler.callKBSearchFlow(metadata);
            break;
        case 'IT_CREATE_TICKET':
            finalResponse = await langFlowHandler.callTicketingFlow(metadata);
            break;
        case 'RESPONSE':
        default:
            finalResponse = response_text;
            break;
    }

    // 5. Send response to user
    await msTeamsService.sendProactiveMessage({
        serviceUrl: conversation.serviceUrl,
        conversationId: conversation.conversationId,
        message: finalResponse
    });

    // 6. Update state: COMPLETED
    await conversationStateService.transitionTo(user_email, channel, 'COMPLETED');

    // 7. End processing (allow new messages)
    requestDebounceService.endProcessing(user_email, channel);

    return res.status(200).json({ success: true });
});
```

### Step 7: Exception Flow Handler

```javascript
async function handleInvalidUser(activity, reason) {
    // 1. Call EXCEPTION_FLOW
    const errorResponse = await langFlowHandler.sendToExceptionFlow({
        reason,
        context: 'user_validation_failed'
    });

    const errorMessage = errorResponse?.text ||
        "I'm sorry, but I wasn't able to verify your account. Please contact your administrator.";

    // 2. Send error message to user
    await msTeamsService.sendProactiveMessage({
        serviceUrl: activity.serviceUrl,
        conversationId: activity.conversation?.id,
        message: errorMessage
    });
}
```

## File Changes Summary

### New Files

| File | Description |
|------|-------------|
| `src/utils/uuid.js` | UUID v7 generation utility |
| `src/Services/ConversationStateService.js` | Session state management |

### Modified Files

| File | Changes |
|------|---------|
| `package.json` | Add `uuid` dependency |
| `src/Services/UserService.js` | Add session fields and methods |
| `src/Handlers/LangFlowHandler.js` | Add ROUTER, EXCEPTION flow methods |
| `src/server.js` | Implement consolidated workflow |

## Environment Variables

Add these to `.env`:

```bash
# New/Updated LangFlow Configuration
ROUTER_FLOW=<router_flow_id>
EXCEPTION_FLOW=<exception_flow_id>

# Webhook Configuration
WEBHOOK_BASE_URL=https://your-bot-framework-server.com
```

## Testing Strategy

### Unit Tests

1. **UserService**
   - `getOrCreateSession` - creates new session if none exists
   - `getOrCreateSession` - returns existing session
   - `updateConversationState` - updates state correctly
   - `clearSession` - clears session ID

2. **LangFlowHandler**
   - `sendToRouter` - sends correct payload
   - `sendToExceptionFlow` - handles error cases
   - `handleRoutingResult` - routes correctly

3. **ConversationStateService**
   - State transitions work correctly
   - Invalid transitions rejected

### Integration Tests

1. Full workflow - new user validation through ServiceNow
2. Full workflow - existing user from Azure Table
3. Full workflow - invalid user triggers exception flow
4. Webhook routing - IT_KNOWLEDGE_SEARCH flow
5. Webhook routing - IT_CREATE_TICKET flow
6. Webhook routing - direct RESPONSE

### Manual Testing Checklist

- [ ] New user in MS Teams - first message
- [ ] Returning user - message within same session
- [ ] Invalid user - error message displayed
- [ ] KB Search intent - correct flow triggered
- [ ] Ticket creation intent - correct flow triggered
- [ ] Chitchat - direct response
- [ ] Typing indicators visible at each step
- [ ] Session persists across multiple messages
- [ ] Session cleared after timeout/reset

## Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| UUID v7 not available in older Node | Session IDs fail | Use `uuid` npm package (supports v7) |
| Azure Table write conflicts | State inconsistency | Use optimistic concurrency with ETags |
| LangFlow ROUTER timeout | User stuck | Timeout handling + error message |
| Webhook not called | Processing never completes | Timeout cleanup in debounce service |

## Rollback Plan

If issues arise:

1. Revert `server.js` to previous workflow
2. Remove new service files
3. Revert `UserService.js` changes
4. Session fields in Azure Table can remain (unused)

## Timeline

| Step | Duration | Dependencies |
|------|----------|--------------|
| Step 1: UUID setup | 15 min | None |
| Step 2: UserService | 30 min | Step 1 |
| Step 3: LangFlowHandler | 45 min | None |
| Step 4: ConversationStateService | 30 min | Step 2 |
| Step 5: server.js workflow | 1 hour | Steps 2-4 |
| Step 6: Webhook routing | 45 min | Steps 3, 5 |
| Step 7: Exception handling | 30 min | Steps 3, 6 |
| Testing | 1-2 hours | All steps |

**Total Estimated Time**: 5-6 hours

## Post-Implementation

1. Monitor error rates in App Insights
2. Review LangFlow flow execution times
3. Verify session persistence across deployments
4. Document any configuration changes for operations
