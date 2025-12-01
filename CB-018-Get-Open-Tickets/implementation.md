# CB-018: Get Open Tickets - Implementation

## Summary

Implemented the "Get Open Tickets" feature for the Ticket Management flow. Users can now view their currently open IT support tickets through the chatbot with IT Agent-generated messages and a dropdown for ticket selection.

## Files Created/Modified

### Created Files

| File | Description |
|------|-------------|
| `docs/CB-018-Get-Open-Tickets/requirements.md` | Feature requirements and user story |
| `docs/CB-018-Get-Open-Tickets/implementation-plan.md` | Detailed implementation plan |
| `docs/CB-018-Get-Open-Tickets/implementation.md` | This implementation document |
| `flows/GET_OPEN_TICKETS/README.md` | LangFlow flow documentation |

### Modified Files

| File | Changes |
|------|---------|
| `src/Handlers/LangFlow/config.js` | Added `TICKET_ACTIONS` constants for ticket sub-routing |
| `src/Handlers/LangFlow/flows/TicketingFlow.js` | Added `handleGetOpenTickets()` with IT Agent integration and dropdown card |
| `src/Handlers/LangFlow/index.js` | Updated IT_TICKET_MGMT routing to handle GET_ALL_TICKETS action |

## Key Implementation Details

### 1. TICKET_ACTIONS Constants

Added new constants for ticket management sub-actions:

```javascript
const TICKET_ACTIONS = Object.freeze({
    CREATE_TICKET: "CREATE_TICKET",
    CHECK_TICKET_STATUS: "CHECK_TICKET_STATUS",
    GET_ALL_TICKETS: "GET_ALL_TICKETS",
    GET_OPEN_TICKETS: "GET_OPEN_TICKETS",
    RENDER_RESULTS: "RENDER_RESULTS",
    OTHER: "OTHER",
});
```

### 2. Multi-Phase IT Agent Communication

The `handleGetOpenTickets()` method now implements a 5-phase flow:

```javascript
async handleGetOpenTickets(params) {
    // PHASE 1: Call IT Agent for acknowledgement message
    const acknowledgementMessage = await this._callITAgentForAcknowledgement({...});

    // PHASE 2: Fetch tickets from ITSM
    const ticketsResult = await this._fetchUserTickets(userEmail);

    // PHASE 3: Get conversation context (summary)
    const summaryResult = await summaryFlow.execute({ sessionId, userEmail });

    // PHASE 4: Call IT Agent with results + context for response
    const responseMessage = await this._callITAgentForResponse({
        ticketsFound, ticketCount, ticketSummary, conversationContext, ...
    });

    // PHASE 5: Build Adaptive Card with dropdown
    const adaptiveCard = this._buildTicketDropdownCard(ticketsResult.tickets);

    return { acknowledgement, response, tickets, attachments };
}
```

### 3. IT Agent LangFlow Calls

#### Acknowledgement Call (Phase 1)

```javascript
payload = {
    action: "GET_OPEN_TICKETS",
    phase: "ACKNOWLEDGEMENT",
    request: "get_open_tickets"
}
```

#### Response Call (Phase 4)

```javascript
payload = {
    action: "RENDER_RESULTS",
    phase: "RENDER_RESULTS",
    ticketsFound: true,
    ticketCount: 3,
    ticketSummary: "- INC001: Issue... (In Progress, High)\n...",
    conversationContext: "User asked about tickets after..."
}
```

### 4. Adaptive Card with Dropdown

Created a rich Adaptive Card that displays:
- Header with ticket count
- **Dropdown (Input.ChoiceSet)** for ticket selection
- Quick summary list of first 5 tickets
- "View Ticket Details" submit button
- "View All in ServiceNow" link

```javascript
_buildTicketDropdownCard(tickets) {
    return {
        type: "AdaptiveCard",
        body: [
            { type: "TextBlock", text: "📋 Your Open Tickets (3)" },
            {
                type: "Input.ChoiceSet",
                id: "selectedTicket",
                style: "compact",
                placeholder: "Choose a ticket...",
                choices: [
                    { title: "🆕 INC001 - Login issue...", value: "INC001" },
                    // ...
                ]
            },
            // Quick summary list...
        ],
        actions: [
            { type: "Action.Submit", title: "View Ticket Details" },
            { type: "Action.OpenUrl", title: "View All in ServiceNow" }
        ]
    };
}
```

### 5. Conversation Context Integration

Uses `SummaryFlow` to get conversation context for the IT Agent:

```javascript
const summaryFlow = getSummaryFlow();
const summaryResult = await summaryFlow.execute({ sessionId, userEmail });
conversationContext = summaryResult?.summary || '';
```

## Flow Diagram

```
User: "Show me my tickets"
        │
        ▼
┌──────────────────────────────┐
│  MAIN ROUTER                  │
│  Route: IT_TICKET_MGMT        │
└───────────┬──────────────────┘
            │
            ▼
┌──────────────────────────────┐
│  SUB_ROUTING Ticket Mgmt      │
│  Action: GET_ALL_TICKETS      │
└───────────┬──────────────────┘
            │
            ▼
┌──────────────────────────────────────────────────┐
│  ticketingFlow.handleGetOpenTickets()             │
│                                                   │
│ PHASE 1: Call IT Agent → Acknowledgement msg      │
│          "Let me fetch your open tickets..."      │
│                                                   │
│ PHASE 2: Call IncidentService.getIncidents()      │
│          → Fetches tickets from ServiceNow        │
│                                                   │
│ PHASE 3: Call SummaryFlow.execute()               │
│          → Gets conversation context              │
│                                                   │
│ PHASE 4: Call IT Agent → Response msg             │
│          "I found 3 open tickets..."              │
│                                                   │
│ PHASE 5: Build Adaptive Card with dropdown        │
└───────────┬──────────────────────────────────────┘
            │
            ▼
┌──────────────────────────────────────────────────┐
│  Bot sends proactive message with:                │
│  - IT Agent generated response message            │
│  - Adaptive Card with dropdown for selection      │
└──────────────────────────────────────────────────┘
```

## Fallback Handling

If IT Agent (LangFlow) is unavailable, the system uses fallback messages:

```javascript
_getFallbackAcknowledgementMessage() {
    return "Let me fetch your open tickets. One moment please... 🔍";
}

_getFallbackResponseMessage(ticketsResult) {
    if (count === 0) {
        return "Good news! 🎉 I couldn't find any open tickets...";
    }
    return `I found ${count} open tickets. Please select from dropdown...`;
}
```

## Testing Notes

### Test Scenarios

1. **User with open tickets**:
   - Expected: IT Agent acknowledgement + dropdown card with tickets

2. **User with no open tickets**:
   - Expected: IT Agent response "I couldn't find any open tickets"

3. **IT Agent unavailable**:
   - Expected: Fallback messages used instead

4. **ITSM service unavailable**:
   - Expected: Error message from fallback

### Test Commands

```bash
# Run the bot locally
cd botframework
pnpm dev

# Test in Teams with messages like:
# - "Show me my tickets"
# - "What are my open tickets?"
# - "List my incidents"
```

## Future Enhancements

1. **Handle Dropdown Selection**: Process Action.Submit to show ticket details
2. **Pagination**: Add "Load more" for users with many tickets
3. **Filtering**: Allow filtering by priority, date range, etc.
4. **Quick Actions**: Add "Add Comment" or "Close Ticket" from card
5. **Real-time Updates**: WebSocket for ticket status changes

## Dependencies

- `IncidentService` - ITSM integration layer (existing)
- `MSTeamsService` - Proactive messaging (existing)
- `ConversationRoutingService` - Route lock management (existing)
- `SummaryFlow` - Conversation summarization (existing, CB-019)
- `IT_TICKET_MGMT_FLOW` - IT Agent LangFlow flow
- `SUB_ROUTING - Ticket Management` LangFlow flow (existing)

## Related Tickets

- CB-012: ITSM Integration APIs
- CB-016: Conversation Routing
- CB-019: Conversation Summarization (for context injection)
- CB-013: Chat Transcript Capture (for transcript attachment in ticket creation)
