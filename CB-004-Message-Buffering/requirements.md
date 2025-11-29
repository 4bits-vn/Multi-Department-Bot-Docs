# CB-004: Message Buffering & Multi-Input Processing

## Ticket Information
- **Ticket ID**: CB-004
- **Type**: Enhancement
- **Priority**: High
- **Status**: In Progress
- **Created**: 2024-11-25

## Overview

Implement a message buffering mechanism that allows users to send multiple inputs within a conversation turn before the bot processes them together. This improves user experience by:

1. Allowing natural typing patterns (multiple short messages instead of one long message)
2. Preventing message loss during processing
3. Combining related messages for better context understanding by LangFlow

## Problem Statement

### Current Behavior
```
[12:00:00] User says: "hello"
           → Bot starts processing
[12:00:02] User says: "i got an issue with outlook"
           → Message REJECTED (bot is processing)
[12:00:05] Bot responds only to "hello"
           → User must repeat their issue
```

### Desired Behavior
```
[12:00:00] User says: "hello"
           → Bot shows typing indicator
           → Debounce timer starts (3 seconds)
[12:00:02] User says: "i got an issue with outlook"
           → Message added to buffer
           → Debounce timer resets
[12:00:05] Debounce timer expires
           → Bot processes: ["hello", "i got an issue with outlook"]
           → Combined input sent to LangFlow
```

## Functional Requirements

### FR-1: Message Buffering
- **FR-1.1**: System SHALL collect multiple user messages within a configurable time window (default: 3 seconds)
- **FR-1.2**: Buffer timer SHALL reset each time a new message arrives
- **FR-1.3**: Buffer SHALL have a maximum message count limit (default: 5 messages)
- **FR-1.4**: Buffer SHALL have a maximum wait time (default: 15 seconds) to prevent indefinite waiting
- **FR-1.5**: Buffered messages SHALL be combined with appropriate separator for LangFlow processing

### FR-2: Processing Lock
- **FR-2.1**: While processing, system SHALL queue incoming messages for the next batch
- **FR-2.2**: System SHALL notify user that previous messages are being processed
- **FR-2.3**: Queued messages during processing SHALL be processed automatically after completion

### FR-3: State Management
- **FR-3.1**: New state `BUFFERING` SHALL be added between `IDLE` and `PROCESSING`
- **FR-3.2**: State transitions SHALL be: `IDLE → BUFFERING → PROCESSING → IDLE`
- **FR-3.3**: State SHALL auto-recover from stuck states (existing timeout mechanism)

### FR-4: User Experience
- **FR-4.1**: Typing indicator SHALL be shown while buffering
- **FR-4.2**: User SHALL receive feedback if messages are queued during processing
- **FR-4.3**: System SHALL not lose any user messages

## Non-Functional Requirements

### NFR-1: Performance
- Buffering should add minimal latency (< 100ms overhead)
- Memory usage per conversation should be bounded (max 10KB buffer per conversation)

### NFR-2: Reliability
- Messages should never be lost
- System should recover gracefully from errors
- Buffer should be cleared on conversation timeout or user inactivity

### NFR-3: Scalability
- Solution should work with in-memory storage (current)
- Solution should be easily adaptable to Redis for distributed environments

## Acceptance Criteria

### AC-1: Basic Buffering
- [ ] Given a user sends "hello" and then "help me" within 3 seconds
- [ ] When the buffer timer expires
- [ ] Then the bot processes both messages together
- [ ] And the user receives a response addressing both messages

### AC-2: Processing Lock
- [ ] Given the bot is processing a previous message
- [ ] When the user sends a new message
- [ ] Then the message is queued (not lost)
- [ ] And the user receives an acknowledgment
- [ ] And the queued message is processed after completion

### AC-3: Buffer Limits
- [ ] Given the user sends 6 messages within 3 seconds (exceeds limit)
- [ ] When the 5th message is received
- [ ] Then processing starts immediately with 5 messages
- [ ] And the 6th message is queued for next batch

### AC-4: Timeout Protection
- [ ] Given the user keeps sending messages continuously for 20 seconds
- [ ] When the max wait time (15s) is reached
- [ ] Then processing starts with current buffer
- [ ] And subsequent messages are queued for next batch

## Configuration Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `BUFFER_DEBOUNCE_MS` | 3000 | Time to wait after last message before processing |
| `BUFFER_MAX_MESSAGES` | 5 | Maximum messages per batch |
| `BUFFER_MAX_WAIT_MS` | 15000 | Maximum wait time before forced processing |
| `BUFFER_MESSAGE_SEPARATOR` | `\n\n` | Separator for combining messages |
| `BUFFER_CLEANUP_INTERVAL_MS` | 60000 | How often to clean stale buffers |

## Dependencies

- ConversationStateService (existing) - Will be extended
- BotController (existing) - Will be updated
- WebhookController (existing) - Will be updated to handle queued messages

## References

- [Bot Framework State Management](https://learn.microsoft.com/en-us/azure/bot-service/bot-builder-concept-state)
- [Debounce Pattern](https://css-tricks.com/debouncing-throttling-explained-examples/)
- Existing implementation: `RequestDebounceService.old.js`
