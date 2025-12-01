# CB-019: Implementation Plan - Simplified Ticket Management Flow

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          USER MESSAGE                                   │
│                     "show my tickets"                                   │
└─────────────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                      ROUTER (Classification)                            │
│                     Route: IT_TICKET_MGMT                               │
│                     Action: GET_ALL_TICKETS                             │
└─────────────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    TICKETING FLOW HANDLER                               │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  1. Get acknowledgment message (hard-coded OR LangFlow)         │   │
│  │  2. Send acknowledgment to user                                 │   │
│  │  3. Call ITSM API (IncidentService.getIncidents)                │   │
│  │  4. Get response message based on results                       │   │
│  │  5. Build Adaptive Card (TicketCardHandler)                     │   │
│  │  6. Return card + message                                        │   │
│  └─────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    BOT CONTROLLER                                       │
│                Sends Adaptive Card to MS Teams                          │
└─────────────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    USER INTERACTION                                     │
│              Selects ticket / Clicks action button                      │
└─────────────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    ACTION HANDLER (in bot.controller)                   │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  VIEW_TICKET_DETAILS → Show ticket detail card                  │   │
│  │  CANCEL              → Cancel ticket, show confirmation         │   │
│  │  ESCALATE            → Escalate ticket, show confirmation       │   │
│  │  ADD_COMMENT         → Show comment input, submit comment       │   │
│  │  GET_ALL_TICKETS     → Loop back to ticket list                 │   │
│  └─────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────┘
```

## Implementation Steps

### Step 1: Create TicketCardHandler.js

Create a dedicated handler for building ticket-related Adaptive Cards.

**File**: `src/Handlers/TicketCardHandler.js`

**Methods**:
- `buildTicketListCard(tickets, message)` - Dropdown card for ticket selection
- `buildTicketDetailCard(ticket, message)` - Detailed ticket card with actions
- `buildCommentInputCard(ticketNumber)` - Card for adding comments
- `buildConfirmationCard(action, ticketNumber, result)` - Action result confirmation

### Step 2: Update TicketingFlow.js

Simplify to focus on message generation only.

**Changes**:
- Remove `_buildTicketDropdownCard` method
- Remove `_buildTicketSummaryItem` method
- Keep message generation methods (acknowledgment, response)
- Delegate card building to TicketCardHandler

### Step 3: Create Ticket Action Types

**File**: `src/Handlers/LangFlow/config.js`

Add new constants for Adaptive Card actions:
```javascript
const TICKET_CARD_ACTIONS = Object.freeze({
    VIEW_TICKET_DETAILS: "VIEW_TICKET_DETAILS",
    CANCEL: "CANCEL",
    ESCALATE: "ESCALATE",
    ADD_COMMENT: "ADD_COMMENT",
    SUBMIT_COMMENT: "SUBMIT_COMMENT",
    GET_ALL_TICKETS: "GET_ALL_TICKETS",
});
```

### Step 4: Update bot.controller.js

Add handler for Adaptive Card submit actions.

**New Function**: `handleTicketCardAction(activity)`

Handles:
- `VIEW_TICKET_DETAILS` - Fetch ticket details and display detail card
- `CANCEL` - Call IncidentService.cancelIncident
- `ESCALATE` - Call IncidentService.escalateIncident
- `ADD_COMMENT` - Show comment input or submit comment
- `GET_ALL_TICKETS` - Loop back to show ticket list

### Step 5: Update handleTeamsMessage

Detect Adaptive Card submit actions (value field) and route to action handler.

```javascript
// In handleTeamsMessage
if (activity.value && activity.value.action) {
    await handleTicketCardAction(activity);
    return;
}
```

## File Changes Summary

| File | Change Type | Description |
|------|-------------|-------------|
| `src/Handlers/TicketCardHandler.js` | **NEW** | Dedicated Adaptive Card handler for tickets |
| `src/Handlers/LangFlow/flows/TicketingFlow.js` | **MODIFY** | Remove card building, focus on messages |
| `src/Handlers/LangFlow/config.js` | **MODIFY** | Add TICKET_CARD_ACTIONS constants |
| `src/controllers/bot.controller.js` | **MODIFY** | Add ticket action handler |

## Adaptive Card Designs

### 1. Ticket List Card

```json
{
  "type": "AdaptiveCard",
  "body": [
    {
      "type": "TextBlock",
      "text": "📋 Your Open Tickets (2)",
      "weight": "Bolder"
    },
    {
      "type": "TextBlock",
      "text": "I found 2 open tickets in your account. Please select one:"
    },
    {
      "type": "Input.ChoiceSet",
      "id": "selectedTicket",
      "choices": [
        { "title": "🆕 INC001234 - VPN not working", "value": "INC001234" },
        { "title": "🔄 INC001235 - Outlook crashes", "value": "INC001235" }
      ]
    }
  ],
  "actions": [
    {
      "type": "Action.Submit",
      "title": "View Ticket Details",
      "data": { "action": "VIEW_TICKET_DETAILS" }
    }
  ]
}
```

### 2. Ticket Detail Card

```json
{
  "type": "AdaptiveCard",
  "body": [
    {
      "type": "Container",
      "style": "emphasis",
      "items": [
        { "type": "TextBlock", "text": "🎫 INC001234", "weight": "Bolder" },
        { "type": "TextBlock", "text": "VPN not working" }
      ]
    },
    {
      "type": "FactSet",
      "facts": [
        { "title": "Status", "value": "🔄 In Progress" },
        { "title": "Priority", "value": "High" },
        { "title": "Created", "value": "Nov 28, 2025" },
        { "title": "Updated", "value": "Nov 30, 2025" }
      ]
    },
    {
      "type": "TextBlock",
      "text": "Description: Unable to connect to VPN..."
    }
  ],
  "actions": [
    {
      "type": "Action.OpenUrl",
      "title": "🔗 Open in ServiceNow",
      "url": "https://servicenow.example.com/incident/INC001234"
    },
    {
      "type": "Action.Submit",
      "title": "❌ Cancel Ticket",
      "data": { "action": "CANCEL", "ticketNumber": "INC001234" }
    },
    {
      "type": "Action.Submit",
      "title": "⬆️ Escalate",
      "data": { "action": "ESCALATE", "ticketNumber": "INC001234" }
    },
    {
      "type": "Action.Submit",
      "title": "💬 Add Comment",
      "data": { "action": "ADD_COMMENT", "ticketNumber": "INC001234" }
    },
    {
      "type": "Action.Submit",
      "title": "📋 Show Another Ticket",
      "data": { "action": "GET_ALL_TICKETS" }
    }
  ]
}
```

### 3. Comment Input Card

```json
{
  "type": "AdaptiveCard",
  "body": [
    {
      "type": "TextBlock",
      "text": "💬 Add Comment to INC001234"
    },
    {
      "type": "Input.Text",
      "id": "commentText",
      "placeholder": "Enter your comment...",
      "isMultiline": true
    }
  ],
  "actions": [
    {
      "type": "Action.Submit",
      "title": "Submit Comment",
      "data": { "action": "SUBMIT_COMMENT", "ticketNumber": "INC001234" }
    },
    {
      "type": "Action.Submit",
      "title": "Cancel",
      "data": { "action": "VIEW_TICKET_DETAILS", "ticketNumber": "INC001234" }
    }
  ]
}
```

## Message Templates

### Hard-coded Messages (Primary)

```javascript
const TICKET_MESSAGES = {
    // Acknowledgments
    ACK_GET_ALL_TICKETS: "Let me fetch your open tickets. One moment please... 🔍",
    ACK_GET_TICKET_DETAIL: "Fetching ticket details... 📋",
    ACK_CANCEL: "Processing your cancellation request... ⏳",
    ACK_ESCALATE: "Escalating your ticket... ⬆️",
    ACK_ADD_COMMENT: "Adding your comment... 💬",

    // Responses
    RESP_TICKETS_FOUND: (count) => `I found ${count} open ticket${count !== 1 ? 's' : ''} in your account.`,
    RESP_NO_TICKETS: "Good news! 🎉 You don't have any open tickets.",
    RESP_TICKET_CANCELLED: (number) => `Ticket ${number} has been cancelled. ✅`,
    RESP_TICKET_ESCALATED: (number) => `Ticket ${number} has been escalated. ⬆️`,
    RESP_COMMENT_ADDED: (number) => `Comment added to ticket ${number}. 💬`,
    RESP_ERROR: "I encountered an issue. Please try again.",
};
```

## Testing Checklist

- [ ] Show tickets displays dropdown card correctly
- [ ] Selecting ticket and clicking View shows detail card
- [ ] Cancel button cancels ticket and shows confirmation
- [ ] Escalate button escalates ticket and shows confirmation
- [ ] Add Comment shows input card and submits comment
- [ ] Show Another Ticket returns to ticket list
- [ ] Open in ServiceNow opens correct URL
- [ ] Error cases show appropriate messages
- [ ] Cards render correctly in MS Teams
