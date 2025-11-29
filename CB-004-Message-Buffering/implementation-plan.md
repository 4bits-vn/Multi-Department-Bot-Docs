# CB-004: Message Buffering Implementation Plan

## Technical Approach

### Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                         User Message                             │
└─────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│                       BotController                              │
│  1. Receive message                                             │
│  2. Check state via ConversationStateService                    │
│  3. Add to MessageBufferService                                 │
└─────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│                    MessageBufferService                          │
│  - Collects messages per conversationId                         │
│  - Manages debounce timers                                      │
│  - Triggers processing callback when ready                      │
└─────────────────────────────────────────────────────────────────┘
                               │
                               ▼ (callback when buffer ready)
┌─────────────────────────────────────────────────────────────────┐
│                    Processing Flow                               │
│  1. Set state to PROCESSING                                     │
│  2. Combine buffered messages                                   │
│  3. Send to LangFlow                                            │
│  4. Wait for webhook callback                                   │
│  5. Set state back to IDLE                                      │
│  6. Process any queued messages                                 │
└─────────────────────────────────────────────────────────────────┘
```

### State Machine

```
         ┌──────────────────────────────────────────┐
         │                                          │
         ▼                                          │
     ┌───────┐    message    ┌───────────┐         │
     │ IDLE  │──────────────▶│ BUFFERING │         │
     └───────┘               └───────────┘         │
         ▲                        │                │
         │                        │ debounce       │
         │                        │ expires        │
         │                        ▼                │
         │                  ┌────────────┐         │
         │                  │ PROCESSING │─────────┘
         │                  └────────────┘    complete
         │                        │
         │    auto-recover        │ error
         └────────────────────────┤
                                  ▼
                             ┌─────────┐
                             │  ERROR  │
                             └─────────┘
```

## Implementation Steps

### Step 1: Create MessageBufferService

Create new service: `src/Services/MessageBufferService.js`

**Responsibilities:**
- Store message buffer per conversationId
- Manage debounce timers
- Trigger processing callback when buffer ready
- Queue messages during processing
- Handle buffer cleanup

**Key Methods:**
- `addMessage(conversationId, message, metadata)` - Add message to buffer
- `getBufferedMessages(conversationId)` - Get all buffered messages
- `clearBuffer(conversationId)` - Clear buffer after processing
- `isBuffering(conversationId)` - Check if actively buffering
- `queueWhileProcessing(conversationId, message)` - Queue during processing
- `getQueuedMessages(conversationId)` - Get queued messages for next batch

### Step 2: Update ConversationStateService

Update: `src/Services/ConversationStateService.js`

**Changes:**
- Add `BUFFERING` state to ProcessingState enum
- Add `startBuffering()` method
- Update state transition logic
- Add buffer metadata tracking

### Step 3: Update BotController

Update: `src/controllers/bot.controller.js`

**Changes in `handleTeamsMessage()`:**
1. Check current state
2. If IDLE: Start buffering, add message, show typing
3. If BUFFERING: Add message to buffer, show typing
4. If PROCESSING: Queue message for next batch, notify user
5. Register callback to process when buffer ready

### Step 4: Update WebhookController

Update: `src/controllers/webhook.controller.js`

**Changes:**
- After completing processing, check for queued messages
- If queued messages exist, start new processing cycle
- Ensure state transitions are correct

### Step 5: Add Configuration

Update: `src/config/index.js`

**Add new configuration:**
```javascript
buffer: {
    debounceMs: parseInt(process.env.BUFFER_DEBOUNCE_MS) || 3000,
    maxMessages: parseInt(process.env.BUFFER_MAX_MESSAGES) || 5,
    maxWaitMs: parseInt(process.env.BUFFER_MAX_WAIT_MS) || 15000,
    messageSeparator: process.env.BUFFER_MESSAGE_SEPARATOR || '\n\n',
    cleanupIntervalMs: parseInt(process.env.BUFFER_CLEANUP_INTERVAL_MS) || 60000
}
```

## Integration Points

### LangFlow Integration
- Combined messages sent as single input with separator
- Session ID maintained across buffered messages
- Metadata includes buffer info for context

### Teams Integration
- Typing indicator shown during buffering and processing
- User notified when messages queued during processing
- All feedback through proactive messages

## Error Handling Strategy

### Buffer Errors
- If buffer fails, immediately process single message
- Log error but don't block user

### Timer Errors
- If timer fails to fire, max wait timeout acts as backup
- Stale buffers cleaned up periodically

### Processing Errors
- Existing error handling in ConversationStateService
- ERROR state with auto-recovery

## Testing Strategy

### Unit Tests
- MessageBufferService.addMessage() collects correctly
- Debounce timer resets on new messages
- Max messages triggers immediate processing
- Max wait time triggers processing
- Queued messages handled correctly

### Integration Tests
- Full flow: buffer → process → respond
- Multiple messages combined correctly
- Queued messages processed after completion
- State transitions correct

### Manual Testing Checklist
| Scenario | Expected |
|----------|----------|
| Send 2 messages quickly | Both processed together |
| Send message while processing | Message queued, processed after |
| Send 5+ messages quickly | First 5 processed, rest queued |
| Send messages for 15+ seconds | Processing triggered at max wait |
| Error during processing | State recovers, messages not lost |

## Risks & Mitigations

### Risk 1: Memory Usage
- **Risk**: Large buffers consuming memory
- **Mitigation**: Max message limit, periodic cleanup, bounded buffer size

### Risk 2: Lost Messages
- **Risk**: Messages lost during state transitions
- **Mitigation**: Queue mechanism for processing state, comprehensive logging

### Risk 3: Infinite Loops
- **Risk**: Queued messages triggering more queued messages
- **Mitigation**: Clear queued messages before processing, max batch size

### Risk 4: Timer Race Conditions
- **Risk**: Multiple timers firing simultaneously
- **Mitigation**: Clear timer before setting new one, single timer per conversation

## File Changes Summary

| File | Change Type |
|------|-------------|
| `src/Services/MessageBufferService.js` | **New** |
| `src/Services/ConversationStateService.js` | Update |
| `src/controllers/bot.controller.js` | Update |
| `src/controllers/webhook.controller.js` | Update |
| `src/config/index.js` | Update |

## Official Documentation References

- [Bot Framework State Management](https://learn.microsoft.com/en-us/azure/bot-service/bot-builder-concept-state)
- [Bot Framework Proactive Messages](https://learn.microsoft.com/en-us/microsoftteams/platform/bots/how-to/conversations/send-proactive-messages)
- [Node.js setTimeout/clearTimeout](https://nodejs.org/api/timers.html)
