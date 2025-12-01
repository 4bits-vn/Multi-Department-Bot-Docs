# CB-019: Simplified Ticket Management Flow

## Overview

Simplify the ticket management flow by having BotFramework handle all UI rendering (Adaptive Cards) while LangFlow IT_TICKET_MGMT flow only generates acknowledgment and response messages.

## Current Behavior

Currently, the TicketingFlow.js handles both:
- Calling LangFlow IT_TICKET_MGMT for message generation
- Building Adaptive Cards for ticket display

This creates complexity and tight coupling between LangFlow and UI rendering.

## Target Behavior

### LangFlow IT_TICKET_MGMT Flow Responsibilities
- Generate acknowledgment messages (e.g., "Let me fetch your open tickets...")
- Generate response messages based on results (e.g., "I found 2 open tickets...")
- **NOT** responsible for rendering ticket information or offering ticket actions

### BotFramework Responsibilities
- Render ticket information using Adaptive Cards
- Offer ticket action buttons via Adaptive Cards
- Combine LangFlow messages with ticket data for display
- Handle Adaptive Card submit actions (CANCEL, ESCALATE, ADD_COMMENT, GET_ALL_TICKETS)

## User Flow Example

### Show My Tickets Flow

1. **User**: "show my tickets"

2. **Bot Acknowledgment**: "Let me fetch your open tickets. One moment please... 🔍"
   - Priority: Hard-coded message
   - Alternative: Use LangFlow IT_TICKET_MGMT for generating message

3. **Bot**: Calls ITSM API to get user's tickets

4. **Bot Response**: "I found 2 open tickets in your account."
   - Priority: Hard-coded message based on ITSM result
   - Alternative: Use LangFlow IT_TICKET_MGMT with result context

5. **Bot**: Displays Adaptive Card with:
   - Response message
   - Dropdown list of open tickets for user selection

6. **User**: Selects a ticket and clicks "View Details"

7. **Bot**: Calls ITSM API to get ticket details

8. **Bot**: Displays ticket detail Adaptive Card with:
   - Message: "Here is your ticket:"
   - Ticket information (number, description, status, priority, dates)
   - Action buttons:
     1. **Open in ServiceNow** - Link to ITSM system
     2. **Cancel Ticket** - Action.Submit → `CANCEL`
     3. **Escalate Ticket** - Action.Submit → `ESCALATE`
     4. **Add Comment** - Action.Submit → `ADD_COMMENT`
     5. **Show Another Ticket** - Action.Submit → `GET_ALL_TICKETS`

### Action Handling

When user clicks an action button:

| Button | Action Data | Handler |
|--------|-------------|---------|
| Open in ServiceNow | `Action.OpenUrl` | Opens ITSM URL directly |
| Cancel Ticket | `{ action: "CANCEL", ticketNumber: "INC..." }` | bot.controller handles, calls IncidentService.cancelIncident |
| Escalate Ticket | `{ action: "ESCALATE", ticketNumber: "INC..." }` | bot.controller handles, calls IncidentService.escalateIncident |
| Add Comment | `{ action: "ADD_COMMENT", ticketNumber: "INC..." }` | bot.controller handles, shows comment input card |
| Show Another Ticket | `{ action: "GET_ALL_TICKETS" }` | bot.controller handles, loops back to ticket list |

## Functional Requirements

### FR-1: Ticket List Display
- Display dropdown with user's open tickets
- Show ticket number, description, and status icon
- Include "View Details" submit action

### FR-2: Ticket Detail Display
- Show comprehensive ticket information
- Display action buttons for ticket management
- Include link to open ticket in ITSM system

### FR-3: Ticket Actions
- Cancel ticket with confirmation
- Escalate ticket with optional reason
- Add comment with text input
- Navigate back to ticket list

### FR-4: Message Generation
- Use hard-coded messages as primary
- LangFlow IT_TICKET_MGMT as fallback for personalized messages
- Messages should be contextual based on results

## Non-Functional Requirements

### NFR-1: Performance
- Card rendering should be fast (< 500ms)
- ITSM API calls should have appropriate timeout (30s)

### NFR-2: User Experience
- Clear visual feedback during operations
- Consistent card styling
- Intuitive action buttons

### NFR-3: Error Handling
- Graceful error messages for failed operations
- Retry logic for transient failures

## Acceptance Criteria

- [ ] User can view list of open tickets via Adaptive Card dropdown
- [ ] User can select a ticket and view its details
- [ ] User can cancel a ticket via action button
- [ ] User can escalate a ticket via action button
- [ ] User can add a comment to a ticket
- [ ] User can navigate back to ticket list from ticket detail
- [ ] All actions provide appropriate feedback messages
- [ ] Cards render correctly in MS Teams

## Technical Dependencies

- IncidentService for ITSM API calls
- MSTeamsService for sending Adaptive Cards
- ConversationRoutingService for route management
- LangFlow IT_TICKET_MGMT flow (optional, for message generation)

## Related Documentation

- [CB-012-ITSM-Integration-APIs](../CB-012-ITSM-Integration-APIs/)
- [CB-016-Conversation-Routing](../CB-016-Conversation-Routing/)
- [CB-018-Get-Open-Tickets](current TicketingFlow implementation)
