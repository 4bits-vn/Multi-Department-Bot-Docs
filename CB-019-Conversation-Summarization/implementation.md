# CB-019: Conversation Summarization - Implementation

## Summary

Implemented a conversation summarization feature that generates summaries from chat transcripts for use as context in medical flows (MED_KB_SEARCH_FLOW and MED_DRUG_SEARCH_FLOW).

**Key Principle**: Single storage location - all conversation data including summaries is stored in `UserService` (BotUsers table). No separate caching or additional storage.

## Files Created

### 1. `src/Handlers/LangFlow/flows/SummaryFlow.js`

Flow handler for generating conversation summaries on-demand:
- **Flow ID**: `SUMMARIZATION_FLOW` from environment
- **Key Method**: `execute()` - Generate summary from transcript
- **Flow**:
  1. Retrieve transcript from ChatTranscriptService
  2. Format transcript for summarization
  3. Call SUMMARIZATION_FLOW in LangFlow
  4. Return the generated summary (no caching)

## Files Modified

### 1. `src/Handlers/LangFlow/config.js`

Added:
- `SUMMARIZATION_FLOW` configuration constant
- Added to `FLOWS` object
- Exported for use by other modules

### 2. `src/Handlers/LangFlow/flows/index.js`

Added exports for:
- `SummaryFlow` class
- `summaryFlow` singleton instance

### 3. `src/Handlers/LangFlow/index.js`

Added:
- Import for `summaryFlow`
- `callSummaryFlow()` method - generates summary from transcript
- Updated `processRoutingResult()` for `MED_KB_SEARCH` and `MED_DRUG_SEARCH`:
  - Uses `metadata.summary` from router (stored in UserService)
  - If not available, generates via SUMMARIZATION_FLOW
- Exported `summaryFlow`

### 4. `src/controllers/bot.controller.js`

- Uses `userData?.conversation_summary` from UserService only
- Summary is stored in UserService during conversation loop (line 716-721)
- No additional caching or storage services

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                    SINGLE STORAGE ARCHITECTURE                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                    UserService (BotUsers table)              │    │
│  │                                                              │    │
│  │  - conversation_summary: Stored by router during loop       │    │
│  │  - last_route: Current route                                 │    │
│  │  - current_input: User's message                            │    │
│  │  - session_id: Session identifier                           │    │
│  │  - All other user conversation data                         │    │
│  │                                                              │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

## Data Flow

```
User Message
     │
     ▼
┌─────────────────────────────────────────────────┐
│ bot.controller.js - handleMessage()             │
│                                                 │
│ 1. Get previousSummary from UserService         │
│ 2. Pass to router                               │
└─────────────────────────────────────────────────┘
     │
     ▼
┌─────────────────────────────────────────────────┐
│ handleRouterResponse() - Conversation Loop      │
│                                                 │
│ When router returns summary:                    │
│ → Store in UserService.conversation_summary     │
│                                                 │
│ Pass summary to processRoutingResult via        │
│ metadata.summary (accumulatedSummary)           │
└─────────────────────────────────────────────────┘
     │
     ▼
┌─────────────────────────────────────────────────┐
│ LangFlowHandler.processRoutingResult()          │
│                                                 │
│ For MED_KB_SEARCH / MED_DRUG_SEARCH:            │
│                                                 │
│ 1. Use metadata.summary (from router)           │
│    └─ Available during request processing       │
│                                                 │
│ 2. If not available → callSummaryFlow()         │
│    └─ Generate from transcript on-demand        │
│                                                 │
│ 3. Pass contextSummary to medical flow          │
└─────────────────────────────────────────────────┘
     │
     ▼
┌─────────────────────────────────────────────────┐
│ MedKBSearchFlow / DrugSearchFlow                │
│                                                 │
│ - Receives contextSummary parameter             │
│ - Builds tweaks with CTX_INJECT_ID              │
│ - Injects summary into LangFlow request         │
└─────────────────────────────────────────────────┘
```

## Configuration

### Environment Variables

```bash
# Required
SUMMARIZATION_FLOW=<langflow_flow_id>

# Optional (with defaults)
SUMMARY_MAX_TRANSCRIPT_ENTRIES=20    # Max transcript entries for summary
SUMMARY_MIN_MESSAGES=2               # Min messages required for summary

# Context injection (existing)
MED_KB_SEARCH_FLOW_CTX_INJECT_ID=<node_id>
MED_DRUG_SEARCH_FLOW_CTX_INJECT_ID=<node_id>
```

## Key Design Decisions

### 1. Single Storage Location
All conversation data including summaries stored in UserService (BotUsers table):
- Simplifies architecture
- Avoids data sync issues
- Summary always available via `userData.conversation_summary`

### 2. Summary Available During Request Processing
The `accumulatedSummary` in the conversation loop ensures summary is always available:
```javascript
// In handleRouterResponse
const { summary } = currentResult;
if (summary) {
    accumulatedSummary = summary;
    await userService.updateUserState(userEmail, channel, {
        conversation_summary: summary,
        last_route: route
    });
}

// Passed to processRoutingResult
metadata: {
    summary: accumulatedSummary,  // Always available during request
    ...
}
```

### 3. On-Demand Summary Generation
If no summary from router, generate from transcript:
```javascript
let contextSummary = metadata.summary || "";
if (!contextSummary) {
    const summaryResult = await this.callSummaryFlow({ sessionId, userEmail });
    contextSummary = summaryResult?.summary || "";
}
```

### 4. Graceful Degradation
Summary generation failures don't block medical flows:
```javascript
try {
    const summaryResult = await this.callSummaryFlow({ sessionId, userEmail });
    contextSummary = summaryResult?.summary || "";
} catch (summaryError) {
    logger.warn("Failed to generate summary, continuing without context");
}
```

## Testing Checklist

- [ ] Summary from router is stored in UserService
- [ ] Summary is passed to MED_KB_SEARCH flow via metadata
- [ ] Summary is passed to MED_DRUG_SEARCH flow via metadata
- [ ] Summary generation works when no router summary available
- [ ] Graceful degradation when SUMMARIZATION_FLOW not configured
- [ ] Graceful degradation when summary generation fails
- [ ] No additional caching services are used
