# CB-003: Implementation Plan - Consolidated Storage

## Technical Approach

### Current Architecture (Two Tables)

```
BotUsers Table:
├── partitionKey: channel
├── rowKey: email
├── User fields: email, name, serviceNowSysId, department
├── Validation: validated, validationSource, validatedAt
├── Session: chat_session_id, current_input, next_action
├── Fallback: last_conversation_id, last_service_url  ← REDUNDANT
└── Activity: lastActivityAt

BotConversations Table:
├── partitionKey: {email}_{botId}_{channel}
├── rowKey: email
├── conversationId, botId, channel
├── userId, serviceUrl
└── createdAt, lastActivityAt
```

### Target Architecture (Single Table)

```
BotUsers Table:
├── partitionKey: channel
├── rowKey: email
├── User fields: email, name, serviceNowSysId, department
├── Validation: validated, validationSource, validatedAt
├── Session: chat_session_id, current_input, next_action
├── Conversation: conversationId, botId, userId, serviceUrl  ← CONSOLIDATED
└── Activity: lastActivityAt
```

## Implementation Steps

### Step 1: Update UserService Data Model

**File**: `src/Services/UserService.js`

1. Remove `CONVERSATIONS_TABLE` constant
2. Remove `this.conversationsClient` initialization
3. Update `UserRecord` typedef to include conversation fields
4. Remove `ConversationRecord` typedef
5. Remove `this.memoryConversations` Map

### Step 2: Merge Conversation Methods into User Methods

**Changes:**
- `saveConversation()` → Update user record directly with conversation fields
- `getConversation()` → Get user record and return conversation fields
- `getConversationByEmail()` → Get user record and return conversation fields
- Remove `updateConversationActivity()` (use updateUserActivity instead)
- Remove `entityToConversation()` helper

### Step 3: Simplify `findUserBySessionId()`

**Current:**
```javascript
// 1. Query users by session_id
// 2. Get user
// 3. Query conversations by email (SECOND QUERY)
// 4. If not found, use fallback from user entity
return { user, conversation };
```

**Target:**
```javascript
// 1. Query users by session_id
// 2. User already has conversation fields
return { user, conversationId, serviceUrl };
```

### Step 4: Update Controllers

**bot.controller.js:**
- `saveConversationInfo()` calls `userService.saveConversation()` which now updates user record

**webhook.controller.js:**
- `findUserBySessionId()` returns consolidated data
- No separate conversation lookup needed

### Step 5: Remove Unused Code

- Remove `BotConversations` table creation in `initializeTables()`
- Remove in-memory `memoryConversations` Map
- Remove conversation-specific methods if redundant

## Files to Modify

| File | Changes |
|------|---------|
| `Services/UserService.js` | Main refactoring - merge conversation into user |
| `controllers/bot.controller.js` | Minor - use updated method signatures |
| `controllers/webhook.controller.js` | Minor - simplified data access |

## Error Handling

- Conversation fields are optional (user may exist before first message)
- Null checks for conversationId, serviceUrl when user has no conversation yet

## Testing Strategy

### Unit Tests
- [ ] User creation without conversation
- [ ] User update with conversation fields
- [ ] Session lookup returns conversation data
- [ ] Backward compatibility with existing records

### Integration Tests
- [ ] Full message flow (Teams → Bot → LangFlow → Webhook → Teams)
- [ ] Proactive message delivery works
- [ ] Multiple conversations from same user handled correctly

### Manual Testing
- [ ] Send message in Teams → bot responds
- [ ] Webhook callback finds user and sends proactive message
- [ ] Check Azure Table only has BotUsers entries

## Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| Data loss during migration | New fields added via merge, existing data preserved |
| Webhook fails to find conversation | Conversation saved before LangFlow call |
| Multiple bots per user | Single conversation per user/channel/bot is current pattern |

## Rollback Plan

If issues occur:
1. Revert UserService.js changes
2. Re-add BotConversations table and methods
3. Data remains intact (only added new fields to users)
