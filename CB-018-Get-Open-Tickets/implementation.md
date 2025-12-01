# CB-018: Get Open Tickets - Implementation

## Summary

Implemented the "Get Open Tickets" feature for the Ticket Management flow. Users can now view their currently open IT support tickets through the chatbot.

## Files Created/Modified

### Created Files

| File | Description |
|------|-------------|
| `docs/CB-018-Get-Open-Tickets/requirements.md` | Feature requirements and user story |
| `docs/CB-018-Get-Open-Tickets/implementation-plan.md` | Detailed implementation plan |
| `docs/CB-018-Get-Open-Tickets/implementation.md` | This implementation document |

### Modified Files

| File | Changes |
|------|---------|
| `src/Handlers/LangFlow/config.js` | Added `TICKET_ACTIONS` constants for ticket sub-routing |
| `src/Handlers/LangFlow/flows/TicketingFlow.js` | Added `handleGetOpenTickets()` and Adaptive Card builder |
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

### 2. TicketingFlow.handleGetOpenTickets()

New method that handles the complete "Get Open Tickets" flow:

```javascript
async handleGetOpenTickets(params) {
    // Phase 1: Generate acknowledgement message
    const acknowledgement = {
        type: 'acknowledgement',
        message: this._getAcknowledgementMessage(),
        action: TICKET_ACTIONS.GET_OPEN_TICKETS
    };

    // Phase 2: Fetch tickets from ITSM
    const ticketsResult = await this._fetchUserTickets(userEmail);

    // Phase 3: Generate response with Adaptive Card
    const responseMessage = this._getResponseMessage(ticketsResult);
    let adaptiveCard = null;
    if (ticketsResult.success && ticketsResult.tickets.length > 0) {
        adaptiveCard = this._buildTicketListCard(ticketsResult.tickets);
    }

    return {
        success: true,
        acknowledgement,
        response: {
            message: responseMessage,
            action: 'COMPLETED',
            adaptiveCard
        },
        tickets: ticketsResult.tickets,
        attachments: adaptiveCard ? [{
            contentType: 'application/vnd.microsoft.card.adaptive',
            content: adaptiveCard
        }] : undefined
    };
}
```

### 3. ITSM Integration

Integrated with existing `IncidentService` to fetch user's open tickets:

```javascript
async _fetchUserTickets(userEmail) {
    const result = await incidentService.getIncidents({
        callerEmail: userEmail,
        state: 'open',
        limit: 10,
        orderBy: 'sys_created_on',
        orderDir: 'desc'
    });
    // ...
}
```

### 4. Adaptive Card for Ticket List

Created a rich Adaptive Card that displays:
- Header with ticket count
- List of tickets showing:
  - Ticket number (clickable to open in ServiceNow)
  - Short description
  - State with icon and color coding
  - Priority level
  - Created date
- "View All in ServiceNow" button

### 5. LangFlowHandler Routing

Updated `processRoutingResult` to handle the GET_ALL_TICKETS action:

```javascript
case ROUTING_RESULTS.IT_TICKET_MGMT:
    const ticketAction = metadata.action || metadata.ticketAction;

    if (ticketAction === "GET_ALL_TICKETS" || ticketAction === "GET_OPEN_TICKETS") {
        result = await ticketingFlow.handleGetOpenTickets({
            sessionId,
            userEmail,
            channel: metadata.channel || 'msteams',
            metadata: { ...metadata, originalInput }
        });
    } else {
        // Default ticket management behavior
        result = await this.callTicketingFlow({...});
    }
    break;
```

## Flow Diagram

```
User: "Show me my tickets"
        │
        ▼
┌─────────────────────────┐
│  MAIN ROUTER            │
│  Route: IT_TICKET_MGMT  │
└──────────┬──────────────┘
           │
           ▼
┌─────────────────────────────┐
│  SUB_ROUTING Ticket Mgmt    │
│  Action: GET_ALL_TICKETS    │
└──────────┬──────────────────┘
           │
           ▼
┌─────────────────────────────────────┐
│  LangFlowHandler.processRoutingResult │
│  → ticketingFlow.handleGetOpenTickets() │
└──────────┬──────────────────────────┘
           │
           ├─── Phase 1: Generate acknowledgement
           │    "Let me fetch your open tickets..."
           │
           ├─── Phase 2: Call IncidentService.getIncidents()
           │    Filter: callerEmail, state='open', limit=10
           │
           └─── Phase 3: Generate response + Adaptive Card
                │
                ▼
┌─────────────────────────────────────┐
│  Bot sends proactive message        │
│  with Adaptive Card (if tickets)    │
└─────────────────────────────────────┘
```

## Known Limitations

1. **LangFlow Flow Not Yet Created**: The implementation currently uses hardcoded messages. A LangFlow GET_OPEN_TICKETS flow can be created later for dynamic message generation.

2. **No Two-Phase Proactive Messaging Yet**: The current implementation returns both acknowledgement and response together. Future enhancement could send acknowledgement first, then fetch tickets and send response separately.

3. **Ticket Limit**: Currently limited to 10 tickets. Could add pagination in future.

## Testing Notes

### Test Scenarios

1. **User with open tickets**:
   - Expected: Acknowledgement + Adaptive Card with ticket list

2. **User with no open tickets**:
   - Expected: Message "I couldn't find any open tickets"

3. **ITSM service unavailable**:
   - Expected: Error message from _getResponseMessage

### Test Commands

```bash
# Run the bot locally
cd botframework
pnpm dev

# Test in Teams or DirectLine with messages like:
# - "Show me my tickets"
# - "What are my open tickets?"
# - "List my incidents"
```

## Future Enhancements

1. **LangFlow Integration**: Create GET_OPEN_TICKETS flow in LangFlow for dynamic messages
2. **Pagination**: Add "Load more" button for users with many tickets
3. **Filtering**: Allow filtering by priority, date range, etc.
4. **Quick Actions**: Add "Add Comment" or "Close Ticket" actions directly from card
5. **Real-time Updates**: WebSocket integration for ticket status changes

## Dependencies

- `IncidentService` - ITSM integration layer (existing)
- `MSTeamsService` - Proactive messaging (existing)
- `ConversationRoutingService` - Route lock management (existing)
- `SUB_ROUTING - Ticket Management` LangFlow flow (existing)

## Related Tickets

- CB-012: ITSM Integration APIs
- CB-016: Conversation Routing
- CB-013: Chat Transcript Capture (for transcript attachment in ticket creation)
