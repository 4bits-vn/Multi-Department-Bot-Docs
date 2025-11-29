# ChatBot Platform - Architecture Documentation

> **Version**: 1.1
> **Last Updated**: November 2024
> **Status**: Production

## Table of Contents

- [Overview](#overview)
- [High-Level Architecture](#high-level-architecture)
- [User Conversation Life-Cycle](#user-conversation-life-cycle)
- [Key Components](#key-components)
- [Data Storage Architecture](#data-storage-architecture)
- [Message Flow](#message-flow)
- [Project Structure](#project-structure)
- [Key Design Decisions](#key-design-decisions)
- [Configuration](#configuration)

---

## Overview

The ChatBot Platform is an enterprise conversational AI system built on Microsoft Bot Framework, integrated with LangFlow for AI processing. It supports multiple channels (MS Teams, DirectLine) and provides IT support, medical knowledge search, and ticket management capabilities.

### Key Features

- **Multi-channel support**: MS Teams (primary) and DirectLine (web chat)
- **AI-powered routing**: Intent classification via LangFlow ROUTER
- **Conversation Loop**: Multi-step interaction support (up to 10 iterations)
- **User validation**: ServiceNow integration for enterprise user verification
- **Proactive messaging**: Send messages to users without requiring their input
- **Consolidated storage**: Single Azure Table for all user data

---

## High-Level Architecture

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                           USER INTERACTION LAYER                                  │
│  ┌─────────────────────┐    ┌─────────────────────┐                              │
│  │     MS Teams        │    │     DirectLine      │                              │
│  │     (Primary)       │    │     (Web Chat)      │                              │
│  └─────────┬───────────┘    └─────────┬───────────┘                              │
│            │                          │                                           │
└────────────┼──────────────────────────┼───────────────────────────────────────────┘
             │                          │
             ▼                          ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│                        AZURE BOT SERVICE                                          │
│  • Routes messages to Bot Framework                                               │
│  • Handles authentication                                                         │
│  • Channel adapters                                                               │
└────────────────────────────────────────────────────────────────────────────────────┘
             │
             ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│                          BOT FRAMEWORK SERVER (Node.js/Express)                   │
│                                                                                   │
│  ┌──────────────────────────────────────────────────────────────────────────┐    │
│  │  Endpoints:                                                               │    │
│  │    POST /api/messages  - Primary Azure Bot endpoint                       │    │
│  │    POST /Messages      - Legacy endpoint                                  │    │
│  │    POST /api/proactive - Proactive messaging                              │    │
│  │    GET  /api/admin/*   - Admin operations                                 │    │
│  └──────────────────────────────────────────────────────────────────────────┘    │
│                                                                                   │
│  ┌────────────────┐  ┌────────────────────┐  ┌─────────────────────────────┐     │
│  │  Bot Controller│──│ LangFlow Handler   │──│ Sub-Flow Handlers           │     │
│  │                │  │                    │  │  • IT KB Search             │     │
│  │  • Activity    │  │  • Router          │  │  • IT Ticket Mgmt           │     │
│  │    Handler     │  │  • Response Parser │  │  • IT Help Desk             │     │
│  │  • State Mgmt  │  │  • Exception Flow  │  │  • Med KB Search            │     │
│  │  • Loop Control│  │                    │  │  • Drug Search              │     │
│  └────────────────┘  └────────────────────┘  │  • Chit Chat                │     │
│                                               └─────────────────────────────┘     │
│                                                                                   │
│  ┌────────────────┐  ┌────────────────────┐  ┌─────────────────────────────┐     │
│  │ User Service   │  │ ConversationState  │  │ MSTeams Service             │     │
│  │                │  │ Service            │  │                             │     │
│  │ • Validation   │  │                    │  │  • Auth (OAuth 2.0)         │     │
│  │ • Sessions     │  │ • IDLE/PROCESSING  │  │  • Proactive Messages       │     │
│  │ • Conversations│  │ • Auto-Recovery    │  │  • Typing Indicators        │     │
│  └────────────────┘  └────────────────────┘  └─────────────────────────────┘     │
│                                                                                   │
└──────────────────────────────────────────────────────────────────────────────────┘
             │                          │                          │
             ▼                          ▼                          ▼
┌──────────────────────┐  ┌──────────────────────┐  ┌──────────────────────────────┐
│   AZURE TABLE        │  │   LANGFLOW           │  │   SERVICENOW                 │
│   STORAGE            │  │   (AI Processing)    │  │   (User Validation)          │
│                      │  │                      │  │                              │
│   BotUsers Table:    │  │   Flows:             │  │   sys_user table             │
│   • User profile     │  │   • ROUTER           │  │                              │
│   • Session state    │  │   • EXCEPTION        │  │                              │
│   • Conversation info│  │   • Sub-flows        │  │                              │
└──────────────────────┘  └──────────────────────┘  └──────────────────────────────┘
```

---

## User Conversation Life-Cycle

The conversation follows the **"Asclepius Router" pattern** with a **Conversation Loop**:

### Phase Overview

```
                            USER MESSAGE RECEIVED
                                    │
                                    ▼
    ┌───────────────────────────────────────────────────────────────┐
    │  PHASE 1: INGRESS & VALIDATION                                │
    │  ─────────────────────────────                                │
    │  1. Azure Bot Service calls POST /api/messages                │
    │  2. Bot Controller receives activity                          │
    │  3. Channel detection (msteams vs directline)                 │
    │  4. Send typing indicator to user                             │
    │  5. Check ConversationState - can we process?                 │
    │     ├─ PROCESSING → Reject: "Still processing your message"  │
    │     └─ IDLE/ERROR → Continue                                  │
    │  6. Get user info from Teams (email, name)                    │
    │  7. Validate user (Azure Table → ServiceNow if not cached)    │
    │     └─ Invalid? → Call EXCEPTION flow → Clear session         │
    │  8. Save/update conversation info to Azure Table              │
    │  9. Get or create session ID (UUID v7)                        │
    │ 10. Mark conversation as PROCESSING                           │
    │ 11. Get previous summary for context continuity               │
    └───────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
    ┌───────────────────────────────────────────────────────────────┐
    │  PHASE 2: ROUTER - INTENT CLASSIFICATION                      │
    │  ───────────────────────────────────────                      │
    │  Call LangFlow ROUTER flow with:                              │
    │    • input_value: [Context: summary]\n\nUser: message         │
    │    • session_id: UUID v7                                      │
    │    • sender_name: user email                                  │
    │                                                               │
    │  Router returns JSON:                                         │
    │  {                                                            │
    │    "route": "IT_KB_SEARCH | IT_TICKET_MGMT | ...",            │
    │    "message": "Acknowledgment to user",                       │
    │    "action": "GO | COMPLETED",                                │
    │    "summary": "Conversation context",                         │
    │    "keyword": "Extracted search term (optional)"              │
    │  }                                                            │
    └───────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
    ┌───────────────────────────────────────────────────────────────┐
    │  PHASE 3: CONVERSATION LOOP (max 10 iterations)               │
    │  ──────────────────────────────────────────────               │
    │                                                               │
    │  ┌────────────────── LOOP START ─────────────────┐            │
    │  │                                               │            │
    │  │  1. Send MESSAGE to user (proactive)          │            │
    │  │  2. Update SUMMARY in Azure Table             │            │
    │  │  3. Check ACTION:                             │            │
    │  │     │                                         │            │
    │  │     ├─► GO:                                   │            │
    │  │     │   • Send typing indicator               │            │
    │  │     │   • Start typing interval (3s)          │            │
    │  │     │   • Call sub-flow based on ROUTE        │            │
    │  │     │   • Process sub-flow result             │            │
    │  │     │   • If sub-flow returns GO:             │            │
    │  │     │     → Call ROUTER again with context    │            │
    │  │     │     → LOOP BACK to iteration start      │            │
    │  │     │   • If COMPLETED → EXIT LOOP            │            │
    │  │     │                                         │            │
    │  │     └─► COMPLETED:                            │            │
    │  │         → EXIT LOOP                           │            │
    │  │                                               │            │
    │  └───────────────────────────────────────────────┘            │
    │                                                               │
    │  SAFEGUARDS:                                                  │
    │  • Max 10 iterations (configurable via env)                   │
    │  • Auto-typing indicator every 3 seconds                      │
    │  • Error handling with graceful fallback                      │
    └───────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
    ┌───────────────────────────────────────────────────────────────┐
    │  PHASE 4: CLEANUP                                             │
    │  ────────────────                                             │
    │  1. Mark conversation as IDLE (completeProcessing)            │
    │  2. Ready for next user message                               │
    └───────────────────────────────────────────────────────────────┘
```

### Conversation Lifecycle Management (CB-006, CB-007)

The platform implements comprehensive conversation lifecycle management to handle:

1. **Session Timeouts**: Conversations that go inactive are automatically cleared
2. **User-Initiated Restart**: Users can type "restart" or use Teams command menu
3. **End-of-Conversation**: Natural conversation endings with "need more help?" follow-up
4. **Error Recovery**: Graceful handling of LangFlow errors
5. **Dynamic Messages (CB-007)**: User-facing messages generated via LangFlow for natural responses

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    CONVERSATION LIFECYCLE STATE MACHINE                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│    ┌────────┐     ┌─────────┐     ┌─────────────┐     ┌───────────┐        │
│    │  IDLE  │────►│ ACTIVE  │────►│ AWAITING    │────►│ COMPLETED │        │
│    └────────┘     └─────────┘     │ _INPUT      │     └───────────┘        │
│         ▲              │          └─────────────┘           │              │
│         │              │                │                   │              │
│         │              ▼                ▼                   │              │
│         │         ┌─────────┐     ┌───────────┐             │              │
│         └─────────│ ERROR   │◄────│ TIMED_OUT │◄────────────┘              │
│                   └─────────┘     └───────────┘                            │
│                                                                              │
│  Timeouts:                                                                   │
│  • Conversation: 30 minutes (CONVERSATION_TIMEOUT_MINUTES)                   │
│  • Response: 5 minutes (RESPONSE_TIMEOUT_MINUTES)                            │
│  • Processing: 2 minutes (PROCESSING_TIMEOUT_MINUTES)                        │
│  • Error Recovery: 30 seconds (ERROR_RECOVERY_SECONDS)                       │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Key Lifecycle Events:**

| Event | Trigger | Action |
|-------|---------|--------|
| Restart (Command) | User selects "restart" from Teams menu | Clear session, send confirmation |
| Restart (Natural) | User says "start over", "cancel", etc. | Clear session, send confirmation |
| Help | User selects "help" from Teams menu | Show capabilities and commands |
| Timeout | Inactivity > 30 minutes | Clear session, notify user on next message |
| End | User says "no thanks" to "need more help?" | Mark COMPLETED, send goodbye |
| Continue | User says "yes" to "need more help?" | Stay ACTIVE, ask how to help |
| Error | LangFlow returns system error | Clear session, send friendly error message |

**Teams Bot Commands (manifest.json):**

```json
"commandLists": [{
    "scopes": ["personal", "team", "groupChat"],
    "commands": [
        { "title": "restart", "description": "Restart conversation" },
        { "title": "help", "description": "Show available commands" }
    ]
}]
```

### Dynamic Lifecycle Messages (CB-007)

User-facing lifecycle messages are generated dynamically via LangFlow for more natural, varied responses:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    DYNAMIC MESSAGE GENERATION                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   Message Request                                                            │
│        │                                                                     │
│        ▼                                                                     │
│   ┌─────────────────────────┐                                               │
│   │ LifecycleMessageService │                                               │
│   │ getMessage(type, opts)  │                                               │
│   └───────────┬─────────────┘                                               │
│               │                                                              │
│               ├── CRITICAL message? ──► Use translated hardcoded message    │
│               │   (SERVICE_UNAVAILABLE)     (8 languages supported)         │
│               │                                                              │
│               ├── CONVERSATIONAL? ──────► CHITCHAT Flow ──► Dynamic msg    │
│               │   (Goodbye, Welcome,                                         │
│               │    Restart, etc.)                                            │
│               │                                                              │
│               └── ERROR? ───────────────► EXCEPTION Flow ──► Error msg      │
│                   (Timeout, RateLimit,                                       │
│                    UserInvalid, etc.)                                        │
│                                                                              │
│   Fallback: If LangFlow unavailable → Static hardcoded messages             │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Message Categories:**

| Category | Flow | Message Types |
|----------|------|---------------|
| CONVERSATIONAL | CHITCHAT | WELCOME, GOODBYE, RESTART_CONFIRMED, NEED_MORE_HELP, HOW_CAN_I_HELP, SESSION_EXPIRED |
| ERROR | EXCEPTION | GENERIC_ERROR, MAX_ITERATIONS_ERROR, PROCESSING_TIMEOUT, RATE_LIMIT_ERROR, USER_NOT_FOUND |
| CRITICAL | Hardcoded | SERVICE_UNAVAILABLE (translated to 8 languages) |

**Supported Languages for SERVICE_UNAVAILABLE:**

| Code | Language | Used When |
|------|----------|-----------|
| `en` | English | Default |
| `vi` | Vietnamese | User locale = vi |
| `fr` | French | User locale = fr |
| `es` | Spanish | User locale = es |
| `de` | German | User locale = de |
| `zh` | Chinese | User locale = zh |
| `ja` | Japanese | User locale = ja |
| `ko` | Korean | User locale = ko |

**Usage Example:**
```javascript
// Dynamic message via CHITCHAT flow
const goodbye = await lifecycleMessageService.getMessage(
    LifecycleMessageType.GOODBYE,
    { userEmail, userName: 'John' }
);

// Translated service unavailable (no LangFlow needed)
const unavailable = lifecycleMessageService.getServiceUnavailableMessage('vi');
```

### Conversation Loop Pattern

The Conversation Loop is implemented in `bot.controller.js` and allows multi-step interactions:

```javascript
// Simplified loop structure
while (iteration < MAX_ITERATIONS) {
    // 1. Send message to user
    if (message) await msTeamsService.sendProactiveMessage({ ... });

    // 2. Update conversation summary
    if (summary) await userService.updateUserState(email, channel, { conversation_summary: summary });

    // 3. Check action
    if (action === 'COMPLETED') break;

    if (action === 'GO') {
        // Call sub-flow based on route
        const result = await langFlowHandler.processRoutingResult({ routingResult: route, ... });

        // If sub-flow returns GO, call router again
        if (result.action === 'GO') {
            currentResult = await langFlowHandler.sendToRouter({ ... });
            continue;
        }

        // Sub-flow completed
        break;
    }
}
```

---

## Key Components

### 1. Bot Controller (`controllers/bot.controller.js`)

The main entry point for all incoming messages.

**Responsibilities:**
- Receive activities from Azure Bot Service
- Orchestrate the conversation flow
- Implement the Conversation Loop pattern
- Handle Teams vs DirectLine channels

**Key Functions:**
```javascript
handleBotMessage(req, res)        // Entry point
handleTeamsActivity(activity)     // Teams-specific handling
handleTeamsMessage(activity)      // Message processing
handleRouterResponse(params)      // Conversation loop implementation
```

### 2. LangFlow Handler (`Handlers/LangFlowHandler.js`)

Manages communication with LangFlow AI backend.

**Available Routes:**

| Route | Flow ID | Description |
|-------|---------|-------------|
| `IT_KB_SEARCH` | `IT_KB_SEARCH_FLOW` | IT knowledge base search |
| `IT_TICKET_MGMT` | `IT_TICKET_MGMT_FLOW` | Ticket operations (create, check, update) |
| `IT_HELP_DESK` | `IT_HELP_DESK_FLOW` | Live agent handover |
| `MED_KB_SEARCH` | `MED_KB_SEARCH_FLOW` | Medical knowledge base search |
| `MED_DRUG_SEARCH` | `MED_DRUG_SEARCH_FLOW` | Drug information search |
| `CHIT_CHAT` | `CHIT_CHAT_FLOW` | Casual conversation |
| `OTHER` | `OTHER_FLOW` | Fallback handler |

**Router Response Format:**
```json
{
  "route": "IT_KB_SEARCH",
  "message": "Let me search the knowledge base for you...",
  "action": "GO",
  "summary": "User is asking about VPN connection issues",
  "keyword": "VPN connection"
}
```

**Action Types:**
- `GO`: Continue processing - call the sub-flow
- `COMPLETED`: Done - send message and finish

### 3. Conversation State Service (`Services/ConversationStateService.js`)

In-memory state management to prevent duplicate message processing.

**States:**
```
IDLE ──────► PROCESSING ──────► IDLE
  │              │
  │              ▼
  │           ERROR
  │              │
  └──────────────┘ (auto-recovery)
```

**Timeouts:**
- Processing timeout: 2 minutes (auto-recover to IDLE)
- Error recovery: 30 seconds (auto-recover to IDLE)

### 3.1. Conversation Lifecycle Service (`Services/ConversationLifecycleService.js`)

Manages the complete conversation lifecycle including timeouts, restarts, and end-of-conversation handling.

**Responsibilities:**
- Detect and handle restart commands
- Manage session timeouts (conversation, response, processing)
- Handle "need more help?" flow after resolution
- Detect and handle LangFlow system errors

**Lifecycle States:**
```
IDLE ──► ACTIVE ──► AWAITING_INPUT ──► COMPLETED
  ▲          │            │                │
  │          ▼            ▼                │
  └──── ERROR ◄──── TIMED_OUT ◄────────────┘
```

**Key Methods:**
```javascript
checkAndHandle(params)         // Main lifecycle check before processing
setAwaitingInput(email, ch)    // Set state to AWAITING_INPUT
setCompleted(email, ch)        // Mark conversation as completed
detectSystemError(response)    // Detect LangFlow errors
handleSystemError(params)      // Handle errors gracefully
clearConversation(email, ch)   // Clear session and reset state
```

### 3.2. Lifecycle Message Service (`Services/LifecycleMessageService.js`)

Orchestrates dynamic message generation via LangFlow with fallback support (CB-007).

**Responsibilities:**
- Route messages to appropriate LangFlow flow (CHITCHAT vs EXCEPTION)
- Handle SERVICE_UNAVAILABLE with multi-language translations
- Provide fallback when LangFlow is unavailable
- Timeout protection for message generation

**Key Methods:**
```javascript
getMessage(type, options)           // Get dynamic or fallback message
getResolutionWithFollowUp(msg, opts) // Append "need more help?" to resolution
getErrorMessage(errorType, opts)    // Get error-specific message
getServiceUnavailableMessage(locale) // Get translated unavailable message
```

**Message Type Categories:**
```
CONVERSATIONAL → CHITCHAT flow
ERROR          → EXCEPTION flow
CRITICAL       → Hardcoded with translations (no LangFlow)
```

### 4. User Service (`Services/UserService.js`)

Manages user data in Azure Table Storage.

**Responsibilities:**
- User validation (Azure Table → ServiceNow)
- Session management (UUID v7)
- Conversation info storage
- State persistence

**Key Functions:**
```javascript
validateAndGetUser(email, channel, userData)  // Validate user
getOrCreateSession(email, channel)            // Session management
saveConversation(conversationData)            // Store conversation info
updateUserState(email, channel, stateData)    // Update state
findUserBySessionId(sessionId)                // Lookup by session
```

### 5. MSTeams Service (`Services/MSTeamsService.js`)

Handles communication with MS Teams.

**Responsibilities:**
- OAuth 2.0 authentication with Azure AD
- Proactive messaging
- Typing indicators
- User info retrieval

**Key Functions:**
```javascript
getAccessToken()                              // OAuth token management
getUserInfo(activity)                         // Get user email/name from Teams
sendProactiveMessage(params)                  // Send message to user
sendTypingIndicator(serviceUrl, conversationId) // Show typing
```

---

## Data Storage Architecture

### BotUsers Table (Consolidated)

All user data is stored in a single Azure Table for simplicity and performance.

```
┌─────────────────────────────────────────────────────────────────┐
│                    BotUsers TABLE (Consolidated)                 │
├─────────────────────────────────────────────────────────────────┤
│  PartitionKey: channel (msteams, directline)                    │
│  RowKey:       email (lowercase)                                │
├─────────────────────────────────────────────────────────────────┤
│  USER PROFILE                                                   │
│  ─────────────                                                  │
│  • email              - User email address                      │
│  • name               - Display name                            │
│  • serviceNowSysId    - ServiceNow sys_id                       │
│  • department         - User department                         │
├─────────────────────────────────────────────────────────────────┤
│  VALIDATION                                                     │
│  ──────────                                                     │
│  • validated          - Boolean: is user validated?             │
│  • validationSource   - Source: servicenow, manual              │
│  • validatedAt        - Timestamp of validation                 │
├─────────────────────────────────────────────────────────────────┤
│  SESSION STATE                                                  │
│  ─────────────                                                  │
│  • chat_session_id    - UUID v7 (shared with LangFlow)          │
│  • current_input      - Last user message                       │
│  • conversation_summary - Context for next turn                 │
│  • next_action        - Current processing state                │
│  • next_action_updated_at - State update timestamp              │
├─────────────────────────────────────────────────────────────────┤
│  CONVERSATION INFO                                              │
│  ─────────────────                                              │
│  • conversationId     - Teams/DirectLine conversation ID        │
│  • botId              - Bot registration ID                     │
│  • userId             - Teams/DirectLine user ID                │
│  • serviceUrl         - Bot connector service URL               │
├─────────────────────────────────────────────────────────────────┤
│  ACTIVITY                                                       │
│  ────────                                                       │
│  • lastActivityAt     - Last activity timestamp                 │
└─────────────────────────────────────────────────────────────────┘
```

### Why Consolidated Storage?

1. **Single query**: Get all user data in one Azure Table lookup
2. **Simpler code**: No joins or multi-table transactions
3. **Lower latency**: Fewer round trips to storage
4. **Easier maintenance**: One table schema to manage

---

## Message Flow

### Example: User asks "My laptop is slow"

```
User "My laptop is slow" → Teams → Azure Bot → POST /api/messages
    │
    ├─► [1] Check state: IDLE? ✓
    ├─► [2] Get user info: john@company.com
    ├─► [3] Validate user: Found in Azure Table ✓
    ├─► [4] Get session: abc123-def456...
    ├─► [5] Mark PROCESSING
    │
    ├─► [6] ROUTER call:
    │       Input: "My laptop is slow"
    │       Response: {
    │         route: "IT_KB_SEARCH",
    │         message: "Let me search our knowledge base...",
    │         action: "GO",
    │         keyword: "laptop slow performance"
    │       }
    │
    ├─► [7] Send message: "Let me search our knowledge base..."
    │
    ├─► [8] Sub-flow call (IT_KB_SEARCH):
    │       Query: "laptop slow performance"
    │       Response: "Here are some solutions:\n1. Clear temp files..."
    │
    ├─► [9] Send message: "Here are some solutions:\n1. Clear temp files..."
    │
    └─► [10] Mark IDLE, ready for next message
```

### Sequence Diagram

```
User        Teams       Azure Bot    BotFramework    LangFlow    Azure Table
 │            │             │             │             │             │
 │──message──►│             │             │             │             │
 │            │──activity──►│             │             │             │
 │            │             │──POST /api/messages──►│             │
 │            │             │             │             │             │
 │            │             │             │◄──getUser───────────────►│
 │            │             │             │             │             │
 │            │             │             │──sendToRouter──────────►│
 │            │             │             │             │◄──route───│
 │            │             │             │             │             │
 │            │◄──proactive msg──────────│             │             │
 │◄──message──│             │             │             │             │
 │            │             │             │             │             │
 │            │             │             │──callSubFlow───────────►│
 │            │             │             │             │◄──result──│
 │            │             │             │             │             │
 │            │◄──proactive msg──────────│             │             │
 │◄──message──│             │             │             │             │
 │            │             │             │             │             │
```

---

## Project Structure

```
botframework/src/
├── server.js                        # Entry point, startup & shutdown
├── app.js                           # Express configuration
├── config/
│   └── index.js                     # Environment configuration
│
├── controllers/
│   ├── bot.controller.js            # ⭐ Main message handler
│   ├── proactive.controller.js      # Proactive messaging API
│   ├── admin.controller.js          # Admin operations
│   └── index.js                     # Controller exports
│
├── Handlers/
│   ├── LangFlowHandler.js           # ⭐ LangFlow integration
│   └── AdaptiveCardHandler.js       # Adaptive Card builder
│
├── Services/
│   ├── UserService.js               # ⭐ User & session management
│   ├── ConversationStateService.js  # ⭐ State machine (per-message)
│   ├── ConversationLifecycleService.js # ⭐ Lifecycle management (CB-006)
│   ├── LifecycleMessageService.js   # ⭐ Dynamic message generation (CB-007)
│   ├── MSTeamsService.js            # ⭐ Teams communication
│   ├── ServiceNowService.js         # User validation
│   ├── Logger.js                    # Logging utility
│   ├── Az.StorageAccount.js         # Azure Blob Storage
│   ├── Az.StorageAccount.Table.js   # Azure Table Storage
│   └── Az.AppInsights.js            # Application Insights
│
├── routes/
│   ├── bot.routes.js                # Bot endpoints
│   ├── proactive.routes.js          # Proactive endpoints
│   ├── admin.routes.js              # Admin endpoints
│   ├── health.routes.js             # Health check endpoints
│   └── index.js                     # Route aggregator
│
├── middleware/
│   ├── auth.js                      # Authentication middleware
│   ├── errorHandler.js              # Global error handling
│   ├── requestLogger.js             # Request logging
│   └── index.js                     # Middleware exports
│
└── utils/
    ├── messageExtractor.js          # Parse LangFlow responses
    ├── kbFormatter.js               # Format KB search results
    ├── uuid.js                      # UUID v7 generation
    ├── intentDetectors.js           # Restart/end conversation detection (CB-006)
    ├── lifecycleMessages.js         # User-facing lifecycle messages (CB-006)
    ├── DACoreUtils.js               # Core utilities
    └── index.js                     # Utility exports
```

---

## Key Design Decisions

### 1. Synchronous Router Pattern

All user messages go through the ROUTER flow first for intent classification before being dispatched to specialized sub-flows.

**Benefits:**
- Centralized intent classification
- Consistent conversation context
- Easy to add new routes

### 2. Conversation Loop

Supports multi-step interactions where sub-flows can request continuation.

**Benefits:**
- Complex workflows (e.g., ticket creation with follow-up questions)
- Natural conversation flow
- Safeguards against infinite loops (max 10 iterations)

### 3. Proactive Messaging

All responses are sent via Bot Framework REST API as proactive messages, not inline replies.

**Benefits:**
- Can send multiple messages per turn
- Typing indicators between messages
- Works for long-running operations

### 4. One Message Per Turn

ConversationStateService prevents processing multiple messages simultaneously.

**Benefits:**
- Prevents race conditions
- Consistent conversation state
- Clear user experience

### 5. Consolidated Storage

Single Azure Table (BotUsers) stores user profile, session state, and conversation info.

**Benefits:**
- Single query for all user data
- Simpler code and maintenance
- Lower latency

### 6. UUID v7 Sessions

Time-sortable UUIDs shared between BotFramework and LangFlow.

**Benefits:**
- Consistent session tracking
- Time-based sorting capability
- No collision risk

---

## Configuration

### Environment Variables

```bash
# Server
PORT=3978
ENV=production

# Bot Configuration
BOT_APP_ID=<azure-app-id>
BOT_APP_PASSWORD=<azure-app-password>
BOT_TENANT_ID=<azure-tenant-id>

# LangFlow
LANGFLOW_URL=https://langflow.example.com
LANGFLOW_API_KEY=<api-key>
LANGFLOW_TIMEOUT=60000

# Flow IDs
ROUTER_FLOW=<flow-id>
EXCEPTION_FLOW=<flow-id>
IT_KB_SEARCH_FLOW=<flow-id>
IT_TICKET_MGMT_FLOW=<flow-id>
IT_HELP_DESK_FLOW=<flow-id>
MED_KB_SEARCH_FLOW=<flow-id>
MED_DRUG_SEARCH_FLOW=<flow-id>
CHIT_CHAT_FLOW=<flow-id>
OTHER_FLOW=<flow-id>

# Azure Storage
AZ_SA_CONFIG={"ACCOUNT_NAME":"...","SECRET":"...","URL":"..."}

# ServiceNow
SERVICENOW_URL=https://instance.service-now.com
SERVICENOW_API_KEY=<api-key>

# Conversation Loop
MAX_CONVERSATION_ITERATIONS=10

# Conversation Lifecycle (CB-006)
CONVERSATION_TIMEOUT_MINUTES=30
RESPONSE_TIMEOUT_MINUTES=5
PROCESSING_TIMEOUT_MINUTES=2
ERROR_RECOVERY_SECONDS=30
ENABLE_AUTO_ASK_MORE_HELP=true
ENABLE_TIMEOUT_WARNING=false

# Dynamic Lifecycle Messages (CB-007)
ENABLE_DYNAMIC_LIFECYCLE_MESSAGES=true   # Enable/disable dynamic messages
LIFECYCLE_MESSAGE_TIMEOUT=5000           # Timeout for message generation (ms)
LIFECYCLE_MESSAGE_CACHE_TTL=0            # Cache TTL (0 = disabled)
```

### API Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/messages` | Primary Azure Bot endpoint |
| POST | `/Messages` | Legacy endpoint |
| POST | `/api/proactive` | Send proactive messages |
| GET | `/api/admin/stats` | Conversation state statistics |
| POST | `/api/admin/clear-state` | Clear conversation state |
| GET | `/api/admin/state/:id` | Get state for conversation |
| GET | `/` | Health check |
| GET | `/health` | Health probe |
| GET | `/ready` | Readiness probe |

---

## References

- [Bot Framework REST API](https://learn.microsoft.com/en-us/azure/bot-service/rest-api/bot-framework-rest-connector-api-reference)
- [MS Teams Platform](https://learn.microsoft.com/en-us/microsoftteams/platform/)
- [Proactive Messages](https://learn.microsoft.com/en-us/microsoftteams/platform/bots/how-to/conversations/send-proactive-messages)
- [LangFlow API](https://docs.langflow.org/api-reference-api-examples)
- [Azure Table Storage](https://learn.microsoft.com/en-us/azure/cosmos-db/table/how-to-use-nodejs)

---

## Related Documentation

- [CB-003-Consolidated-Storage](./CB-003-Consolidated-Storage/) - Storage architecture details
- [CB-005-Conversation-Loop](./CB-005-Conversation-Loop/) - Conversation loop implementation
- [CB-006-Conversation-Lifecycle](./CB-006-Conversation-Lifecycle/) - Conversation lifecycle management
- [CB-007-Dynamic-Lifecycle-Messages](./CB-007-Dynamic-Lifecycle-Messages/) - Dynamic message generation via LangFlow
- [flows/router/README.md](../flows/router/README.md) - Router flow documentation
- [flows/CHITCHAT/README.md](../flows/CHITCHAT/README.md) - CHITCHAT flow for conversational messages
- [flows/EXCEPTION/README.md](../flows/EXCEPTION/README.md) - EXCEPTION flow for error messages
