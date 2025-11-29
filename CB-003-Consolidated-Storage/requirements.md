# CB-003: Consolidated Storage - Single Azure Table

## Overview

Consolidate MS Teams conversation ID and session ID management into a SINGLE Azure Table (`BotUsers`) instead of using two tables (`BotUsers` + `BotConversations`).

## Problem Statement

Currently, the application uses two Azure Tables:
1. **BotUsers** - User profile, validation, session state
2. **BotConversations** - MS Teams conversation mapping

This creates:
- Redundancy: Conversation info stored in both tables (Conversations table + fallback fields in Users)
- Complex lookups: Webhook needs to query Users by session ID, then Conversations by email
- More Azure Table operations and potential race conditions
- Code complexity with fallback patterns

## Proposed Solution

Consolidate ALL data into a single `BotUsers` table:
- User profile and validation status
- Session state (already exists)
- Conversation info (move from BotConversations)

## Functional Requirements

### FR-001: Single Table Storage
- All user, session, and conversation data stored in `BotUsers` table
- Remove `BotConversations` table creation and usage
- Remove fallback patterns for conversation lookup

### FR-002: Unified Data Model
- PartitionKey: `channel` (msteams, directline)
- RowKey: `email` (lowercase)
- Fields:
  - User: email, name, serviceNowSysId, department, validated, validationSource, validatedAt
  - Session: chat_session_id, current_input, next_action, next_action_updated_at
  - Conversation: conversationId, botId, userId, serviceUrl
  - Activity: lastActivityAt

### FR-003: Simplified Lookups
- `findUserBySessionId()`: Returns user WITH conversation info in single query
- No separate conversation table lookup needed
- No fallback patterns required

### FR-004: Backward Compatibility
- Existing user records continue to work
- New conversation fields added via merge operations

## Non-Functional Requirements

### NFR-001: Performance
- Reduced Azure Table operations (single table vs two)
- Faster webhook lookups (no secondary queries)

### NFR-002: Data Consistency
- Single source of truth for user + conversation data
- No data duplication across tables

## Acceptance Criteria

1. ✅ Only `BotUsers` table is created/used
2. ✅ Conversation data saved directly to user record
3. ✅ Webhook can find user + conversation in single operation
4. ✅ No `BotConversations` table client or methods
5. ✅ All existing functionality works (message handling, proactive messages)

## Dependencies

- Azure Data Tables SDK (`@azure/data-tables`)
- Existing UserService and ConversationStateService

## References

- [Azure Table Storage Best Practices](https://learn.microsoft.com/en-us/azure/cosmos-db/table/how-to-use-nodejs)
- CB-002-ConversationState-Refactor documentation
