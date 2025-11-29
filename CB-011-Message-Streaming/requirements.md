# CB-011: Message Streaming for MS Teams

## Ticket Information

| Field | Value |
|-------|-------|
| Ticket ID | CB-011 |
| Type | Enhancement |
| Priority | High |
| Status | Planning |
| Created | 2024-11-29 |

## Overview

Implement message streaming capability for the Bot Framework integration with MS Teams to significantly enhance user experience. Instead of waiting for complete responses, users will see real-time updates as the bot processes their requests.

### Why This Matters

Currently, when users send messages to the bot:
- They see only a simple typing indicator
- They wait without feedback while LangFlow processes (can take 5-30+ seconds)
- The complete response appears all at once after processing completes

With message streaming:
- Users see **informative updates** during processing (e.g., "Searching knowledge base...")
- Users see the **response being typed out** in real-time
- This creates a more engaging, responsive experience that increases user trust

## Functional Requirements

### FR-1: Informative Updates During Processing

The bot MUST display informative status messages during different processing phases:

| Processing Phase | Informative Message Example |
|-----------------|----------------------------|
| Router processing | "Understanding your request..." |
| IT KB Search | "Searching IT knowledge base..." |
| Medical KB Search | "Looking up health information..." |
| Drug Search | "Searching drug database..." |
| Ticket Management | "Checking your tickets..." |
| ServiceNow API call | "Connecting to support system..." |
| LangFlow sub-flow | "Processing your request..." |

### FR-2: Response Streaming

When LangFlow generates a response:
- The bot SHOULD stream text content to users as it becomes available
- Streaming MUST follow the Teams API contract (each chunk contains previous content)
- The final message MAY include attachments (Adaptive Cards, etc.)

### FR-3: Stop Streaming Support

- Users SHOULD be able to stop streaming responses using Teams' built-in Stop button
- The bot MUST handle `ContentStreamNotAllowed` error gracefully when streaming is stopped

### FR-4: Channel Support

- Message streaming is ONLY supported for **one-on-one chats** in MS Teams
- Group chats and channels MUST fall back to standard message delivery
- DirectLine channel MUST continue using standard message delivery

## Non-Functional Requirements

### NFR-1: Performance

- Informative updates MUST appear within 100ms of state change
- Streaming chunks SHOULD be buffered for 1.5-2 seconds to avoid throttling
- Maximum streaming duration: 2 minutes (Teams platform limit)

### NFR-2: Reliability

- If streaming fails, fall back to standard message delivery
- Handle rate limiting (1 request/second for streaming API)
- Implement proper cleanup on stream termination

### NFR-3: Compatibility

- MUST work with existing LangFlow integration
- MUST maintain backward compatibility with current message flow
- MUST support existing Adaptive Card responses in final message

## Acceptance Criteria

1. ✅ Users see informative status messages during bot processing
2. ✅ Text responses stream incrementally to users
3. ✅ Final messages can include Adaptive Cards and attachments
4. ✅ Streaming works reliably in one-on-one Teams chats
5. ✅ Graceful fallback when streaming is not available
6. ✅ No regression in existing bot functionality
7. ✅ Error handling for all streaming edge cases

## Dependencies

### Technical Dependencies

- MS Teams Platform (GA support for streaming in web, desktop, mobile)
- Bot Framework REST API
- Existing LangFlow integration

### External Documentation

- [Microsoft Teams Streaming Bot Messages](https://learn.microsoft.com/en-us/microsoftteams/platform/bots/streaming-ux)
- [Bot Framework REST API Reference](https://learn.microsoft.com/en-us/azure/bot-service/rest-api/bot-framework-rest-connector-api-reference)

## Out of Scope

- Streaming for group chats (not supported by Teams platform)
- Streaming for DirectLine channel
- Real-time LLM token streaming from LangFlow (future enhancement)
- AI-powered features (citations, sensitivity labels) beyond final message

## References

- [Microsoft Teams Streaming UX Documentation](https://learn.microsoft.com/en-us/microsoftteams/platform/bots/streaming-ux?tabs=csharp#stream-message-through-rest-api)
- [Bot Messages with AI-Generated Content](https://learn.microsoft.com/en-us/microsoftteams/platform/bots/how-to/bot-messages-ai-generated-content)
