# CB-013: Implementation - Chat Transcript Capture

## Summary

Implemented a chat transcript capture system that:
1. Records all user messages and bot responses to Azure Table Storage
2. Automatically attaches formatted transcripts to ServiceNow tickets when created
3. Uses non-blocking async operations to avoid impacting bot performance

## Files Created

| File | Description |
|------|-------------|
| `src/Services/ChatTranscriptService.js` | Core service for transcript recording, retrieval, and formatting |

## Files Modified

| File | Changes |
|------|---------|
| `src/config/index.js` | Added transcript configuration section |
| `src/controllers/bot.controller.js` | Added transcript recording at message capture points |
| `src/Services/ServiceNowService.js` | Added `createIncidentWithTranscript()` and `addWorkNotes()` methods |
| `src/Handlers/LangFlow/flows/TicketingFlow.js` | Integrated transcript attachment for ticket creation |
| `.env.example` | Added transcript-related environment variables |

## Key Implementation Details

### 1. ChatTranscriptService

**Location**: `src/Services/ChatTranscriptService.js`

```javascript
// Core methods
chatTranscriptService.recordUserMessage({...})     // Record user input
chatTranscriptService.recordBotResponse({...})     // Record bot response
chatTranscriptService.getTranscript(sessionId)     // Get all messages
chatTranscriptService.formatTranscriptForDisplay() // Format for display
chatTranscriptService.formatTranscriptForServiceNow() // Format for tickets
```

**Storage Schema** (Azure Table: `BotTranscripts`):
- `partitionKey`: Session ID (UUID v7)
- `rowKey`: Timestamp-based unique ID
- `messageType`: `USER` or `BOT`
- `sender`: User name or bot name
- `message`: Message content
- `timestamp`: ISO 8601 timestamp
- `metadata`: JSON with additional context

### 2. Message Capture Points

**User Message Recording** (bot.controller.js:353-364):
```javascript
// After obtaining sessionId, before processing
chatTranscriptService.recordUserMessage({
    sessionId,
    userEmail,
    channel,
    message: messageText,
    senderName: enrichedActivity.from?.name,
    metadata: { conversationId, botId }
}).catch(err => logger.warn('Failed to record user message', { error: err.message }));
```

**Bot Response Recording** (bot.controller.js - multiple locations):
- Main router response (line 581-589)
- Resolution follow-up message (line 640-648)
- Sub-flow GO response (line 728-736)
- Sub-flow COMPLETED response (line 780-788)

### 3. Transcript Format

Output format as specified:
```
[User]: What is the dosage for aspirin?

[Asclepius]: Aspirin dosage varies based on the condition being treated...

[User]: Can I take it with food?

[Asclepius]: Yes, it's recommended to take aspirin with food or milk...
```

### 4. ServiceNow Integration

**New Methods** in `ServiceNowService.js`:

```javascript
// Create incident with transcript
await serviceNowService.createIncidentWithTranscript({
    shortDescription,
    description,
    callerId,
    transcript,  // Formatted transcript string
    category,
    urgency,
    impact
});

// Add work notes to existing incident
await serviceNowService.addWorkNotes(incidentSysId, workNotes);
```

**Transcript in Incident** (work_notes field):
```
--- Chat Transcript ---

[User]: I need help with my VPN connection

[Asclepius]: I'd be happy to help with your VPN issue. What error are you seeing?

--- End of Transcript ---
```

### 5. TicketingFlow Integration

The `TicketingFlow.execute()` method now:
1. Checks if action is 'create'
2. Fetches transcript entries for the session
3. Formats transcript for ServiceNow
4. Includes transcript in the ticket data sent to LangFlow

```javascript
// Automatic transcript attachment
if (action === 'create' && sessionId) {
    const transcriptEntries = await chatTranscriptService.getTranscript(sessionId);
    const formattedTranscript = chatTranscriptService.formatTranscriptForServiceNow(transcriptEntries);
    transcriptData = { transcript: formattedTranscript };
}
```

## Configuration

### Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `ENABLE_TRANSCRIPT_CAPTURE` | `true` | Enable/disable transcript capture |
| `BOT_DISPLAY_NAME` | `Asclepius` | Bot name in transcript |
| `TRANSCRIPT_MAX_ENTRIES` | `100` | Max entries per ticket |
| `TRANSCRIPT_RETENTION_DAYS` | `90` | Days to keep transcripts |
| `TRANSCRIPT_MAX_LENGTH` | `32000` | Max chars for ServiceNow |

### Config Object

```javascript
config.transcript = {
    enabled: true,
    botName: 'Asclepius',
    maxEntriesForTicket: 100,
    retentionDays: 90,
    maxLengthForServiceNow: 32000
};
```

## Design Decisions

### 1. Non-Blocking Recording
Transcript recording uses fire-and-forget pattern to avoid impacting response time:
```javascript
chatTranscriptService.recordUserMessage({...})
    .catch(err => logger.warn('...'));  // Don't await
```

### 2. Session-Based Partitioning
Using session ID as partition key enables:
- Efficient retrieval of all messages in a conversation
- Natural grouping by conversation
- Good query performance

### 3. Graceful Degradation
If transcript operations fail:
- Bot continues to function normally
- Errors are logged for monitoring
- Ticket creation proceeds without transcript

### 4. In-Memory Fallback
For local development without Azure Storage:
- Transcripts stored in memory Map
- Same API, different backend
- Enables testing without cloud resources

## Testing Performed

### Manual Testing Checklist
- [ ] User message recorded on send
- [ ] Bot response recorded after proactive message
- [ ] Transcript retrieved by session ID
- [ ] Transcript formatted correctly
- [ ] Ticket creation includes transcript
- [ ] Transcript visible in ServiceNow work_notes
- [ ] Feature can be disabled via env var
- [ ] In-memory fallback works locally

## Known Limitations

1. **No Transcript Cleanup Job**: Retention policy defined but cleanup job not implemented
2. **Single Table**: All transcripts in one table - may need sharding for very high volume
3. **No Encryption**: Transcripts stored in plain text (follows existing pattern)
4. **LangFlow Dependency**: Ticket creation still goes through LangFlow - transcript passed as data

## Follow-Up Tasks

1. **Transcript Cleanup Job**: Implement scheduled job to delete old transcripts
2. **Transcript Search API**: Add endpoint to search transcripts by content
3. **Analytics Dashboard**: Analyze conversation patterns from transcripts
4. **Export Feature**: Export transcripts to PDF/CSV format

## References

- [Azure Table Storage SDK](https://learn.microsoft.com/en-us/javascript/api/@azure/data-tables/)
- [ServiceNow Table API](https://www.servicenow.com/docs/bundle/zurich-api-reference/page/integrate/inbound-rest/concept/c_TableAPI.html)
- [CB-003-Consolidated-Storage](../CB-003-Consolidated-Storage) - Storage patterns used
