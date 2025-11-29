# CB-001: Consolidated Bot Framework Workflow

## Ticket Information

- **Ticket ID**: CB-001
- **Title**: Consolidated Bot Framework Workflow
- **Type**: Enhancement
- **Priority**: High
- **Created**: 2025-01-15

## Overview

Implement a consolidated workflow for the Bot Framework server that handles user messages through a structured flow:

1. **User Validation** → Azure Storage Table → ServiceNow → Exception handling
2. **Session Management** → UUID v7 session ID management
3. **Intent Classification** → LangFlow ROUTER flow
4. **Async Processing** → LangFlow webhook callback
5. **Response Handling** → Route-based action execution

## Functional Requirements

### FR-001: User Validation Flow

When a user sends a message to the bot:

1. Check if user exists in Azure Storage Table (`BotUsers`)
2. If not found, query ServiceNow `sys_user` table for validation
3. If not found in ServiceNow, call LangFlow `EXCEPTION_FLOW` to generate error message
4. Store validated user info with MS Teams Conversation ID in Azure Storage Table

**Acceptance Criteria:**
- User validation checks Azure Table first (fast path)
- Falls back to ServiceNow if not in Azure Table
- Exception flow returns user-friendly error message
- User info includes: email, name, serviceNowSysId, channel, conversationId

### FR-002: Session Management

Implement chat session lifecycle management using UUID v7:

1. Check if `chat_session_id` exists in user's Azure Storage record
2. If no session exists, generate new UUID v7 session ID
3. Session ID persists throughout the conversation lifecycle
4. Session stored in user record in Azure Storage Table

**Acceptance Criteria:**
- UUID v7 used for session IDs (time-ordered)
- Session ID persists for conversation duration
- Session linked to user email + channel combination
- Session can be reset/cleared when needed

### FR-003: Intent Classification via LangFlow ROUTER

Call LangFlow ROUTER flow for intent classification:

1. Send user message to ROUTER flow with session ID
2. Update Azure Storage: `current_input` = user message
3. Update Azure Storage: `next_action` = "WAITING_FOR_LANGFLOW"
4. Capture acknowledgment from LangFlow

**Acceptance Criteria:**
- ROUTER flow receives: message, session_id, user_email, channel
- State tracked in Azure Storage Table
- Typing indicator sent while processing
- Request timeout handled gracefully

### FR-004: LangFlow Webhook Callback

LangFlow calls back to Bot Framework with routing results:

1. Receive webhook POST with: routing result, session ID, user_email
2. Look up user's conversation info from Azure Storage
3. Process based on routing result:
   - Call another flow (KB Search, Ticketing, etc.)
   - Return message directly to user via Teams REST API
4. Update Azure Storage: `next_action` = routing result or "COMPLETED"

**Acceptance Criteria:**
- Webhook endpoint handles LangFlow callbacks
- Session ID matches existing conversation
- Routing result determines next action
- Final response sent to Teams via proactive message

### FR-005: Typing Indicators

Send typing indicators for all operations:

1. Before user validation
2. Before LangFlow ROUTER call
3. During LangFlow processing
4. Before sending final response

**Acceptance Criteria:**
- Typing indicator sent at start of each operation
- Periodic typing indicators during long operations
- Good user experience with visual feedback

## Non-Functional Requirements

### NFR-001: Performance

- User validation: < 500ms (Azure Table) or < 2s (ServiceNow fallback)
- LangFlow request: Timeout after 60s
- Overall response time: < 90s for complex flows

### NFR-002: Reliability

- Retry logic for LangFlow calls (max 2 retries)
- Graceful handling of service unavailability
- Error messages always returned to user

### NFR-003: Scalability

- In-memory debounce for single instance
- Azure Table Storage for multi-instance state
- Stateless request handling where possible

### NFR-004: Security

- API key authentication for webhooks
- User validation against ServiceNow
- No sensitive data in logs

## Data Model

### BotUsers Table (Azure Storage)

| Field | Type | Description |
|-------|------|-------------|
| partitionKey | string | Channel (msteams, directline) |
| rowKey | string | User email (lowercase) |
| email | string | User email |
| name | string | Display name |
| serviceNowSysId | string | ServiceNow sys_id |
| department | string | User department |
| validated | boolean | Is user validated |
| validationSource | string | Source (servicenow, manual) |
| validatedAt | string | Validation timestamp |
| **chat_session_id** | string | Current session UUID v7 |
| **current_input** | string | Last user input |
| **next_action** | string | Current processing state |
| **next_action_updated_at** | string | State update timestamp |
| lastActivityAt | string | Last activity timestamp |

### BotConversations Table (Azure Storage)

| Field | Type | Description |
|-------|------|-------------|
| partitionKey | string | {email}_{botId}_{channel} |
| rowKey | string | User email |
| email | string | User email |
| botId | string | Bot ID |
| channel | string | Channel |
| conversationId | string | Teams conversation ID |
| userId | string | Teams user ID |
| serviceUrl | string | Bot connector URL |
| createdAt | string | Creation timestamp |
| lastActivityAt | string | Last activity timestamp |

## Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          USER SENDS MESSAGE                              │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 1. SEND TYPING INDICATOR                                                 │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 2. CHECK USER IN AZURE TABLE                                             │
│    ├─ Found & Validated ──────────────────────────────────────────┐     │
│    └─ Not Found ─┐                                                │     │
│                  ▼                                                │     │
│    3. CHECK USER IN SERVICENOW                                    │     │
│       ├─ Found & Active ─────────────────────────────────────┐    │     │
│       │  └─ Save to Azure Table                              │    │     │
│       └─ Not Found / Inactive ─┐                             │    │     │
│                                ▼                             │    │     │
│       4. CALL LANGFLOW EXCEPTION_FLOW                        │    │     │
│          └─ Return error message to user ─────────────► STOP │    │     │
│                                                              │    │     │
└──────────────────────────────────────────────────────────────┼────┼─────┘
                                                               │    │
                                    ┌──────────────────────────┘    │
                                    │      ┌───────────────────────┘
                                    ▼      ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 5. CHECK/CREATE SESSION ID (UUID v7)                                     │
│    └─ Update Azure Table: chat_session_id                               │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 6. SEND TYPING INDICATOR                                                 │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 7. CALL LANGFLOW ROUTER                                                  │
│    └─ Update Azure Table: current_input, next_action=WAITING_FOR_LANGFLOW│
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼ (Async - Bot Framework returns 200 OK)
┌─────────────────────────────────────────────────────────────────────────┐
│ 8. LANGFLOW PROCESSES AND CALLS WEBHOOK                                  │
│    Payload: { session_id, user_email, routing_result, ... }             │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 9. WEBHOOK RECEIVES ROUTING RESULT                                       │
│    ├─ routing_result = "IT_KNOWLEDGE_SEARCH" → Call KB Search Flow               │
│    ├─ routing_result = "IT_CREATE_TICKET" → Call Ticketing Flow           │
│    ├─ routing_result = "CHITCHAT" → Return response directly           │
│    └─ routing_result = "RESPONSE" → Send message to user               │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 10. SEND RESPONSE TO TEAMS VIA PROACTIVE MESSAGE                         │
│     └─ Update Azure Table: next_action=COMPLETED                        │
└─────────────────────────────────────────────────────────────────────────┘
```

## External Dependencies

### LangFlow
- **API Version**: v1
- **Documentation**: https://docs.langflow.org/api-reference-api-examples
- **Endpoints Used**:
  - `POST /api/v1/run/{flow_id}` - Flow execution
  - Environment variables: `ROUTER_FLOW`, `EXCEPTION_FLOW`

### Azure Table Storage
- **SDK**: `@azure/data-tables` v13.3.0
- **Documentation**: https://learn.microsoft.com/en-us/azure/cosmos-db/table/how-to-use-nodejs
- **Tables**: `BotUsers`, `BotConversations`

### ServiceNow
- **API**: Table API
- **Documentation**: https://www.servicenow.com/docs/bundle/zurich-api-reference/page/integrate/inbound-rest/concept/c_TableAPI.html
- **Table**: `sys_user`

### MS Teams / Bot Framework
- **Documentation**: https://learn.microsoft.com/en-us/azure/bot-service/rest-api/bot-framework-rest-connector-api-reference
- **Features Used**: Proactive messages, typing indicators

### UUID v7
- **Package**: `uuid` (npm)
- **Documentation**: https://www.npmjs.com/package/uuid
- **Usage**: Time-ordered unique session IDs

## Environment Variables

```bash
# LangFlow Configuration
LANGFLOW_URL=https://langflow.example.com
LANGFLOW_API_KEY=xxx
ROUTER_FLOW=router_flow_id
EXCEPTION_FLOW=exception_flow_id
LANGFLOW_TIMEOUT=60000

# Webhook Configuration
WEBHOOK_BASE_URL=https://botframework.example.com
CODE_ACCESS_API=webhook_api_key

# Azure Storage
AZ_SA_CONFIG={"ACCOUNT_NAME":"xxx","SECRET":"xxx","URL":"xxx","CONN_STRING":"xxx"}

# ServiceNow
SERVICENOW_URL=https://instance.service-now.com
SERVICENOW_USERNAME=xxx
SERVICENOW_PASSWORD=xxx

# Bot Configuration
BOT_ENTRIES=[{"appId":"xxx","appPassword":"xxx","tenant":"xxx"}]
```

## Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| LangFlow timeout | User gets no response | Timeout handling + error message |
| ServiceNow unavailable | New users can't be validated | Cache validated users, graceful error |
| Azure Storage unavailable | State lost | In-memory fallback for development |
| Session ID collision | Wrong session used | UUID v7 ensures uniqueness |
| Concurrent messages | Duplicate processing | Request debounce service |

## Testing Checklist

- [ ] User validation - new user via ServiceNow
- [ ] User validation - existing user in Azure Table
- [ ] User validation - invalid user (exception flow)
- [ ] Session creation - new conversation
- [ ] Session continuity - existing conversation
- [ ] LangFlow ROUTER - successful classification
- [ ] LangFlow ROUTER - timeout handling
- [ ] Webhook - routing to KB Search
- [ ] Webhook - routing to Ticketing
- [ ] Webhook - direct response
- [ ] Typing indicators - visible at each step
- [ ] Error handling - graceful error messages
