# CB-018: Get Open Tickets Feature

## Ticket Information

| Field | Value |
|-------|-------|
| **Ticket ID** | CB-018 |
| **Title** | Get Open Tickets Feature |
| **Type** | Feature |
| **Priority** | High |
| **Created** | 2024-12-01 |
| **Status** | In Progress |

## Overview

Implement the "Get Open Tickets" functionality as part of the Ticket Management flow. This allows users to view their currently open IT support tickets through the chatbot.

## User Story

**As a user**, I want the bot to show a list of my opening tickets so that I can track the status of my IT support requests without leaving the chat interface.

## Workflow Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         GET OPEN TICKETS WORKFLOW                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  User: "Show me my tickets"                                                  │
│           │                                                                  │
│           ▼                                                                  │
│  ┌─────────────────────┐                                                    │
│  │   MAIN ROUTER       │                                                    │
│  │   (Intent: IT_TICKET_MGMT)                                               │
│  └──────────┬──────────┘                                                    │
│             │                                                                │
│             ▼                                                                │
│  ┌─────────────────────┐                                                    │
│  │  SUB_ROUTING -      │                                                    │
│  │  Ticket Management  │                                                    │
│  │  (Action: GET_ALL_TICKETS)                                               │
│  └──────────┬──────────┘                                                    │
│             │                                                                │
│             ▼                                                                │
│  ┌─────────────────────────────────────────────────────────────┐            │
│  │ GET_OPEN_TICKETS_FLOW (Phase 1: Acknowledgement)            │            │
│  │                                                              │            │
│  │  Response:                                                   │            │
│  │  {                                                           │            │
│  │    "message": "Let me fetch your open tickets...",          │            │
│  │    "action": "GET_OPEN_TICKETS"                             │            │
│  │  }                                                           │            │
│  └──────────────────────────┬──────────────────────────────────┘            │
│                              │                                               │
│                              ▼                                               │
│  ┌─────────────────────────────────────────────────────────────┐            │
│  │ BOT SENDS ACKNOWLEDGEMENT (Proactive Message)               │            │
│  │ "Let me fetch your open tickets..."                         │            │
│  └──────────────────────────┬──────────────────────────────────┘            │
│                              │                                               │
│                              ▼                                               │
│  ┌─────────────────────────────────────────────────────────────┐            │
│  │ BOT CALLS ITSM API (IncidentService.getIncidents)           │            │
│  │                                                              │            │
│  │  Params:                                                     │            │
│  │  - callerEmail: user's email                                │            │
│  │  - state: 'open'                                            │            │
│  │  - limit: 10                                                │            │
│  └──────────────────────────┬──────────────────────────────────┘            │
│                              │                                               │
│                 ┌────────────┴────────────┐                                  │
│                 │                         │                                  │
│                 ▼                         ▼                                  │
│         ┌───────────┐            ┌───────────────┐                          │
│         │ Has Tickets│            │ No Tickets    │                          │
│         └─────┬─────┘            └───────┬───────┘                          │
│               │                          │                                   │
│               ▼                          ▼                                   │
│  ┌─────────────────────────────────────────────────────────────┐            │
│  │ GET_OPEN_TICKETS_FLOW (Phase 2: Response Generation)        │            │
│  │                                                              │            │
│  │ Input: { signal: "RENDER_RESULTS", tickets: [...] }         │            │
│  │                                                              │            │
│  │ Response:                                                    │            │
│  │ - If tickets: "I found X open tickets..."                   │            │
│  │ - If no tickets: "I couldn't find any open tickets..."      │            │
│  └──────────────────────────┬──────────────────────────────────┘            │
│                              │                                               │
│                              ▼                                               │
│  ┌─────────────────────────────────────────────────────────────┐            │
│  │ BOT SENDS RESPONSE WITH ADAPTIVE CARD                       │            │
│  │                                                              │            │
│  │  Adaptive Card contains:                                     │            │
│  │  - Header: "Your Open Tickets"                              │            │
│  │  - Dropdown list of tickets (number, description, status)   │            │
│  │  - "View Details" button per ticket                         │            │
│  └─────────────────────────────────────────────────────────────┘            │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Functional Requirements

### FR-1: Intent Detection
- The bot MUST recognize ticket listing intents from user messages
- Examples: "show my tickets", "what are my open cases", "list my incidents"
- Intent classification uses the existing SUB_ROUTING - Ticket Management flow

### FR-2: Two-Phase LangFlow Communication
- **Phase 1**: Initial call to GET_OPEN_TICKETS_FLOW returns:
  - `message`: Acknowledgement text for the user
  - `action`: "GET_OPEN_TICKETS" to signal API call needed
- **Phase 2**: Second call with ticket results returns:
  - `message`: Response text ("Found X tickets" or "No tickets found")
  - `action`: "COMPLETED" to signal flow completion

### FR-3: ITSM Integration
- Bot MUST call the ITSM IncidentService to fetch user's open tickets
- Filter by user's email as caller
- Filter by state = 'open' (non-closed tickets)
- Limit results to reasonable number (default: 10)

### FR-4: Proactive Messaging
- Acknowledgement message MUST be sent immediately via proactive messaging
- Response with ticket list MUST be sent via proactive messaging

### FR-5: Adaptive Card Display
- When tickets are found, display them in an Adaptive Card
- Card MUST include:
  - Header with ticket count
  - List of tickets with: Number, Short Description, State, Priority
  - Action button to view details (link to ServiceNow portal)
- When no tickets found, display appropriate message

## Non-Functional Requirements

### NFR-1: Performance
- ITSM API call should complete within 10 seconds
- Acknowledgement message sent within 1 second of intent detection

### NFR-2: Error Handling
- Graceful error handling if ITSM service is unavailable
- User-friendly error messages via Exception flow

### NFR-3: Logging
- Log all ticket fetch attempts with user email
- Log ITSM API response times
- Do NOT log sensitive ticket data

## Acceptance Criteria

### AC-1: Happy Path - User Has Open Tickets
- [ ] User sends "show my tickets"
- [ ] Bot responds with acknowledgement: "Let me fetch your open tickets..."
- [ ] Bot retrieves tickets from ITSM
- [ ] Bot displays Adaptive Card with ticket list
- [ ] User can see ticket number, description, status for each ticket

### AC-2: No Tickets Found
- [ ] User sends "show my tickets"
- [ ] Bot responds with acknowledgement
- [ ] ITSM returns empty list
- [ ] Bot responds: "I couldn't find any open tickets for your account."

### AC-3: ITSM Service Error
- [ ] Bot handles ITSM service unavailability gracefully
- [ ] User receives friendly error message
- [ ] Error is logged appropriately

### AC-4: Ticket Details Navigation
- [ ] Each ticket in the list has a "View in ServiceNow" link
- [ ] Link opens ServiceNow portal in a new tab

## Dependencies

### Internal Dependencies
- `IncidentService` - ITSM integration layer
- `MSTeamsService` - Proactive messaging
- `TicketingFlow` - Existing ticket flow handler
- LangFlow `SUB_ROUTING - Ticket Management` flow

### External Dependencies
- ServiceNow API (via ITSM provider)
- LangFlow GET_OPEN_TICKETS_FLOW (needs to be created in LangFlow)

## Environment Variables

```bash
# New flow ID for Get Open Tickets
GET_OPEN_TICKETS_FLOW=<langflow_flow_id>

# Existing ITSM configuration (already configured)
SERVICENOW_INSTANCE=<instance>
SERVICENOW_URL=<url>
SERVICENOW_USERNAME=<user>
SERVICENOW_PASSWORD=<password>
```

## References

- [SUB_ROUTING - Ticket Management Flow](../../flows/SUB_ROUTING%20-%20Ticket%20Management/README.md)
- [ServiceNow Integration Guidelines](../../.cursor/rules/servicenow-integration.mdc)
- [ITSM Service Implementation](../../botframework/src/Services/ITSM/)
- [LangFlow Integration Guidelines](../../.cursor/rules/langflow-integration.mdc)
