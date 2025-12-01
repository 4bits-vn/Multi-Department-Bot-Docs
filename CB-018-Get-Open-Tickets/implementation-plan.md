# CB-018: Get Open Tickets - Implementation Plan

## Overview

This document outlines the implementation plan for the "Get Open Tickets" feature. The implementation follows a two-phase LangFlow communication pattern where the bot first acknowledges the request, then fetches data from ITSM, and finally renders the results.

## Technical Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              BOT FRAMEWORK SERVER                                    │
│                                                                                     │
│  ┌───────────────────┐    ┌───────────────────┐    ┌───────────────────────────┐   │
│  │   Bot Controller  │───▶│  LangFlowHandler  │───▶│  TicketingFlow            │   │
│  │                   │    │  (processRouting) │    │  (handleGetOpenTickets)   │   │
│  └───────────────────┘    └───────────────────┘    └───────────────────────────┘   │
│            │                                                    │                   │
│            │                                                    ▼                   │
│            │                                        ┌───────────────────────────┐   │
│            │                                        │     IncidentService       │   │
│            │                                        │     (getIncidents)        │   │
│            │                                        └───────────────────────────┘   │
│            │                                                    │                   │
│            ▼                                                    ▼                   │
│  ┌───────────────────┐                              ┌───────────────────────────┐   │
│  │  MSTeamsService   │◀─────────────────────────────│  Adaptive Card Generator  │   │
│  │  (sendProactive)  │                              │  (buildTicketListCard)    │   │
│  └───────────────────┘                              └───────────────────────────┘   │
│                                                                                     │
└─────────────────────────────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              LANGFLOW SERVER                                         │
│                                                                                     │
│  ┌───────────────────────────────────────────────────────────────────────────┐     │
│  │                      GET_OPEN_TICKETS_FLOW                                 │     │
│  │                                                                            │     │
│  │  Input Modes:                                                              │     │
│  │  1. Initial Request: { "action": "FETCH" }                                │     │
│  │     → Returns: { message: "Let me fetch...", action: "GET_OPEN_TICKETS" } │     │
│  │                                                                            │     │
│  │  2. Render Results: { "signal": "RENDER_RESULTS", tickets: [...] }        │     │
│  │     → Returns: { message: "Found X tickets...", action: "COMPLETED" }     │     │
│  └───────────────────────────────────────────────────────────────────────────┘     │
│                                                                                     │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

## Implementation Steps

### Step 1: Update LangFlow Configuration

**File**: `botframework/src/Handlers/LangFlow/config.js`

Add ticket management action constants:

```javascript
// ============================================
// TICKET MANAGEMENT ACTIONS
// ============================================

/**
 * Ticket Management Sub-Actions
 * Returned by SUB_ROUTING - Ticket Management flow
 * @readonly
 * @enum {string}
 */
const TICKET_ACTIONS = Object.freeze({
    /** Create a new ticket */
    CREATE_TICKET: "CREATE_TICKET",
    /** Check status of specific ticket */
    CHECK_TICKET_STATUS: "CHECK_TICKET_STATUS",
    /** Get list of all user's open tickets */
    GET_ALL_TICKETS: "GET_ALL_TICKETS",
    /** Get open tickets (action returned by LangFlow) */
    GET_OPEN_TICKETS: "GET_OPEN_TICKETS",
    /** Other/fallback */
    OTHER: "OTHER",
});

// Add new flow ID
const GET_OPEN_TICKETS_FLOW = process.env.GET_OPEN_TICKETS_FLOW || "";
```

### Step 2: Refactor TicketingFlow Handler

**File**: `botframework/src/Handlers/LangFlow/flows/TicketingFlow.js`

Refactor to handle the two-phase GET_OPEN_TICKETS flow:

```javascript
/**
 * Handle GET_OPEN_TICKETS action
 * Two-phase communication:
 * 1. Get acknowledgement message from LangFlow
 * 2. Fetch tickets from ITSM
 * 3. Get render message from LangFlow with results
 *
 * @param {Object} params - Request parameters
 * @returns {Promise<Object>} Result with messages and adaptive card
 */
async handleGetOpenTickets(params) {
    const { sessionId, userEmail, channel } = params;

    // Phase 1: Get acknowledgement from LangFlow
    const ackResult = await this._callGetOpenTicketsFlow({
        action: 'FETCH',
        sessionId,
        userEmail
    });

    // Return acknowledgement immediately
    // (caller will send via proactive message)
    const acknowledgement = {
        type: 'acknowledgement',
        message: ackResult.message,
        action: ackResult.action // Should be 'GET_OPEN_TICKETS'
    };

    // Phase 2: Fetch tickets from ITSM
    const ticketsResult = await this._fetchUserTickets(userEmail);

    // Phase 3: Get render message from LangFlow
    const renderResult = await this._callGetOpenTicketsFlow({
        signal: 'RENDER_RESULTS',
        ticketsFound: ticketsResult.tickets.length > 0,
        ticketCount: ticketsResult.tickets.length,
        sessionId,
        userEmail
    });

    // Build adaptive card if tickets found
    let adaptiveCard = null;
    if (ticketsResult.tickets.length > 0) {
        adaptiveCard = this._buildTicketListCard(ticketsResult.tickets);
    }

    return {
        success: true,
        acknowledgement,
        response: {
            message: renderResult.message,
            action: 'COMPLETED',
            adaptiveCard
        },
        tickets: ticketsResult.tickets
    };
}
```

### Step 3: Implement Ticket Fetch Logic

Add method to fetch tickets from ITSM:

```javascript
/**
 * Fetch user's open tickets from ITSM
 * @private
 */
async _fetchUserTickets(userEmail) {
    try {
        const result = await incidentService.getIncidents({
            callerEmail: userEmail,
            state: 'open',
            limit: 10,
            orderBy: 'sys_created_on',
            orderDir: 'desc'
        });

        return {
            success: true,
            tickets: result.incidents || [],
            total: result.total || 0
        };
    } catch (error) {
        this.logger.error('Failed to fetch user tickets', {
            userEmail,
            error: error.message
        });

        return {
            success: false,
            tickets: [],
            error: error.message
        };
    }
}
```

### Step 4: Create Adaptive Card Builder

**File**: `botframework/src/Handlers/LangFlow/flows/TicketingFlow.js` (or new file)

```javascript
/**
 * Build Adaptive Card for ticket list display
 * @private
 * @param {Array} tickets - Array of normalized incidents
 * @returns {Object} Adaptive Card JSON
 */
_buildTicketListCard(tickets) {
    const ticketItems = tickets.map(ticket => ({
        type: "Container",
        items: [
            {
                type: "ColumnSet",
                columns: [
                    {
                        type: "Column",
                        width: "auto",
                        items: [{
                            type: "TextBlock",
                            text: ticket.number,
                            weight: "Bolder",
                            color: "Accent"
                        }]
                    },
                    {
                        type: "Column",
                        width: "stretch",
                        items: [{
                            type: "TextBlock",
                            text: ticket.shortDescription,
                            wrap: true
                        }]
                    },
                    {
                        type: "Column",
                        width: "auto",
                        items: [{
                            type: "TextBlock",
                            text: ticket.stateDisplay,
                            color: this._getStateColor(ticket.state)
                        }]
                    }
                ]
            }
        ],
        selectAction: {
            type: "Action.OpenUrl",
            url: ticket.url,
            title: "View in ServiceNow"
        },
        separator: true
    }));

    return {
        type: "AdaptiveCard",
        $schema: "http://adaptivecards.io/schemas/adaptive-card.json",
        version: "1.4",
        body: [
            {
                type: "TextBlock",
                text: `📋 Your Open Tickets (${tickets.length})`,
                weight: "Bolder",
                size: "Medium"
            },
            ...ticketItems
        ]
    };
}

/**
 * Get color for ticket state
 * @private
 */
_getStateColor(state) {
    const colors = {
        'new': 'Attention',
        'in_progress': 'Accent',
        'on_hold': 'Warning',
        'pending': 'Warning',
        'resolved': 'Good',
        'closed': 'Good'
    };
    return colors[state] || 'Default';
}
```

### Step 5: Update LangFlowHandler processRoutingResult

**File**: `botframework/src/Handlers/LangFlow/index.js`

Update the routing processor to handle GET_ALL_TICKETS action:

```javascript
// Inside processRoutingResult switch statement
case ROUTING_RESULTS.IT_TICKET_MGMT:
    // Check for sub-action from ticket management sub-routing
    const ticketAction = metadata.action || metadata.ticketAction;

    if (ticketAction === TICKET_ACTIONS.GET_ALL_TICKETS) {
        result = await this.callTicketingFlow({
            action: 'get_open_tickets',
            sessionId,
            userEmail,
            channel: metadata.channel,
            metadata
        });
    } else {
        // Default ticket management behavior
        result = await this.callTicketingFlow({
            action: ticketAction || 'manage',
            sessionId,
            userEmail,
            ticketData: metadata.ticketData || {},
            metadata: { ...metadata, originalInput }
        });
    }
    break;
```

### Step 6: Update Bot Controller Message Flow

**File**: `botframework/src/controllers/bot.controller.js`

The existing conversation loop pattern will handle the two-phase flow:

```javascript
// In handleRouterResponse - no changes needed
// The conversation loop already supports:
// 1. Sending acknowledgement message (first proactive message)
// 2. Processing GO action to continue
// 3. Sending final response (second proactive message)

// The TicketingFlow will return:
// First iteration: { action: 'GO', message: 'Fetching...', nextAction: 'GET_OPEN_TICKETS' }
// Second iteration: { action: 'COMPLETED', message: 'Found X tickets', adaptiveCard: {...} }
```

## File Changes Summary

| File | Action | Description |
|------|--------|-------------|
| `Handlers/LangFlow/config.js` | Modify | Add TICKET_ACTIONS constants, GET_OPEN_TICKETS_FLOW ID |
| `Handlers/LangFlow/flows/TicketingFlow.js` | Modify | Add handleGetOpenTickets(), ticket card builder |
| `Handlers/LangFlow/index.js` | Modify | Update processRoutingResult for GET_ALL_TICKETS |
| `flows/GET_OPEN_TICKETS/` | Create | New LangFlow flow definition (documented) |

## LangFlow GET_OPEN_TICKETS Flow Design

The LangFlow flow needs to handle two input modes:

### Input Mode 1: Initial Fetch Request
```json
{
  "action": "FETCH"
}
```

**Output**:
```json
{
  "message": "Let me fetch your open tickets. One moment please...",
  "action": "GET_OPEN_TICKETS"
}
```

### Input Mode 2: Render Results
```json
{
  "signal": "RENDER_RESULTS",
  "ticketsFound": true,
  "ticketCount": 3
}
```

**Output (tickets found)**:
```json
{
  "message": "I found 3 open tickets in your account. Here they are:",
  "action": "COMPLETED"
}
```

**Output (no tickets)**:
```json
{
  "message": "Good news! I couldn't find any open tickets for your account. Your queue is clear! 🎉",
  "action": "COMPLETED"
}
```

## Testing Strategy

### Unit Tests
- [ ] Test TicketingFlow.handleGetOpenTickets() with mock ITSM data
- [ ] Test adaptive card generation with various ticket counts
- [ ] Test error handling for ITSM failures

### Integration Tests
- [ ] Test end-to-end flow with LangFlow mock responses
- [ ] Test ITSM integration with ServiceNow sandbox

### Manual Tests
- [ ] MS Teams: User with open tickets
- [ ] MS Teams: User with no open tickets
- [ ] MS Teams: ITSM service unavailable

## Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| ITSM API timeout | User waits too long | Set 10s timeout, show error message |
| LangFlow flow not deployed | Feature broken | Fallback to hardcoded messages |
| Adaptive Card not rendering | Poor UX | Fallback to plain text list |

## Rollout Plan

1. **Development**: Implement and test locally
2. **LangFlow**: Create GET_OPEN_TICKETS flow in LangFlow UI
3. **Integration**: Deploy to dev environment
4. **Testing**: QA validation
5. **Production**: Deploy to production

## Dependencies

- LangFlow GET_OPEN_TICKETS flow must be created
- ServiceNow ITSM integration must be working
- Environment variable `GET_OPEN_TICKETS_FLOW` must be set
