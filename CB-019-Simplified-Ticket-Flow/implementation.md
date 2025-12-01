# CB-019: Implementation - Simplified Ticket Management Flow

## Summary

Simplified the ticket management flow by separating concerns:
- **TicketCardHandler.js**: Handles all Adaptive Card building
- **TicketingFlow.js**: Handles message generation and ITSM operations
- **bot.controller.js**: Handles Adaptive Card submit actions

## Files Created

### `src/Handlers/TicketCardHandler.js`

New dedicated handler for ticket-related Adaptive Cards.

**Exports**:
- `buildTicketListCard({ tickets, message })` - Dropdown card for ticket selection
- `buildTicketDetailCard({ ticket, message })` - Detailed ticket card with actions
- `buildCommentInputCard({ ticketNumber })` - Card for adding comments
- `buildActionResultCard({ action, ticketNumber, success, message })` - Action result confirmation
- `buildNoTicketsCard(message)` - Empty state card
- `TICKET_CARD_ACTIONS` - Constants for card actions
- `TICKET_MESSAGES` - Hard-coded message templates

## Files Modified

### `src/Handlers/LangFlow/flows/TicketingFlow.js`

**Removed**:
- `handleGetOpenTickets()` - Complex multi-phase flow
- `_buildTicketDropdownCard()` - Card building
- `_buildTicketSummaryItem()` - Card building helper
- `_callITAgentForAcknowledgement()` - LangFlow messaging
- `_callITAgentForResponse()` - LangFlow messaging
- `execute()` method for LangFlow calls

**Simplified to**:
- `getAcknowledgmentMessage(action)` - Get hard-coded ack message
- `getResponseMessage(action, result)` - Get hard-coded response message
- `fetchUserTickets(userEmail)` - Fetch from ITSM
- `fetchTicketDetails(ticketNumber)` - Fetch single ticket
- `cancelTicket(ticketNumber, reason)` - Cancel ticket
- `escalateTicket(ticketNumber, reason)` - Escalate ticket
- `addComment(ticketNumber, comment)` - Add comment
- `generateMessage(params)` - LangFlow fallback (optional)

### `src/Handlers/LangFlow/index.js`

**Changes**:
- Simplified `IT_TICKET_MGMT` case in `processRoutingResult()`
- Added `_handleTicketManagement()` private method
- Simplified `_detectTicketIntent()` method
- Added imports for `TicketCardHandler`

### `src/Handlers/LangFlow/config.js`

**Added**:
- `TICKET_CARD_ACTIONS` constant for Adaptive Card actions
- `SUBMIT_COMMENT` to `TICKET_ACTIONS`

### `src/controllers/bot.controller.js`

**Added**:
- Import for `TicketCardHandler`
- Import for `ticketingFlow`
- Check for Adaptive Card submit at start of `handleTeamsMessage()`
- `handleTicketCardAction(activity, cardData)` function
- Handling for `subFlowResult.attachments` in router response

### `src/Services/MSTeamsService.js`

**Added**:
- `sendAdaptiveCard({ serviceUrl, conversationId, card, message })` convenience method

## Flow Diagram

```
┌──────────────────────────────────────────────────────────────────┐
│                    USER: "show my tickets"                        │
└──────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌──────────────────────────────────────────────────────────────────┐
│                  ROUTER → IT_TICKET_MGMT                          │
└──────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌──────────────────────────────────────────────────────────────────┐
│               _handleTicketManagement()                           │
│  1. Classify action → GET_ALL_TICKETS                            │
│  2. ticketingFlow.fetchUserTickets()                             │
│  3. buildTicketListCard()                                        │
│  4. Return { text, attachments }                                 │
└──────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌──────────────────────────────────────────────────────────────────┐
│               bot.controller.js                                   │
│  Send Adaptive Card via msTeamsService.sendProactiveMessage()    │
└──────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌──────────────────────────────────────────────────────────────────┐
│               USER: Clicks "View Ticket Details"                  │
└──────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌──────────────────────────────────────────────────────────────────┐
│               handleTicketCardAction()                            │
│  action: VIEW_TICKET_DETAILS                                     │
│  1. ticketingFlow.fetchTicketDetails()                           │
│  2. buildTicketDetailCard()                                      │
│  3. msTeamsService.sendAdaptiveCard()                            │
└──────────────────────────────────────────────────────────────────┘
```

## Adaptive Card Actions

| Button | Action | Handler |
|--------|--------|---------|
| View Ticket Details | `VIEW_TICKET_DETAILS` | `handleTicketCardAction` |
| Cancel Ticket | `CANCEL` | `handleTicketCardAction` → `ticketingFlow.cancelTicket()` |
| Escalate | `ESCALATE` | `handleTicketCardAction` → `ticketingFlow.escalateTicket()` |
| Add Comment | `ADD_COMMENT` | `handleTicketCardAction` → Show comment input card |
| Submit Comment | `SUBMIT_COMMENT` | `handleTicketCardAction` → `ticketingFlow.addComment()` |
| Show Another Ticket | `GET_ALL_TICKETS` | `handleTicketCardAction` → Loop back to list |
| Open in ServiceNow | `Action.OpenUrl` | Opens URL directly |

## Message Templates

Hard-coded messages are used as primary, with LangFlow IT_TICKET_MGMT as optional fallback.

```javascript
const TICKET_MESSAGES = {
    ACK_GET_ALL_TICKETS: "Let me fetch your open tickets. One moment please... 🔍",
    ACK_GET_TICKET_DETAIL: "Fetching ticket details... 📋",
    ACK_CANCEL: "Processing your cancellation request... ⏳",
    ACK_ESCALATE: "Escalating your ticket... ⬆️",
    ACK_ADD_COMMENT: "Adding your comment... 💬",

    ticketsFound: (count) => `I found ${count} open ticket${count !== 1 ? 's' : ''} in your account.`,
    noTickets: "Good news! 🎉 You don't have any open tickets.",
    ticketCancelled: (number) => `Ticket ${number} has been cancelled successfully. ✅`,
    ticketEscalated: (number) => `Ticket ${number} has been escalated to the support team. ⬆️`,
    commentAdded: (number) => `Your comment has been added to ticket ${number}. 💬`,
    error: "I encountered an issue processing your request. Please try again.",
};
```

## Benefits of This Refactor

1. **Separation of Concerns**: Card building is isolated in `TicketCardHandler.js`
2. **Cleaner Code**: `TicketingFlow.js` reduced from ~800 lines to ~200 lines
3. **Testability**: Each component can be tested independently
4. **Maintainability**: Easy to update card designs without touching flow logic
5. **Consistency**: All ticket cards use the same styles and patterns
6. **Flexibility**: Easy to add new card types or actions
7. **Accurate Transcripts**: Card displays and actions recorded with descriptive content
8. **Context Continuity**: Conversation summaries updated with ticket actions

## Transcript & Summary Best Practices

### User Actions (Card Interactions)
User card actions are recorded with descriptive text:
```
[Card Action] User requested to view all open tickets.
[Card Action] User selected to view details for ticket INC0012345.
[Card Action] User requested to escalate ticket INC0012345.
```

### Bot Responses (Card Displays)
Card displays are recorded with full content descriptions:
```
I found 2 open tickets in your account. Tickets displayed:
INC0012345: "VPN not working" (In Progress);
INC0012346: "Outlook crashes" (New).
User can select a ticket to view details.
```

### Summary Updates
Conversation summaries are updated after ticket actions:
- `User requested to view tickets. Found 2 open tickets.`
- `User viewing ticket INC0012345 (In Progress).`
- `User escalated ticket INC0012345.`
- `User added comment to ticket INC0012345.`

### Implementation Details

1. **`handleTicketCardAction()`** records:
   - User action description (via `_buildUserActionDescription()`)
   - Bot response with descriptive card content
   - Summary updates after successful actions

2. **`_handleTicketManagement()`** returns:
   - `text`: Full descriptive content for transcript
   - `displayText`: Short message for user display
   - `summaryContext`: Context for summary update
   - `ticketAction`: Action performed
   - `ticketNumber`: Ticket involved (if applicable)

## Testing

To test the flow:

1. Send "show my tickets" → Should show ticket list card
2. Select a ticket and click "View Details" → Should show detail card
3. Click "Cancel Ticket" → Should cancel and show confirmation
4. Click "Escalate" → Should escalate and show confirmation
5. Click "Add Comment" → Should show comment input
6. Submit comment → Should add comment and show confirmation
7. Click "Show Another Ticket" → Should loop back to list

### Verify Transcript Accuracy

After testing, check the transcript in Azure Table Storage (`BotTranscripts`) to verify:
- User actions are recorded with `[Card Action]` prefix
- Bot responses include full card content descriptions
- Metadata includes `isCardInteraction: true` for card actions
- Metadata includes `hasCard: true` for card responses

### Verify Summary Updates

Check user state in Azure Table Storage (`BotUsers`) to verify:
- `conversation_summary` includes ticket context
- `last_ticket_action` tracks the last action
- `last_ticket_number` tracks the last ticket involved
