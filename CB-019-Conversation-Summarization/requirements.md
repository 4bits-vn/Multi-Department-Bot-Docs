# CB-019: Conversation Summarization Feature

## Ticket Information

| Field | Value |
|-------|-------|
| **Ticket ID** | CB-019 |
| **Title** | Conversation Summarization |
| **Type** | Feature |
| **Priority** | High |
| **Created** | 2024-12-01 |
| **Status** | Completed |

## Overview

Implement a conversation summarization flow that generates summaries of user conversations. These summaries are stored and used as additional context for medical flows (MED_KB_SEARCH and MED_DRUG_SEARCH) to improve response quality.

## User Story

**As a user**, I want the bot to remember the context of my conversation so that when I ask medical or drug-related questions, it can provide more relevant and personalized responses.

## Workflow Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    CONVERSATION SUMMARIZATION WORKFLOW                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  User sends medical/drug query                                               │
│           │                                                                  │
│           ▼                                                                  │
│  ┌─────────────────────┐                                                    │
│  │   ROUTER            │                                                    │
│  │   (Intent: MED_KB_SEARCH │                                               │
│  │    or MED_DRUG_SEARCH)   │                                               │
│  └──────────┬──────────┘                                                    │
│             │                                                                │
│             ▼                                                                │
│  ┌─────────────────────────────────────────────────────────────┐            │
│  │ LangFlowHandler.processRoutingResult()                       │            │
│  │                                                              │            │
│  │ 1. Check for cached summary                                  │            │
│  │ 2. If no cache → Generate summary via SUMMARIZATION_FLOW           │            │
│  │ 3. Store summary in ConversationSummaryService               │            │
│  │ 4. Pass summary as contextSummary to medical flow            │            │
│  └──────────────────────────┬──────────────────────────────────┘            │
│                              │                                               │
│                              ▼                                               │
│  ┌─────────────────────────────────────────────────────────────┐            │
│  │ Medical Flow (MED_KB_SEARCH or MED_DRUG_SEARCH)              │            │
│  │                                                              │            │
│  │ Receives:                                                    │            │
│  │ - query: User's search query                                │            │
│  │ - contextSummary: Conversation context                       │            │
│  │                                                              │            │
│  │ Context is injected via LangFlow tweaks (CTX_INJECT_ID)     │            │
│  └─────────────────────────────────────────────────────────────┘            │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Functional Requirements

### FR-1: Summary Flow Integration
- The bot MUST support a `SUMMARIZATION_FLOW` configuration in LangFlow
- The flow ID MUST be configurable via environment variable
- The flow MUST accept conversation transcript as input and return a summary

### FR-2: Automatic Summary Generation
- When routing to MED_KB_SEARCH or MED_DRUG_SEARCH, the bot MUST:
  - Check for an existing cached summary
  - If no cache exists, generate a new summary via SUMMARIZATION_FLOW
  - Store the generated summary for future use

### FR-3: Summary Storage
- Summaries MUST be stored in ConversationSummaryService
- Storage MUST support both in-memory cache and Azure Table Storage persistence
- Summaries MUST have a configurable TTL (default: 24 hours)

### FR-4: Context Injection
- Summaries MUST be passed to medical flows via the `contextSummary` parameter
- Medical flows MUST inject the summary into LangFlow via tweaks (CTX_INJECT_ID)
- If summary generation fails, flows MUST continue without context (graceful degradation)

### FR-5: Transcript Handling
- Summary generation MUST retrieve transcript from ChatTranscriptService
- A minimum number of messages MUST be required for summary generation (configurable)
- Maximum transcript entries MUST be configurable to prevent excessive input

## Non-Functional Requirements

### NFR-1: Performance
- Summary generation MUST complete within 30 seconds
- Cached summaries MUST be retrieved in < 50ms
- Summary generation failures MUST NOT block the medical flow

### NFR-2: Reliability
- Summary generation errors MUST be logged but not cause user-facing failures
- Flows MUST continue without context if summary is unavailable

### NFR-3: Storage
- Azure Table Storage MUST be used for persistence when configured
- In-memory cache MUST be used as fallback for local development

## Configuration

### Environment Variables

```bash
# Summary Flow ID (required for summarization)
SUMMARIZATION_FLOW=<langflow_flow_id>

# Context injection node IDs (already existing)
MED_KB_SEARCH_FLOW_CTX_INJECT_ID=<node_id>
MED_DRUG_SEARCH_FLOW_CTX_INJECT_ID=<node_id>

# Optional configuration
SUMMARY_MAX_LENGTH=2000                    # Max summary length
SUMMARY_TTL_MS=86400000                    # Summary TTL (24h default)
SUMMARY_MAX_TRANSCRIPT_ENTRIES=20          # Max transcript entries
SUMMARY_MIN_MESSAGES=2                     # Min messages for summary
SUMMARY_ENABLE_CACHING=true                # Enable caching
```

## Acceptance Criteria

### AC-1: Summary Generation
- [ ] Bot generates conversation summary when medical flow is triggered
- [ ] Summary is generated from conversation transcript
- [ ] Summary is stored in ConversationSummaryService

### AC-2: Context Injection
- [ ] Summary is passed to MED_KB_SEARCH flow as contextSummary
- [ ] Summary is passed to MED_DRUG_SEARCH flow as contextSummary
- [ ] Medical flows use summary for enhanced responses

### AC-3: Caching
- [ ] Generated summaries are cached in memory
- [ ] Cached summaries are persisted to Azure Table Storage
- [ ] Cache respects configured TTL

### AC-4: Graceful Degradation
- [ ] Medical flows work correctly without summary
- [ ] Summary generation failures are logged but don't block flow
- [ ] Missing SUMMARIZATION_FLOW configuration is handled gracefully

## Dependencies

### Internal Dependencies
- `ChatTranscriptService` - Provides conversation transcript
- `ConversationSummaryService` - Stores/retrieves summaries
- `MedKBSearchFlow` - Consumes summary for medical queries
- `DrugSearchFlow` - Consumes summary for drug queries

### External Dependencies
- LangFlow `SUMMARIZATION_FLOW` (needs to be created in LangFlow)
- Azure Table Storage (optional, for persistence)

## References

- [LangFlow Integration Guidelines](../../.cursor/rules/langflow-integration.mdc)
- [Medical Integration Guidelines](../../.cursor/rules/medical-integration.mdc)
- [Chat Transcript Service](../../botframework/src/Services/ChatTranscriptService.js)
