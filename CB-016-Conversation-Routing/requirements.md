# CB-016: Conversation Routing Refactor with Route Locking

> **Ticket**: CB-016-Conversation-Routing
> **Type**: Feature Enhancement
> **Priority**: High
> **Status**: Planning
> **Created**: November 2024

---

## Overview

Refactor the conversation routing system to use the local Classification API (CB-014/CB-015) instead of LangFlow Router, and implement route locking for multi-turn conversation flows. This enables better control over conversation lifecycle and context management.

---

## Problem Statement

Currently, the system has the following limitations:

1. **External Dependency**: Routing relies on LangFlow Router flow, adding network latency and external service dependency
2. **No Route Context**: Each turn is classified independently, losing context in multi-turn flows (e.g., ticket management)
3. **No User Control**: Users cannot easily switch topics or cancel mid-flow
4. **Timeout Handling**: Abandoned conversations leave stale route context

---

## Use Cases

### UC-1: Single-Turn Conversation (No Lock)
**Scenario**: User asks a simple question
- User: "What's the weather today?"
- Bot classifies → CHIT_CHAT → Responds
- **Expected**: No route lock, next turn uses fresh classification

### UC-2: Multi-Turn Flow Entry (Lock)
**Scenario**: User starts a ticket management flow
- User: "Check my ticket"
- Bot classifies → IT_TICKET_MGMT → Locks route
- User: "Show me INC123456"
- **Expected**: Stays in IT_TICKET_MGMT (locked), no re-classification

### UC-3: Multi-Turn Flow Continuation (Locked)
**Scenario**: User continues in locked flow
- User is in IT_TICKET_MGMT (locked)
- User: "Escalate it"
- **Expected**: Stays in IT_TICKET_MGMT, processes escalation

### UC-4: Flow Completion (Release)
**Scenario**: User completes a multi-turn task
- User is in IT_TICKET_MGMT (locked)
- Bot: "Ticket escalated. Is there anything else?"
- User: "No, that's all"
- **Expected**: Route released, next turn uses fresh classification

### UC-5: Topic Switch Request (User Control)
**Scenario**: User wants to change topic mid-flow
- User is in IT_TICKET_MGMT (locked)
- User: "What's the weather?"
- Bot: "You're working on a ticket. Switch to new topic or continue?"
- User: "Switch"
- **Expected**: Route released, classifies new message

### UC-6: User Cancellation (Restart Command)
**Scenario**: User wants to cancel and start fresh
- User is in any locked flow
- User: "restart"
- **Expected**: Route released immediately, session cleared

### UC-7: Timeout Recovery
**Scenario**: User abandons conversation
- User is in IT_TICKET_MGMT (locked)
- User leaves for 30+ minutes
- User returns: "Hello"
- **Expected**: Lock expired, fresh classification, notify user session expired

---

## Functional Requirements

### FR-1: Local Classification Integration

Replace LangFlow Router with local Classification API:

| Method | Description |
|--------|-------------|
| Direct Call | Import `classifyRoute()` from IntentClassificationService |
| HTTP Call | POST to `/api/classification/route` (configurable) |

Configuration toggle:
```javascript
classification: {
    useLocalClassification: true,  // true = local, false = LangFlow
    useHttpClient: false,          // true = HTTP, false = direct call
}
```

### FR-2: Route Lock State Machine

Implement route lock states:

```
STATES:
├── UNLOCKED        - Fresh routing needed, use Classification API
├── LOCKED          - Multi-turn flow active, skip classification
├── AWAITING_SWITCH - Topic change detected, waiting for user choice
└── PENDING_RELEASE - Task done, will release on next turn
```

State transitions:
```
UNLOCKED → LOCKED           (Enter multi-turn flow)
LOCKED → LOCKED             (Continue in flow)
LOCKED → AWAITING_SWITCH    (Different topic detected)
LOCKED → PENDING_RELEASE    (Task completed)
LOCKED → UNLOCKED           (User cancels/restart)
AWAITING_SWITCH → LOCKED    (User continues)
AWAITING_SWITCH → UNLOCKED  (User switches)
PENDING_RELEASE → UNLOCKED  (User says no more help)
PENDING_RELEASE → LOCKED    (User has follow-up)
```

### FR-3: Route Lock Rules

| Route | Should Lock? | Release Condition |
|-------|--------------|-------------------|
| IT_TICKET_MGMT | YES | After ticket action completed |
| IT_HELP_DESK | YES | After handoff or user cancels |
| IT_KB_SEARCH | NO | Single-turn query |
| MED_KB_SEARCH | NO | Single-turn query |
| MED_DRUG_SEARCH | NO | Single-turn query |
| CHIT_CHAT | NO | Single-turn response |
| OTHER | NO | Single-turn response |

### FR-4: User Control Commands

| Command | Action | State Transition |
|---------|--------|------------------|
| `restart` | Release lock, clear all context | Any → UNLOCKED |
| `cancel` | Release lock, keep chat context | LOCKED → UNLOCKED |
| `back` | Go back one step in current flow | Stay LOCKED |
| `status` | Show current progress | No change |

### FR-5: Topic Switch Confirmation

When locked and different topic detected:
1. Detect high-confidence different intent (>0.7)
2. Set state to AWAITING_SWITCH
3. Ask: "You're working on [current task]. Continue or switch to new topic?"
4. Handle response: continue (stay LOCKED) or switch (release to UNLOCKED)

### FR-6: Timeout Handling

| Scenario | Timeout | Action |
|----------|---------|--------|
| Lock expires | 30 minutes | Auto-release, notify on return |
| User returns before timeout | < 30 min | Offer to resume or start fresh |
| User returns after timeout | > 30 min | Fresh start with notification |

### FR-7: Route Context Storage

Store in Azure Table Storage (UserService):

```javascript
{
    // Route Lock State
    route_lock_state: 'UNLOCKED' | 'LOCKED' | 'AWAITING_SWITCH' | 'PENDING_RELEASE',
    locked_route: 'IT_TICKET_MGMT',  // null when UNLOCKED
    locked_at: '2024-11-30T10:00:00Z',
    lock_expires_at: '2024-11-30T10:30:00Z',

    // Route Context (JSON string)
    route_context: JSON.stringify({
        selectedTicketId: 'INC0012345',
        ticketAction: 'escalate',
        currentStep: 'confirm_escalation',
        completedSteps: ['list_tickets', 'select_ticket'],
        turnCount: 3,
        pendingSwitchTo: null
    })
}
```

---

## Non-Functional Requirements

### NFR-1: Performance
- Route lock check: < 50ms
- Classification (local): < 500ms
- No regression in message processing latency

### NFR-2: Reliability
- Route lock state persisted to Azure Table Storage
- Recovery from server restarts
- Graceful degradation if Classification API fails

### NFR-3: Configurability
- Toggle between local Classification and LangFlow Router
- Toggle between HTTP and direct call for Classification
- Configurable lock timeout (default: 30 minutes)

### NFR-4: Observability
- Log all route lock state transitions
- Track lock timeout occurrences
- Monitor classification latency

---

## Acceptance Criteria

### AC-1: Local Classification
- [ ] Bot uses local Classification API by default
- [ ] Can toggle to LangFlow Router via config
- [ ] Classification accuracy matches or exceeds LangFlow

### AC-2: Route Locking
- [ ] IT_TICKET_MGMT locks on entry
- [ ] Locked routes skip re-classification
- [ ] Lock released on task completion

### AC-3: User Control
- [ ] `restart` command releases lock and clears session
- [ ] Topic switch shows confirmation prompt
- [ ] User can continue or switch

### AC-4: Timeout
- [ ] Lock expires after 30 minutes
- [ ] User notified on return after timeout
- [ ] Fresh classification on expired lock

### AC-5: Persistence
- [ ] Route lock state saved to Azure Table
- [ ] Survives server restart
- [ ] Route context preserved in lock

---

## Dependencies

- CB-014: Zero-Shot Classification API
- CB-015: Intent Classification Quality (Ensemble)
- CB-006: Conversation Lifecycle Service
- UserService (Azure Table Storage)
- IntentClassificationService

---

## References

- [CB-014-Zero-Shot-Classification](../CB-014-Zero-Shot-Classification/)
- [CB-015-Intent-Classification-Quality](../CB-015-Intent-Classification-Quality/)
- [CB-006-Conversation-Lifecycle](../CB-006-Conversation-Lifecycle/)
- [ARCHITECTURE.md](../ARCHITECTURE.md)
