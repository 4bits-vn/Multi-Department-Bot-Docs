# CB-005: Conversation Loop Pattern Implementation

## Summary

Implemented a best-practice conversation loop pattern for controlling the back-and-forth communication between BotFramework and LangFlow. The loop uses proactive messaging to send responses to users during processing and continues until a `COMPLETED` signal is received.

## Files Modified

### 1. `botframework/src/controllers/bot.controller.js`

**Changes:**
- Completely refactored `handleRouterResponse()` function to implement the conversation loop pattern
- Added loop control with `GO` and `COMPLETED` actions
- Added safeguards (max iterations to prevent infinite loops)
- Added continuous typing indicator during long-running operations
- Proactive messaging at each iteration step

### 2. `botframework/src/Handlers/LangFlowHandler.js`

**Changes:**
- Updated `processRoutingResult()` to support returning action fields for loop continuation
- Added new `parseSubFlowResult()` method to extract action from sub-flow responses
- Sub-flows can now return structured responses with `action: "GO"` or `action: "COMPLETED"`

## Conversation Loop Pattern

```
┌─────────────────────────────────────────────────────────────────┐
│                    USER MESSAGE RECEIVED                        │
│                           ↓                                     │
│              1. Validate user & save context                    │
│                           ↓                                     │
│              2. Call ROUTER for intent classification           │
│                           ↓                                     │
│     ┌─────────── CONVERSATION LOOP ──────────┐                  │
│     │                                         │                 │
│     │  3. Send MESSAGE to user (proactive)    │                 │
│     │  4. Update SUMMARY for context          │                 │
│     │  5. Check ACTION:                       │                 │
│     │     ├─ GO → Call sub-flow → Loop back   │                 │
│     │     └─ COMPLETED → Exit loop            │                 │
│     │                                         │                 │
│     └─────────────────────────────────────────┘                 │
│                           ↓                                     │
│              6. Cleanup & complete processing                   │
└─────────────────────────────────────────────────────────────────┘
```

## Loop Controls

| Action | Description | Behavior |
|--------|-------------|----------|
| `GO` | Continue processing | Call next flow, loop back to step 3 |
| `COMPLETED` | Stop processing | Exit loop, cleanup |

## Key Features

### 1. Proactive Messaging
- Messages are sent to the user at each loop iteration
- Users see intermediate responses during multi-step processing
- Typing indicator shown between iterations

### 2. Continuous Typing Indicator
- Typing indicator sent every 3 seconds during long-running operations
- Prevents "bot is not responding" perception

### 3. Max Iterations Safeguard
- Default: 10 iterations (configurable via `MAX_CONVERSATION_ITERATIONS` env var)
- Prevents infinite loops
- User-friendly error message when max reached

### 4. Context Continuity
- Summary accumulated across iterations
- User state updated at each step
- Previous route tracked for sub-flow context

## Response Format

### Router Response
```json
{
  "route": "IT_KB_SEARCH",
  "message": "I'll search the knowledge base for you...",
  "action": "GO",
  "summary": "User is looking for WiFi troubleshooting information"
}
```

### Sub-Flow Response (for loop continuation)
```json
{
  "message": "I found some results. Would you like more details?",
  "action": "GO",
  "summary": "Found 3 KB articles about WiFi. User may need more info."
}
```

### Sub-Flow Response (for completion)
```json
{
  "message": "Here are the search results...",
  "action": "COMPLETED"
}
```

## Configuration

### Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `MAX_CONVERSATION_ITERATIONS` | 10 | Maximum loop iterations before forced exit |

### Constants

| Constant | Value | Description |
|----------|-------|-------------|
| `TYPING_INDICATOR_INTERVAL` | 3000ms | Interval for sending typing indicators |

## Example Conversation Flow

```
User: "I need help with my WiFi"
                ↓
Router Response:
  route: IT_KB_SEARCH
  message: "I'll search our knowledge base for WiFi troubleshooting..."
  action: GO
                ↓
[Proactive Message Sent: "I'll search our knowledge base..."]
[Typing Indicator]
                ↓
KB Search Sub-Flow:
  message: "I found 3 articles. Would you like me to create a ticket?"
  action: GO
                ↓
[Proactive Message Sent: "I found 3 articles..."]
[Call Router Again]
                ↓
Router Response:
  route: IT_TICKET_MGMT
  message: "I'll help you create a ticket..."
  action: GO
                ↓
[Continue Loop...]
                ↓
Final Sub-Flow:
  message: "Your ticket INC123456 has been created."
  action: COMPLETED
                ↓
[Proactive Message Sent: "Your ticket INC123456..."]
[Loop Exits]
[Cleanup]
```

## Error Handling

1. **Flow Call Failure**: Send error message, exit loop
2. **Sub-Flow Error**: Clear typing interval, send error message, exit loop
3. **Max Iterations**: Send user-friendly message, exit loop
4. **Unknown Action**: Treat as COMPLETED, exit loop

## Testing Checklist

- [ ] Single iteration (COMPLETED on first response)
- [ ] Multi-iteration loop (GO → GO → COMPLETED)
- [ ] Max iterations safeguard triggers
- [ ] Error handling in loop
- [ ] Typing indicator during processing
- [ ] Proactive messages at each step
- [ ] Summary accumulation across iterations
- [ ] Sub-flow returns GO action
- [ ] Sub-flow returns COMPLETED action
- [ ] Sub-flow returns no action (defaults to COMPLETED)

## Related Documentation

- [LangFlow API Reference](https://docs.langflow.org/api-reference-api-examples)
- [Bot Framework Proactive Messages](https://learn.microsoft.com/en-us/microsoftteams/platform/bots/how-to/conversations/send-proactive-messages)
