# CB-007: Dynamic Lifecycle Messages

## Ticket Information

- **Ticket ID**: CB-007
- **Type**: Enhancement
- **Priority**: Medium
- **Status**: Completed

## Overview

Enhancement to the Conversation Lifecycle Management system to replace hardcoded lifecycle messages with dynamically generated messages via LangFlow. This provides more natural, context-aware, and varied responses.

## Problem Statement

The previous implementation used static, hardcoded messages for lifecycle events (greetings, goodbyes, error messages, etc.). This approach had limitations:

1. **Repetitive responses**: Same message every time
2. **No personalization**: Couldn't adapt to user context
3. **Maintenance burden**: Changes required code updates
4. **Internationalization**: Hard to support multiple languages

## Solution

Route lifecycle messages through LangFlow flows:

- **CHITCHAT flow**: For conversational messages (greetings, follow-ups, goodbyes)
- **EXCEPTION flow**: For error messages (system errors, validation failures)

Static messages are kept as fallbacks when LangFlow is unavailable.

## Functional Requirements

### FR-1: Message Type Classification

- All lifecycle messages must be classified into categories:
  - `CONVERSATIONAL`: Routed to CHITCHAT flow
  - `ERROR`: Routed to EXCEPTION flow

### FR-2: Dynamic Message Generation

- System must call appropriate LangFlow flow for each message type
- Must provide context to the flow for better responses:
  - User name (if available)
  - Error details (for error messages)
  - Session context

### FR-3: Fallback Mechanism

- If LangFlow is unavailable, system must use static fallback messages
- If flow call fails, system must gracefully fallback
- Fallback must be transparent to users

### FR-4: Configuration

- Dynamic messages can be disabled via environment variable
- Timeout for message generation must be configurable
- Optional caching of generated messages

## Non-Functional Requirements

### NFR-1: Performance

- Message generation timeout: 5 seconds (configurable)
- Should not significantly impact response time

### NFR-2: Reliability

- Must never fail to provide a message (fallback guaranteed)
- Must handle LangFlow outages gracefully

### NFR-3: Logging

- Must log whether dynamic or fallback message was used
- Must log any errors in message generation

## Message Types

### Conversational Messages (CHITCHAT flow)

| Type | Description |
|------|-------------|
| `NEED_MORE_HELP` | Ask if user needs more assistance |
| `GOODBYE` | End of conversation farewell |
| `HOW_CAN_I_HELP` | User wants more help |
| `RESTART_CONFIRMED` | Session restart confirmation |
| `SESSION_EXPIRED` | Timeout notification |
| `TIMEOUT_WARNING` | Inactivity warning |
| `ERROR_RECOVERY` | Recovery after error |
| `STILL_PROCESSING` | Request is being processed |
| `TAKING_LONGER` | Long processing notification |
| `WELCOME` | New user greeting |
| `WELCOME_BACK` | Returning user greeting |
| `COMMANDS_HINT` | Available commands tip |

### Error Messages (EXCEPTION flow)

| Type | Description |
|------|-------------|
| `GENERIC_ERROR` | General error message |
| `MAX_ITERATIONS_ERROR` | Processing limit reached |
| `PROCESSING_TIMEOUT` | Request timeout |
| `RATE_LIMIT_ERROR` | Too many requests |
| `USER_NOT_FOUND` | User verification failed |
| `USER_INACTIVE` | Inactive account |

## Acceptance Criteria

- [x] All lifecycle messages use LifecycleMessageType constants
- [x] CHITCHAT flow receives appropriate prompts for conversational messages
- [x] EXCEPTION flow receives appropriate prompts for error messages
- [x] Static fallback messages are used when flows unavailable
- [x] No hardcoded messages remain in bot.controller.js
- [x] Environment variable can disable dynamic messages
- [x] Documentation updated

## Technical Approach

### Architecture

```
┌─────────────────────┐
│  bot.controller.js  │
└─────────┬───────────┘
          │
          ▼
┌─────────────────────────────┐
│ LifecycleMessageService     │
│ - getMessage(type, options) │
│ - getResolutionWithFollowUp │
│ - getErrorMessage           │
└─────────┬───────────────────┘
          │
          ▼
┌─────────────────────────────┐
│ LangFlowHandler             │
│ - generateLifecycleMessage  │
│ - generateConversational    │
│ - generateErrorMessage      │
└──────────┬─────────┬────────┘
           │         │
           ▼         ▼
      ┌────────┐ ┌──────────┐
      │CHITCHAT│ │EXCEPTION │
      │  Flow  │ │   Flow   │
      └────────┘ └──────────┘
```

### Flow Prompts

For CHITCHAT flow, prompts are designed to generate appropriate responses:

```
// GOODBYE example prompt
"The user has finished their conversation and needs no more help.
Say goodbye professionally and warmly. Mention they can return anytime."
```

For EXCEPTION flow, structured input is provided:

```json
{
  "situation": "SYSTEM_ERROR",
  "description": "The system reached a processing limit while handling this request"
}
```

## Files Changed

### New Files

- `botframework/src/Services/LifecycleMessageService.js` - Orchestrates message generation

### Modified Files

- `botframework/src/utils/lifecycleMessages.js` - Added message types, prompts, and fallbacks
- `botframework/src/Handlers/LangFlowHandler.js` - Added generateLifecycleMessage method
- `botframework/src/Services/ConversationLifecycleService.js` - Uses LifecycleMessageService
- `botframework/src/controllers/bot.controller.js` - Uses dynamic messages

## Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `ENABLE_DYNAMIC_LIFECYCLE_MESSAGES` | `true` | Enable/disable dynamic messages |
| `LIFECYCLE_MESSAGE_TIMEOUT` | `5000` | Timeout for message generation (ms) |
| `LIFECYCLE_MESSAGE_CACHE_TTL` | `0` | Cache TTL (0 = disabled) |

## Related Documentation

- [CB-006 Conversation Lifecycle](../CB-006-Conversation-Lifecycle/requirements.md)
- [CHITCHAT Flow](../../flows/CHITCHAT/README.md)
- [EXCEPTION Flow](../../flows/EXCEPTION/README.md)
