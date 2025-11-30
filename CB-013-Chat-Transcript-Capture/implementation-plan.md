# CB-013: Implementation Plan - Chat Transcript Capture

## Technical Approach

### Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          CHAT TRANSCRIPT SYSTEM                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌──────────────┐    ┌────────────────────┐    ┌────────────────────────┐  │
│  │    User      │───▶│   Bot Controller   │───▶│ ChatTranscriptService  │  │
│  │   Message    │    │  (Capture Point)   │    │   (Record Message)     │  │
│  └──────────────┘    └────────────────────┘    └──────────┬─────────────┘  │
│                                                           │                 │
│  ┌──────────────┐    ┌────────────────────┐              │                 │
│  │     Bot      │◀───│   LangFlow/Flows   │◀─────────────┘                 │
│  │   Response   │    │  (Capture Point)   │                                │
│  └──────────────┘    └─────────┬──────────┘                                │
│                                │                                            │
│                                ▼                                            │
│                      ┌────────────────────┐    ┌────────────────────────┐  │
│                      │ ChatTranscriptSvc  │───▶│   Azure Table Storage  │  │
│                      │  (Record Response) │    │   (BotTranscripts)     │  │
│                      └─────────┬──────────┘    └────────────────────────┘  │
│                                │                                            │
│                                ▼                                            │
│                      ┌────────────────────┐    ┌────────────────────────┐  │
│                      │   TicketingFlow    │───▶│     ServiceNow API     │  │
│                      │ (Attach Transcript)│    │  (Incident Creation)   │  │
│                      └────────────────────┘    └────────────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Data Model

#### BotTranscripts Table Schema

| Field | Type | Description |
|-------|------|-------------|
| `partitionKey` | string | Session ID (UUID v7) |
| `rowKey` | string | Timestamp-based unique ID (`{timestamp}_{sequence}`) |
| `sessionId` | string | Session ID (for queries) |
| `userEmail` | string | User's email address |
| `channel` | string | Channel (msteams, directline) |
| `messageType` | string | `USER` or `BOT` |
| `sender` | string | Sender display name |
| `message` | string | Message content |
| `timestamp` | string | ISO 8601 timestamp |
| `metadata` | string | JSON string with additional data |

#### Transcript Entry Structure

```javascript
{
  partitionKey: "session-uuid-v7",
  rowKey: "1701234567890_001",  // timestamp_sequence
  sessionId: "session-uuid-v7",
  userEmail: "user@example.com",
  channel: "msteams",
  messageType: "USER",  // or "BOT"
  sender: "John Doe",   // or "Asclepius"
  message: "What is the dosage for aspirin?",
  timestamp: "2024-11-29T10:30:45.123Z",
  metadata: "{\"conversationId\":\"...\",\"route\":\"DRUG_SEARCH\"}"
}
```

### Message Capture Points

#### User Message Capture
Location: `bot.controller.js` → `handleTeamsMessage()`

```javascript
// After extracting user message, before processing
await chatTranscriptService.recordUserMessage({
  sessionId,
  userEmail,
  channel,
  message: messageText,
  senderName: enrichedActivity.from?.name
});
```

#### Bot Response Capture
Location: `bot.controller.js` → `handleRouterResponse()` (after sending proactive message)

```javascript
// After sending each proactive message
await chatTranscriptService.recordBotResponse({
  sessionId,
  userEmail,
  channel,
  message: extractedMessage,
  route: currentResult.route
});
```

---

## Implementation Steps

### Step 1: Create ChatTranscriptService

**File**: `src/Services/ChatTranscriptService.js`

Create a new service to handle transcript operations:

```javascript
/**
 * Chat Transcript Service
 *
 * Captures and stores conversation transcripts between users and the bot.
 * Provides transcript retrieval for ServiceNow ticket attachment.
 *
 * Storage: Azure Table Storage (BotTranscripts table)
 *
 * @see https://learn.microsoft.com/en-us/javascript/api/@azure/data-tables/tableclient
 */

const { TableClient, AzureNamedKeyCredential } = require('@azure/data-tables');
const { createLogger } = require('./Logger');

const logger = createLogger('ChatTranscriptService');

// Configuration
const AZ_SA_CONFIG = process.env.AZ_SA_CONFIG ? JSON.parse(process.env.AZ_SA_CONFIG) : null;
const TRANSCRIPTS_TABLE = 'BotTranscripts';
const BOT_NAME = process.env.BOT_DISPLAY_NAME || 'Asclepius';

// Message types
const MessageType = {
  USER: 'USER',
  BOT: 'BOT'
};

class ChatTranscriptService {
  // Implementation details in Step 1
}

module.exports = {
  ChatTranscriptService,
  chatTranscriptService: new ChatTranscriptService(),
  MessageType
};
```

**Key Methods**:

1. `recordUserMessage(params)` - Record user message entry
2. `recordBotResponse(params)` - Record bot response entry
3. `getTranscript(sessionId)` - Get all messages for a session
4. `getTranscriptByEmail(email, options)` - Get messages by user email
5. `formatTranscriptForDisplay(entries)` - Format as readable transcript
6. `formatTranscriptForServiceNow(entries)` - Format for ServiceNow attachment

### Step 2: Implement Core Recording Methods

```javascript
/**
 * Record a user message to the transcript
 * @param {Object} params - Message parameters
 * @returns {Promise<void>}
 */
async recordUserMessage({ sessionId, userEmail, channel, message, senderName, metadata = {} }) {
  if (!sessionId || !message) {
    logger.warn('Cannot record user message - missing sessionId or message');
    return;
  }

  const entry = this._buildEntry({
    sessionId,
    userEmail,
    channel,
    messageType: MessageType.USER,
    sender: senderName || userEmail?.split('@')[0] || 'User',
    message,
    metadata
  });

  await this._saveEntry(entry);
}

/**
 * Record a bot response to the transcript
 * @param {Object} params - Response parameters
 * @returns {Promise<void>}
 */
async recordBotResponse({ sessionId, userEmail, channel, message, route, metadata = {} }) {
  if (!sessionId || !message) {
    logger.warn('Cannot record bot response - missing sessionId or message');
    return;
  }

  const entry = this._buildEntry({
    sessionId,
    userEmail,
    channel,
    messageType: MessageType.BOT,
    sender: BOT_NAME,
    message,
    metadata: { ...metadata, route }
  });

  await this._saveEntry(entry);
}
```

### Step 3: Implement Transcript Retrieval

```javascript
/**
 * Get full transcript for a session
 * @param {string} sessionId - Session ID
 * @returns {Promise<Array>} Transcript entries in chronological order
 */
async getTranscript(sessionId) {
  if (!sessionId) return [];

  try {
    const filter = `PartitionKey eq '${sessionId}'`;
    const entries = this.transcriptClient.listEntities({
      queryOptions: { filter }
    });

    const results = [];
    for await (const entity of entries) {
      results.push(this._entityToEntry(entity));
    }

    // Sort by timestamp (rowKey)
    return results.sort((a, b) => a.timestamp.localeCompare(b.timestamp));
  } catch (error) {
    logger.error('Failed to get transcript', { sessionId, error: error.message });
    return [];
  }
}

/**
 * Format transcript for display
 * Format: [User]: message\n[Bot name]: response
 * @param {Array} entries - Transcript entries
 * @returns {string} Formatted transcript
 */
formatTranscriptForDisplay(entries) {
  if (!entries || entries.length === 0) return '';

  return entries
    .map(entry => {
      const label = entry.messageType === MessageType.USER ? 'User' : BOT_NAME;
      return `[${label}]: ${entry.message}`;
    })
    .join('\n\n');
}
```

### Step 4: Update Bot Controller for Message Capture

**File**: `src/controllers/bot.controller.js`

Add transcript capture at message entry and response points:

```javascript
// Import the service
const { chatTranscriptService } = require('../Services/ChatTranscriptService');

// In handleTeamsMessage(), after getting sessionId:
// Record user message (non-blocking)
chatTranscriptService.recordUserMessage({
  sessionId,
  userEmail,
  channel,
  message: messageText,
  senderName: enrichedActivity.from?.name,
  metadata: {
    conversationId,
    botId: enrichedActivity.recipient?.id
  }
}).catch(err => logger.warn('Failed to record user message', { error: err.message }));

// In handleRouterResponse(), after each proactive message send:
// Record bot response (non-blocking)
chatTranscriptService.recordBotResponse({
  sessionId,
  userEmail,
  channel,
  message: extractedMessage,
  route: route,
  metadata: {
    conversationId,
    iteration
  }
}).catch(err => logger.warn('Failed to record bot response', { error: err.message }));
```

### Step 5: Update ServiceNow Integration

**File**: `src/Services/ServiceNowService.js`

Add method to create incident with transcript:

```javascript
/**
 * Create incident with chat transcript attached
 * @param {Object} params - Incident parameters
 * @returns {Promise<Object>} Created incident
 */
async createIncidentWithTranscript({
  shortDescription,
  description,
  callerId,
  transcript,
  category,
  urgency,
  impact
}) {
  if (!this.isConfigured) {
    logger.warn('ServiceNow not configured');
    return null;
  }

  // Format transcript for work notes
  const workNotes = transcript
    ? `--- Chat Transcript ---\n\n${transcript}\n\n--- End of Transcript ---`
    : '';

  const incidentData = {
    short_description: shortDescription,
    description: description,
    caller_id: callerId,
    category: category || 'inquiry',
    urgency: urgency || '3',
    impact: impact || '3',
    work_notes: workNotes
  };

  try {
    const response = await axios({
      method: 'POST',
      url: this.getTableEndpoint('incident'),
      headers: this.getHeaders(),
      timeout: this.timeout,
      data: incidentData
    });

    logger.info('Incident created with transcript', {
      incidentNumber: response.data?.result?.number,
      hasTranscript: !!transcript
    });

    return response.data?.result;
  } catch (error) {
    logger.error('Failed to create incident', { error: error.message });
    throw this.handleError(error);
  }
}
```

### Step 6: Update TicketingFlow

**File**: `src/Handlers/LangFlow/flows/TicketingFlow.js`

Update to include transcript when creating tickets:

```javascript
const { chatTranscriptService } = require('../../../Services/ChatTranscriptService');

async execute(params) {
  const { action, sessionId, userEmail, ticketData = {} } = params;

  // ... existing validation ...

  // If creating a ticket, fetch and attach transcript
  if (action === 'create') {
    try {
      const transcriptEntries = await chatTranscriptService.getTranscript(sessionId);
      const formattedTranscript = chatTranscriptService.formatTranscriptForDisplay(transcriptEntries);

      // Add transcript to ticket data
      ticketData.transcript = formattedTranscript;
    } catch (err) {
      this.logger.warn('Failed to fetch transcript for ticket', { error: err.message });
    }
  }

  // ... rest of execute method ...
}
```

### Step 7: Add Configuration

**File**: `src/config/index.js`

Add transcript configuration:

```javascript
/**
 * Chat Transcript Configuration
 */
transcript: {
  // Bot display name for transcript
  botName: process.env.BOT_DISPLAY_NAME || 'Asclepius',
  // Maximum transcript entries to include in ticket
  maxEntriesForTicket: parseInt(process.env.TRANSCRIPT_MAX_ENTRIES) || 100,
  // Cleanup: days to keep transcripts
  retentionDays: parseInt(process.env.TRANSCRIPT_RETENTION_DAYS) || 90,
  // Enable/disable transcript capture
  enabled: process.env.ENABLE_TRANSCRIPT_CAPTURE !== 'false'
}
```

**File**: `.env.example`

Add environment variables:

```env
# Chat Transcript Configuration
BOT_DISPLAY_NAME=Asclepius
TRANSCRIPT_MAX_ENTRIES=100
TRANSCRIPT_RETENTION_DAYS=90
ENABLE_TRANSCRIPT_CAPTURE=true
```

---

## Files to Create

| File | Purpose |
|------|---------|
| `src/Services/ChatTranscriptService.js` | Core transcript capture and retrieval service |

## Files to Modify

| File | Changes |
|------|---------|
| `src/controllers/bot.controller.js` | Add transcript recording at message/response points |
| `src/Services/ServiceNowService.js` | Add `createIncidentWithTranscript()` method |
| `src/Handlers/LangFlow/flows/TicketingFlow.js` | Include transcript when creating tickets |
| `src/config/index.js` | Add transcript configuration |
| `.env.example` | Add transcript-related environment variables |

---

## Error Handling Strategy

### Non-Blocking Capture
Transcript recording should never block the main message flow:

```javascript
// Fire-and-forget with error logging
chatTranscriptService.recordUserMessage(params)
  .catch(err => logger.warn('Transcript recording failed', { error: err.message }));
```

### Retry Logic
For transient Azure Storage failures, implement retry with exponential backoff:

```javascript
async _saveEntryWithRetry(entry, retries = 3) {
  for (let attempt = 1; attempt <= retries; attempt++) {
    try {
      await this.transcriptClient.upsertEntity(entry, 'Replace');
      return;
    } catch (error) {
      if (attempt === retries) throw error;
      await this._delay(Math.pow(2, attempt) * 100);
    }
  }
}
```

### Graceful Degradation
If transcript retrieval fails during ticket creation, continue without transcript:

```javascript
try {
  const transcript = await chatTranscriptService.getTranscript(sessionId);
  ticketData.transcript = formatTranscript(transcript);
} catch (err) {
  logger.warn('Proceeding without transcript', { error: err.message });
}
```

---

## Testing Strategy

### Unit Tests
- [ ] `ChatTranscriptService` methods
- [ ] Transcript formatting
- [ ] Row key generation (uniqueness, ordering)
- [ ] In-memory fallback

### Integration Tests
- [ ] Azure Table Storage operations
- [ ] ServiceNow incident creation with transcript
- [ ] Full message flow with transcript capture

### Manual Testing

| Scenario | Channel | Expected Result |
|----------|---------|-----------------|
| Simple conversation | MS Teams | Transcript recorded in correct format |
| Multi-turn conversation | MS Teams | All messages captured in order |
| Ticket creation | MS Teams | Transcript attached to incident |
| Long conversation | MS Teams | All messages included (up to limit) |
| Service failure | MS Teams | Bot continues working, errors logged |

---

## Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| Performance impact | Non-blocking async writes |
| Storage costs | Retention policy, cleanup job |
| Large transcripts | Truncation with configurable limit |
| Sensitive data | No password/PII storage, follow existing patterns |
| ServiceNow field limits | Truncate transcript if exceeds limit |

---

## Rollback Plan

If issues occur:
1. Set `ENABLE_TRANSCRIPT_CAPTURE=false` to disable capture
2. Bot continues to function normally
3. Existing transcripts remain in storage
4. Can re-enable after fixing issues

---

## Future Considerations

1. **Transcript Cleanup Job**: Scheduled deletion of old transcripts
2. **Search API**: Search transcripts by content/keywords
3. **Analytics**: Analyze conversation patterns
4. **Export**: Export transcripts to other formats (PDF, CSV)
5. **Encryption**: Encrypt transcript content at rest

---

## Official Documentation References

| Topic | Link |
|-------|------|
| Azure Table Storage SDK | [TableClient API](https://learn.microsoft.com/en-us/javascript/api/@azure/data-tables/tableclient) |
| upsertEntity method | [upsertEntity](https://learn.microsoft.com/en-us/javascript/api/@azure/data-tables/tableclient#upsertEntity) |
| listEntities method | [listEntities](https://learn.microsoft.com/en-us/javascript/api/@azure/data-tables/tableclient#listEntities) |
| ServiceNow Table API | [Incident API](https://www.servicenow.com/docs/bundle/zurich-api-reference/page/integrate/inbound-rest/concept/c_TableAPI.html) |
