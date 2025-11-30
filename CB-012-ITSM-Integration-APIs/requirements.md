# CB-012: ITSM Integration APIs

## Ticket Information

- **Ticket Number**: CB-012-ITSM-Integration-APIs
- **Type**: Feature
- **Priority**: High
- **Created**: 2024-11-30

## Overview

Due to LangFlow's limitations in calling external APIs directly, this feature implements REST APIs for **multi-provider ITSM integration**. These APIs will be called by LangFlow flows for incident management operations, following the same pattern as the existing Drug and Medical query APIs.

The architecture uses an **adapter/provider pattern** to support multiple ITSM systems:
- **ServiceNow** (Phase 1 - Primary)
- **Jira Service Management** (Phase 2)
- **Zendesk** (Future)
- **Freshservice** (Future)
- **BMC Remedy** (Future)

The approach separates concerns:
- **LangFlow**: Handles classification, intent routing, and payload preparation
- **Bot Framework APIs**: Handles actual ITSM provider interactions through a unified interface

## Problem Statement

LangFlow has limitations when making direct API calls to external systems like ServiceNow ITSM. Additionally, different clients may use different ITSM systems. To maintain a clean separation of concerns, leverage LangFlow's strengths in conversation flow management, and support multiple ITSM providers, we need to expose dedicated APIs with a provider-agnostic interface that LangFlow can call.

## Key Design Decisions

### Provider Abstraction Pattern

All ITSM providers implement a common interface (`IITSMProvider`), ensuring:
- **Unified API**: LangFlow flows work identically regardless of backend ITSM system
- **Easy Switching**: Change ITSM provider via configuration without code changes
- **Normalized Data**: Consistent incident structure across all providers
- **Extensibility**: Add new providers by implementing the interface

## Functional Requirements

### FR-1: Get List of Incidents

**Description**: Retrieve a filtered list of incidents for a user or based on query criteria.

**Endpoint**: `GET /api/itsm/incidents`

**Query Parameters**:
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `callerId` | string | No | Caller email or sys_id |
| `state` | string | No | Filter by state (e.g., "open", "closed", "all") |
| `priority` | string | No | Filter by priority (1-5) |
| `limit` | number | No | Max results (default: 10, max: 50) |
| `orderBy` | string | No | Sort field (default: "sys_created_on") |
| `orderDir` | string | No | Sort direction ("asc"/"desc", default: "desc") |

**Use Cases**:
- User asks "What are my open tickets?"
- User asks "Show me my recent tickets"
- User asks "Check ticket status" (without ticket number)

### FR-2: Get Incident Details

**Description**: Retrieve detailed information about a specific incident by number or sys_id.

**Endpoint**: `GET /api/itsm/incidents/:identifier`

**Path Parameters**:
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `identifier` | string | Yes | Incident number (INCxxxxxxx) or sys_id |

**Query Parameters**:
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `includeComments` | boolean | No | Include work notes and comments |
| `includeHistory` | boolean | No | Include state change history |

**Use Cases**:
- User provides ticket number "Show me INC0010001"
- User selects a ticket from the list
- System needs to display full ticket information

### FR-3: Create Incident

**Description**: Create a new incident with provided data.

**Endpoint**: `POST /api/itsm/incidents`

**Request Body**:
```json
{
  "shortDescription": "string (required)",
  "description": "string",
  "callerId": "string (email or sys_id, required)",
  "category": "string",
  "subcategory": "string",
  "urgency": "number (1-3)",
  "impact": "number (1-3)",
  "assignmentGroup": "string",
  "additionalComments": "string",
  "configurationItem": "string"
}
```

**Use Cases**:
- User creates a new ticket through the chatbot
- System creates ticket from collected information
- Escalation to live agent creates ticket

### FR-4: Update Incident - Add Comment

**Description**: Add a comment or work note to an existing incident.

**Endpoint**: `POST /api/itsm/incidents/:identifier/comments`

**Path Parameters**:
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `identifier` | string | Yes | Incident number or sys_id |

**Request Body**:
```json
{
  "comment": "string (required)",
  "type": "work_notes | comments (default: comments)",
  "isInternal": "boolean (default: false for comments)"
}
```

**Use Cases**:
- User wants to add information to existing ticket
- System adds chat transcript to ticket
- Escalation notes are added

### FR-5: Update Incident State

**Description**: Update the state of an incident (cancel, resolve, etc.).

**Endpoint**: `PATCH /api/itsm/incidents/:identifier/state`

**Path Parameters**:
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `identifier` | string | Yes | Incident number or sys_id |

**Request Body**:
```json
{
  "state": "number (required) - see state codes",
  "closeCode": "string (required for resolved/closed)",
  "closeNotes": "string (required for resolved/closed)",
  "reason": "string (optional, for audit trail)"
}
```

**State Codes**:
| Code | State |
|------|-------|
| 1 | New |
| 2 | In Progress |
| 3 | On Hold |
| 6 | Resolved |
| 7 | Closed |
| 8 | Canceled |

**Use Cases**:
- User cancels their own ticket
- User confirms resolution
- System updates ticket status

### FR-6: Escalate Incident

**Description**: Escalate an incident by adding escalation comment and optionally updating priority.

**Endpoint**: `POST /api/itsm/incidents/:identifier/escalate`

**Path Parameters**:
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `identifier` | string | Yes | Incident number or sys_id |

**Request Body**:
```json
{
  "reason": "string (optional, default: 'User requested escalation')",
  "increasePriority": "boolean (default: false)",
  "notifyAssignee": "boolean (default: true)"
}
```

**Use Cases**:
- User wants to escalate existing ticket
- User needs urgent attention on ticket

### FR-7: Health Check

**Description**: Check ServiceNow connectivity and service health.

**Endpoint**: `GET /api/itsm/health`

**Response**:
```json
{
  "status": "healthy | degraded | unhealthy",
  "servicenow": {
    "connected": true,
    "latencyMs": 150,
    "instanceUrl": "https://xxx.service-now.com"
  },
  "timestamp": "ISO date"
}
```

## Non-Functional Requirements

### NFR-1: Performance

- API response time < 3 seconds for single incident operations
- List queries < 5 seconds with pagination
- Connection timeout: 30 seconds (configurable)

### NFR-2: Security

- Basic authentication with ServiceNow
- No sensitive data in logs (mask passwords, sys_ids in error logs)
- Input validation on all endpoints
- Rate limiting consideration for ServiceNow API

### NFR-3: Reliability

- Retry logic for transient failures (3 retries with exponential backoff)
- Graceful degradation when ServiceNow is unavailable
- Comprehensive error handling with user-friendly messages

### NFR-4: Observability

- Request/response logging with correlation IDs
- Metrics for API latency and error rates
- ServiceNow API call tracking

## Acceptance Criteria

### AC-1: Incident List
- [ ] Can retrieve incidents filtered by caller email
- [ ] Can filter by state (open/closed/all)
- [ ] Pagination works correctly
- [ ] Returns formatted incident summaries

### AC-2: Incident Details
- [ ] Can retrieve by incident number (INCxxxxxxx)
- [ ] Can retrieve by sys_id
- [ ] Returns all relevant incident fields
- [ ] Optionally includes comments and history

### AC-3: Create Incident
- [ ] Creates incident with required fields
- [ ] Validates caller exists or creates caller reference
- [ ] Returns created incident number and sys_id
- [ ] Handles field mapping correctly

### AC-4: Add Comment
- [ ] Adds comment to specified incident
- [ ] Supports both work_notes and comments field
- [ ] Returns success confirmation

### AC-5: Update State
- [ ] Updates incident state correctly
- [ ] Validates state transitions
- [ ] Requires close notes for resolution/closure
- [ ] Returns updated incident

### AC-6: Escalate
- [ ] Adds escalation comment
- [ ] Optionally increases priority
- [ ] Returns confirmation

### AC-7: Health Check
- [ ] Returns ServiceNow connectivity status
- [ ] Measures and returns latency
- [ ] Returns appropriate status codes

## Dependencies

### External Dependencies

**ServiceNow (Phase 1)**:
- ServiceNow instance (configured via environment variables)
- ServiceNow Table API access
- Valid ServiceNow integration credentials (Basic Auth)

**Jira Service Management (Phase 2)**:
- Atlassian Cloud or Data Center instance
- Jira REST API v3 access
- API token authentication

**Future Providers**:
- Provider-specific API access and credentials

### Internal Dependencies
- Logger service
- Express routing infrastructure
- Axios for HTTP requests

### Architecture Dependencies
- Provider abstraction pattern (adapter pattern)
- Factory pattern for provider instantiation
- Normalized data models across providers

## References

### Official Documentation
- [ServiceNow Table API](https://www.servicenow.com/docs/bundle/zurich-api-reference/page/integrate/inbound-rest/concept/c_TableAPI.html)
- [ServiceNow Encoded Query Strings](https://www.servicenow.com/docs/bundle/zurich-platform-user-interface/page/use/using-lists/concept/c_EncodedQueryStrings.html)

### Project Documentation
- [ServiceNow Integration Guidelines](/.cursor/rules/servicenow-integration.mdc)
- [Drug Controller Pattern](botframework/src/controllers/drug.controller.js) - Reference implementation
- [Medical Controller Pattern](botframework/src/controllers/medical.controller.js) - Reference implementation

## Out of Scope

- Live agent integration (discussed separately)
- ServiceNow OAuth authentication (use basic auth for now)
- File attachment handling
- Approval workflow integration
- Change request management
