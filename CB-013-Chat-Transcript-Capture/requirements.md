# CB-013: Chat Transcript Capture

## Ticket Information

| Field | Value |
|-------|-------|
| **Ticket ID** | CB-013 |
| **Title** | Chat Transcript Capture |
| **Type** | Feature |
| **Priority** | High |
| **Components** | Services, Handlers, Controllers |

## Overview

Implement a chat transcript capture system that records the conversation history between users and the bot. The transcript will be:
1. Stored in Azure Table Storage for persistence and audit purposes
2. Attached to ServiceNow Incident tickets when ticket creation occurs

## Functional Requirements

### FR-1: Transcript Recording
- **FR-1.1**: Capture every user message sent to the bot
- **FR-1.2**: Capture every bot response sent to the user
- **FR-1.3**: Associate transcript entries with the user's session ID
- **FR-1.4**: Maintain chronological order of messages

### FR-2: Transcript Format
- **FR-2.1**: Format must follow the specified pattern:
  ```
  [User]: <user input>
  [Bot name]: <bot response>
  [User]: <user input>
  [Bot name]: <bot response>
  ```
- **FR-2.2**: Bot name should be configurable (e.g., "Asclepius", "ChatBot")
- **FR-2.3**: Handle multi-line messages properly (preserve line breaks)
- **FR-2.4**: Sanitize sensitive information if needed

### FR-3: Azure Table Storage
- **FR-3.1**: Store transcript entries in Azure Table Storage
- **FR-3.2**: Use session ID as partition key for efficient querying
- **FR-3.3**: Use timestamp-based row key for ordering
- **FR-3.4**: Include metadata (user email, channel, timestamp, message type)
- **FR-3.5**: Support transcript retrieval by session ID
- **FR-3.6**: Support transcript retrieval by user email

### FR-4: ServiceNow Integration
- **FR-4.1**: When a ticket is created, attach the full transcript to the incident
- **FR-4.2**: Include transcript in the `work_notes` or `additional_comments` field
- **FR-4.3**: Format transcript clearly for ServiceNow readability
- **FR-4.4**: Handle large transcripts (ServiceNow field limits)

## Non-Functional Requirements

### NFR-1: Performance
- Transcript storage should be non-blocking (async)
- Minimal impact on message response time
- Efficient retrieval for transcript compilation

### NFR-2: Reliability
- Transcript storage failure should not affect bot functionality
- Implement retry logic for transient failures
- Log errors for monitoring

### NFR-3: Security
- Do not store sensitive information (passwords, PII beyond email)
- Follow existing authentication patterns for Azure Storage
- Secure transcript access

### NFR-4: Scalability
- Handle high message volumes
- Efficient storage schema for query patterns
- Consider transcript cleanup/archival strategy

## Acceptance Criteria

### AC-1: Basic Transcript Capture
- [ ] User messages are recorded with timestamp
- [ ] Bot responses are recorded with timestamp
- [ ] Messages are associated with correct session

### AC-2: Transcript Format
- [ ] Transcript follows `[User]: message / [Bot name]: response` format
- [ ] Chronological order is maintained
- [ ] Multi-line messages are handled correctly

### AC-3: Azure Storage
- [ ] Transcript entries stored in Azure Table Storage
- [ ] Can retrieve full transcript by session ID
- [ ] Can retrieve transcript by user email

### AC-4: ServiceNow Integration
- [ ] Transcript attached to incident when ticket is created
- [ ] Transcript is readable in ServiceNow interface
- [ ] Large transcripts are handled appropriately

### AC-5: Error Handling
- [ ] Transcript failures do not break bot functionality
- [ ] Errors are logged appropriately
- [ ] Retry logic works for transient failures

## Dependencies

### Internal Dependencies
- `Az.StorageAccount.Table.js` - Existing Azure Table service
- `UserService.js` - User session management
- `ServiceNowService.js` - ServiceNow integration
- `TicketingFlow.js` - Ticket creation flow
- `bot.controller.js` - Message handling

### External Dependencies
| Package | Version | Documentation |
|---------|---------|---------------|
| `@azure/data-tables` | ^13.x | [Azure Table Storage SDK](https://learn.microsoft.com/en-us/javascript/api/@azure/data-tables/tableclient) |

## References

### Official Documentation
- [Azure Table Storage Node.js SDK](https://learn.microsoft.com/en-us/javascript/api/@azure/data-tables/)
- [Azure Table upsertEntity](https://learn.microsoft.com/en-us/javascript/api/@azure/data-tables/tableclient#upsertEntity)
- [ServiceNow Table API](https://www.servicenow.com/docs/bundle/zurich-api-reference/page/integrate/inbound-rest/concept/c_TableAPI.html)

### Internal Documentation
- [CB-003-Consolidated-Storage](../CB-003-Consolidated-Storage) - Storage patterns
- [ARCHITECTURE.md](../ARCHITECTURE.md) - System architecture
