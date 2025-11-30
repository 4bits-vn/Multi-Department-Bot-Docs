# CB-012: ITSM Integration APIs - Implementation Plan

## Overview

This document outlines the implementation plan for **multi-provider ITSM Integration APIs** that will be consumed by LangFlow flows for incident management operations. The architecture uses an **adapter/provider pattern** to support multiple ITSM systems including ServiceNow, Jira Service Management, Zendesk, Freshservice, BMC Remedy, and others.

## Architecture

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              LangFlow                                        │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  Flows: Intent Classification, Payload Preparation, Response Format │   │
│  └───────────────────────────────┬─────────────────────────────────────┘   │
│                                  │                                         │
└──────────────────────────────────┼─────────────────────────────────────────┘
                                   │ HTTP API Calls
                                   ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                        Bot Framework Server                                  │
│                                                                             │
│  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────────┐     │
│  │  ITSM Routes    │───▶│ ITSM Controller │───▶│  IncidentService    │     │
│  │  /api/itsm/*    │    │                 │    │  (Orchestration)    │     │
│  └─────────────────┘    └─────────────────┘    └──────────┬──────────┘     │
│                                                            │                │
│                                                            ▼                │
│                              ┌─────────────────────────────────────────┐   │
│                              │       ITSMProviderFactory               │   │
│                              │   (Provider Selection & Creation)       │   │
│                              └──────────────────┬──────────────────────┘   │
│                                                 │                          │
│                    ┌────────────────────────────┼────────────────────────┐ │
│                    │                            │                        │ │
│                    ▼                            ▼                        ▼ │
│  ┌─────────────────────────┐  ┌─────────────────────────┐  ┌───────────────┐
│  │  ServiceNowProvider     │  │  JiraServiceProvider    │  │  Future...    │
│  │  implements             │  │  implements             │  │  Zendesk      │
│  │  IITSMProvider          │  │  IITSMProvider          │  │  Freshservice │
│  └───────────┬─────────────┘  └───────────┬─────────────┘  │  BMC Remedy   │
│              │                            │                └───────────────┘
└──────────────┼────────────────────────────┼────────────────────────────────┘
               │ REST API                   │ REST API
               ▼                            ▼
┌──────────────────────────┐  ┌──────────────────────────┐
│    ServiceNow Instance   │  │  Jira Service Management │
│    /api/now/table/*      │  │  /rest/api/3/*           │
└──────────────────────────┘  └──────────────────────────┘
```

### Provider Abstraction Pattern

```
┌─────────────────────────────────────────────────────────────────┐
│                      IITSMProvider Interface                     │
├─────────────────────────────────────────────────────────────────┤
│  + getIncidents(options): Promise<IncidentListResult>           │
│  + getIncident(identifier, options): Promise<IncidentResult>    │
│  + createIncident(data): Promise<CreateIncidentResult>          │
│  + updateIncident(id, data): Promise<UpdateIncidentResult>      │
│  + addComment(id, comment, type): Promise<CommentResult>        │
│  + updateState(id, state, options): Promise<StateResult>        │
│  + escalateIncident(id, options): Promise<EscalateResult>       │
│  + validateUser(email): Promise<UserResult>                     │
│  + healthCheck(): Promise<HealthResult>                         │
│  + getProviderInfo(): ProviderInfo                              │
└─────────────────────────────────────────────────────────────────┘
                              △
                              │ implements
          ┌───────────────────┼───────────────────┐
          │                   │                   │
┌─────────┴─────────┐ ┌───────┴───────┐ ┌────────┴────────┐
│ ServiceNowProvider│ │ JiraProvider  │ │ ZendeskProvider │
│                   │ │               │ │                 │
│ - Maps to         │ │ - Maps to     │ │ - Maps to       │
│   ServiceNow API  │ │   Jira API    │ │   Zendesk API   │
│ - INC numbers     │ │ - Issue keys  │ │ - Ticket IDs    │
│ - sys_id          │ │ - Issue IDs   │ │ - Ticket IDs    │
└───────────────────┘ └───────────────┘ └─────────────────┘
```

## Supported ITSM Providers

| Provider | Status | Ticket Format | API Type |
|----------|--------|---------------|----------|
| **ServiceNow** | ✅ Phase 1 | INCxxxxxxx | REST Table API |
| **Jira Service Management** | 📋 Phase 2 | PROJECT-123 | REST API v3 |
| **Zendesk** | 📋 Future | #123456 | REST API v2 |
| **Freshservice** | 📋 Future | #123456 | REST API v2 |
| **BMC Remedy** | 📋 Future | INCxxxxxxx | REST API |
| **Custom/Mock** | 📋 Testing | MOCK-123 | In-memory |

## Technical Approach

### Component Structure

```
botframework/src/
├── controllers/
│   └── itsm.controller.js              # HTTP request handlers
│
├── routes/
│   └── itsm.routes.js                  # API route definitions
│
├── Services/
│   └── ITSM/                           # NEW: ITSM Module
│       ├── index.js                    # Module exports
│       ├── interfaces.js               # TypeScript-style interfaces (JSDoc)
│       ├── types.js                    # Shared types and enums
│       ├── ITSMProviderFactory.js      # Provider factory
│       ├── IncidentService.js          # High-level orchestration
│       │
│       └── providers/                  # Provider implementations
│           ├── index.js                # Provider exports
│           ├── BaseITSMProvider.js     # Abstract base class
│           ├── ServiceNowProvider.js   # ServiceNow implementation
│           ├── JiraServiceProvider.js  # Jira SM implementation (Phase 2)
│           └── MockProvider.js         # Mock for testing
│
├── utils/
│   └── itsmFormatter.js                # Response formatting utilities
│
└── config/
    └── itsm.config.js                  # ITSM provider configuration
```

### File Summary

| File | Action | Description |
|------|--------|-------------|
| `src/Services/ITSM/index.js` | Create | Module exports |
| `src/Services/ITSM/interfaces.js` | Create | Interface definitions (JSDoc) |
| `src/Services/ITSM/types.js` | Create | Shared types, enums, constants |
| `src/Services/ITSM/ITSMProviderFactory.js` | Create | Provider factory pattern |
| `src/Services/ITSM/IncidentService.js` | Create | Orchestration layer |
| `src/Services/ITSM/providers/BaseITSMProvider.js` | Create | Abstract base class |
| `src/Services/ITSM/providers/ServiceNowProvider.js` | Create | ServiceNow implementation |
| `src/Services/ITSM/providers/MockProvider.js` | Create | Mock for testing |
| `src/controllers/itsm.controller.js` | Create | HTTP handlers |
| `src/routes/itsm.routes.js` | Create | Express route definitions |
| `src/routes/index.js` | Modify | Register ITSM routes |
| `src/utils/itsmFormatter.js` | Create | Format incident data |
| `src/config/itsm.config.js` | Create | Provider configuration |

---

## Implementation Steps

### Step 1: Create ITSM Types and Interfaces

**File**: `src/Services/ITSM/types.js`

Define shared types, enums, and constants used across all ITSM providers.

```javascript
/**
 * ITSM Types and Constants
 *
 * Shared types, enums, and constants for multi-provider ITSM integration.
 * These provide a normalized data model across different ITSM systems.
 *
 * @module Services/ITSM/types
 * @see CB-012-ITSM-Integration-APIs
 */

/**
 * Supported ITSM Providers
 * @readonly
 * @enum {string}
 */
const ITSMProvider = {
    SERVICENOW: 'servicenow',
    JIRA: 'jira',
    ZENDESK: 'zendesk',
    FRESHSERVICE: 'freshservice',
    BMC_REMEDY: 'bmc_remedy',
    MOCK: 'mock'
};

/**
 * Normalized Incident States (provider-agnostic)
 * Each provider maps their states to these normalized values
 * @readonly
 * @enum {string}
 */
const IncidentState = {
    NEW: 'new',
    IN_PROGRESS: 'in_progress',
    ON_HOLD: 'on_hold',
    PENDING: 'pending',
    RESOLVED: 'resolved',
    CLOSED: 'closed',
    CANCELED: 'canceled'
};

/**
 * Normalized Priority Levels
 * @readonly
 * @enum {string}
 */
const IncidentPriority = {
    CRITICAL: 'critical',      // P1
    HIGH: 'high',              // P2
    MEDIUM: 'medium',          // P3
    LOW: 'low',                // P4
    PLANNING: 'planning'       // P5
};

/**
 * Comment Types
 * @readonly
 * @enum {string}
 */
const CommentType = {
    PUBLIC: 'public',          // Customer-visible comments
    INTERNAL: 'internal'       // Internal/work notes
};

/**
 * State display configuration
 */
const STATE_DISPLAY = {
    [IncidentState.NEW]: { label: 'New', color: 'attention', icon: '🆕' },
    [IncidentState.IN_PROGRESS]: { label: 'In Progress', color: 'accent', icon: '🔄' },
    [IncidentState.ON_HOLD]: { label: 'On Hold', color: 'warning', icon: '⏸️' },
    [IncidentState.PENDING]: { label: 'Pending', color: 'warning', icon: '⏳' },
    [IncidentState.RESOLVED]: { label: 'Resolved', color: 'good', icon: '✅' },
    [IncidentState.CLOSED]: { label: 'Closed', color: 'good', icon: '🔒' },
    [IncidentState.CANCELED]: { label: 'Canceled', color: 'default', icon: '❌' }
};

/**
 * Priority display configuration
 */
const PRIORITY_DISPLAY = {
    [IncidentPriority.CRITICAL]: { label: 'Critical', color: 'attention', level: 1 },
    [IncidentPriority.HIGH]: { label: 'High', color: 'warning', level: 2 },
    [IncidentPriority.MEDIUM]: { label: 'Medium', color: 'accent', level: 3 },
    [IncidentPriority.LOW]: { label: 'Low', color: 'good', level: 4 },
    [IncidentPriority.PLANNING]: { label: 'Planning', color: 'default', level: 5 }
};

module.exports = {
    ITSMProvider,
    IncidentState,
    IncidentPriority,
    CommentType,
    STATE_DISPLAY,
    PRIORITY_DISPLAY
};
```

**File**: `src/Services/ITSM/interfaces.js`

Define JSDoc interfaces for type safety and documentation.

```javascript
/**
 * ITSM Interfaces
 *
 * JSDoc interface definitions for ITSM provider abstraction.
 * These define the contract that all providers must implement.
 *
 * @module Services/ITSM/interfaces
 */

/**
 * @typedef {Object} ProviderConfig
 * @property {string} type - Provider type (servicenow, jira, etc.)
 * @property {string} url - Instance URL
 * @property {Object} auth - Authentication configuration
 * @property {string} [auth.username] - Username for basic auth
 * @property {string} [auth.password] - Password for basic auth
 * @property {string} [auth.apiKey] - API key authentication
 * @property {string} [auth.token] - OAuth/Bearer token
 * @property {number} [timeout=30000] - Request timeout in ms
 * @property {Object} [defaults] - Default values for incidents
 */

/**
 * @typedef {Object} NormalizedIncident
 * @property {string} id - Provider-specific unique ID (sys_id, issue id, etc.)
 * @property {string} number - Human-readable ticket number (INC000001, PROJ-123)
 * @property {string} shortDescription - Brief description/summary
 * @property {string} [description] - Full description
 * @property {string} state - Normalized state (IncidentState enum)
 * @property {string} stateDisplay - Display label for state
 * @property {string} priority - Normalized priority (IncidentPriority enum)
 * @property {string} priorityDisplay - Display label for priority
 * @property {string} [category] - Category/type
 * @property {string} [subcategory] - Subcategory
 * @property {Object} [caller] - Caller/reporter info
 * @property {string} caller.id - Caller ID
 * @property {string} caller.name - Caller display name
 * @property {string} [caller.email] - Caller email
 * @property {Object} [assignee] - Assigned person
 * @property {Object} [assignmentGroup] - Assigned team/group
 * @property {string} createdAt - ISO date string
 * @property {string} updatedAt - ISO date string
 * @property {string} [resolvedAt] - ISO date string
 * @property {string} [closedAt] - ISO date string
 * @property {string} url - Direct link to ticket in ITSM system
 * @property {string} provider - Provider type that returned this incident
 * @property {Object} [raw] - Original provider-specific data
 */

/**
 * @typedef {Object} IncidentListResult
 * @property {boolean} success - Operation success
 * @property {NormalizedIncident[]} incidents - List of incidents
 * @property {number} total - Total matching incidents
 * @property {number} limit - Applied limit
 * @property {number} offset - Applied offset
 * @property {boolean} hasMore - More results available
 * @property {string} provider - Provider type
 */

/**
 * @typedef {Object} IncidentResult
 * @property {boolean} success - Operation success
 * @property {boolean} found - Incident found
 * @property {NormalizedIncident} [incident] - The incident
 * @property {string[]} [availableActions] - Actions available for this incident
 * @property {string} provider - Provider type
 */

/**
 * @typedef {Object} CreateIncidentData
 * @property {string} shortDescription - Brief description (required)
 * @property {string} [description] - Full description
 * @property {string} callerId - Caller email or ID (required)
 * @property {string} [category] - Category
 * @property {string} [subcategory] - Subcategory
 * @property {string} [priority] - Priority (normalized)
 * @property {string} [urgency] - Urgency level
 * @property {string} [impact] - Impact level
 * @property {string} [assignmentGroup] - Assignment group
 * @property {string} [additionalComments] - Initial comments
 * @property {Object} [customFields] - Provider-specific custom fields
 */

/**
 * @typedef {Object} CreateIncidentResult
 * @property {boolean} success - Operation success
 * @property {string} id - Created incident ID
 * @property {string} number - Created incident number
 * @property {string} url - URL to view incident
 * @property {NormalizedIncident} incident - Full incident data
 * @property {string} provider - Provider type
 */

/**
 * @typedef {Object} HealthCheckResult
 * @property {string} status - 'healthy' | 'degraded' | 'unhealthy'
 * @property {string} provider - Provider type
 * @property {boolean} connected - Connection successful
 * @property {number} latencyMs - Response latency
 * @property {string} [instanceUrl] - Instance URL
 * @property {string} [error] - Error message if unhealthy
 * @property {string} timestamp - ISO timestamp
 */

/**
 * @typedef {Object} ProviderInfo
 * @property {string} type - Provider type
 * @property {string} name - Display name
 * @property {string} version - API version
 * @property {string[]} capabilities - Supported capabilities
 * @property {string} ticketPrefix - Expected ticket number prefix
 */

module.exports = {
    // JSDoc types are exported for documentation
    // No runtime exports needed
};
```

---

### Step 2: Create Base ITSM Provider

**File**: `src/Services/ITSM/providers/BaseITSMProvider.js`

Abstract base class that all ITSM providers extend.

```javascript
/**
 * Base ITSM Provider
 *
 * Abstract base class that defines the interface for all ITSM providers.
 * Each concrete provider (ServiceNow, Jira, etc.) extends this class.
 *
 * @abstract
 * @module Services/ITSM/providers/BaseITSMProvider
 * @see CB-012-ITSM-Integration-APIs
 */

const axios = require('axios');
const { createLogger } = require('../../Logger');
const { ITSMProvider, IncidentState, IncidentPriority } = require('../types');

class BaseITSMProvider {
    /**
     * @param {import('../interfaces').ProviderConfig} config
     */
    constructor(config) {
        if (new.target === BaseITSMProvider) {
            throw new Error('BaseITSMProvider is abstract and cannot be instantiated directly');
        }

        this.config = config;
        this.providerType = config.type;
        this.instanceUrl = config.url;
        this.timeout = config.timeout || 30000;
        this.defaults = config.defaults || {};
        this.logger = createLogger(`ITSM:${this.providerType}`);

        // Validate required config
        this._validateConfig();
    }

    /**
     * Validate provider configuration
     * @protected
     */
    _validateConfig() {
        if (!this.instanceUrl) {
            throw new Error(`${this.providerType}: Instance URL is required`);
        }
    }

    /**
     * Get provider information
     * @returns {import('../interfaces').ProviderInfo}
     */
    getProviderInfo() {
        throw new Error('getProviderInfo() must be implemented by subclass');
    }

    /**
     * Check if provider is configured and ready
     * @returns {boolean}
     */
    isConfigured() {
        return !!(this.instanceUrl && this.config.auth);
    }

    // ============================================
    // ABSTRACT METHODS - Must be implemented
    // ============================================

    /**
     * Get list of incidents
     * @abstract
     * @param {Object} options - Query options
     * @returns {Promise<import('../interfaces').IncidentListResult>}
     */
    async getIncidents(options = {}) {
        throw new Error('getIncidents() must be implemented by subclass');
    }

    /**
     * Get single incident by identifier
     * @abstract
     * @param {string} identifier - Ticket number or ID
     * @param {Object} [options] - Query options
     * @returns {Promise<import('../interfaces').IncidentResult>}
     */
    async getIncident(identifier, options = {}) {
        throw new Error('getIncident() must be implemented by subclass');
    }

    /**
     * Create new incident
     * @abstract
     * @param {import('../interfaces').CreateIncidentData} data
     * @returns {Promise<import('../interfaces').CreateIncidentResult>}
     */
    async createIncident(data) {
        throw new Error('createIncident() must be implemented by subclass');
    }

    /**
     * Update incident
     * @abstract
     * @param {string} id - Incident ID
     * @param {Object} updateData - Fields to update
     * @returns {Promise<Object>}
     */
    async updateIncident(id, updateData) {
        throw new Error('updateIncident() must be implemented by subclass');
    }

    /**
     * Add comment to incident
     * @abstract
     * @param {string} id - Incident ID
     * @param {string} comment - Comment text
     * @param {string} [type='public'] - Comment type
     * @returns {Promise<Object>}
     */
    async addComment(id, comment, type = 'public') {
        throw new Error('addComment() must be implemented by subclass');
    }

    /**
     * Update incident state
     * @abstract
     * @param {string} id - Incident ID
     * @param {string} state - New state (normalized)
     * @param {Object} [options] - Additional options
     * @returns {Promise<Object>}
     */
    async updateState(id, state, options = {}) {
        throw new Error('updateState() must be implemented by subclass');
    }

    /**
     * Escalate incident
     * @param {string} id - Incident ID
     * @param {Object} [options] - Escalation options
     * @returns {Promise<Object>}
     */
    async escalateIncident(id, options = {}) {
        // Default implementation: add escalation comment
        const reason = options.reason || 'User requested escalation via chatbot';
        return this.addComment(id, `[ESCALATION] ${reason}`, 'internal');
    }

    /**
     * Validate/lookup user
     * @abstract
     * @param {string} email - User email
     * @returns {Promise<Object|null>}
     */
    async validateUser(email) {
        throw new Error('validateUser() must be implemented by subclass');
    }

    /**
     * Health check
     * @returns {Promise<import('../interfaces').HealthCheckResult>}
     */
    async healthCheck() {
        const startTime = Date.now();
        try {
            // Default: try to get 1 incident
            await this.getIncidents({ limit: 1 });
            return {
                status: 'healthy',
                provider: this.providerType,
                connected: true,
                latencyMs: Date.now() - startTime,
                instanceUrl: this.instanceUrl,
                timestamp: new Date().toISOString()
            };
        } catch (error) {
            return {
                status: 'unhealthy',
                provider: this.providerType,
                connected: false,
                latencyMs: Date.now() - startTime,
                instanceUrl: this.instanceUrl,
                error: error.message,
                timestamp: new Date().toISOString()
            };
        }
    }

    // ============================================
    // HELPER METHODS - Shared across providers
    // ============================================

    /**
     * Make HTTP request with error handling
     * @protected
     * @param {Object} options - Axios request options
     * @returns {Promise<Object>}
     */
    async _request(options) {
        try {
            const response = await axios({
                ...options,
                timeout: this.timeout,
                headers: {
                    ...this._getAuthHeaders(),
                    'Content-Type': 'application/json',
                    'Accept': 'application/json',
                    ...options.headers
                }
            });
            return response.data;
        } catch (error) {
            throw this._handleError(error);
        }
    }

    /**
     * Get authentication headers
     * @protected
     * @returns {Object}
     */
    _getAuthHeaders() {
        const auth = this.config.auth || {};

        if (auth.username && auth.password) {
            const basic = Buffer.from(`${auth.username}:${auth.password}`).toString('base64');
            return { 'Authorization': `Basic ${basic}` };
        }

        if (auth.token) {
            return { 'Authorization': `Bearer ${auth.token}` };
        }

        if (auth.apiKey) {
            return { 'X-API-Key': auth.apiKey };
        }

        return {};
    }

    /**
     * Handle API errors
     * @protected
     * @param {Error} error - Axios error
     * @returns {Error}
     */
    _handleError(error) {
        if (error.response) {
            const status = error.response.status;
            const data = error.response.data;

            if (status === 401) {
                return new Error(`${this.providerType}: Authentication failed`);
            }
            if (status === 403) {
                return new Error(`${this.providerType}: Access denied`);
            }
            if (status === 404) {
                return new Error(`${this.providerType}: Resource not found`);
            }
            if (status === 429) {
                return new Error(`${this.providerType}: Rate limit exceeded`);
            }

            return new Error(`${this.providerType}: ${data?.error?.message || error.message}`);
        }

        if (error.code === 'ECONNABORTED') {
            return new Error(`${this.providerType}: Request timeout`);
        }

        return error;
    }

    /**
     * Get available actions for an incident based on state
     * @protected
     * @param {string} state - Normalized state
     * @returns {string[]}
     */
    _getAvailableActions(state) {
        const openStates = [IncidentState.NEW, IncidentState.IN_PROGRESS, IncidentState.ON_HOLD, IncidentState.PENDING];

        if (openStates.includes(state)) {
            return ['add_comment', 'cancel', 'escalate', 'view_details'];
        }

        return ['view_details'];
    }
}

module.exports = BaseITSMProvider;
```

---

### Step 3: Create ServiceNow Provider

**File**: `src/Services/ITSM/providers/ServiceNowProvider.js`

ServiceNow-specific implementation of the ITSM provider interface.

```javascript
/**
 * ServiceNow ITSM Provider
 *
 * Implements the ITSM provider interface for ServiceNow.
 * Handles all ServiceNow-specific API interactions and data mapping.
 *
 * @extends BaseITSMProvider
 * @module Services/ITSM/providers/ServiceNowProvider
 * @see https://www.servicenow.com/docs/bundle/zurich-api-reference/page/integrate/inbound-rest/concept/c_TableAPI.html
 */

const BaseITSMProvider = require('./BaseITSMProvider');
const { ITSMProvider, IncidentState, IncidentPriority, CommentType } = require('../types');

/**
 * ServiceNow state code mapping to normalized states
 */
const SERVICENOW_STATE_MAP = {
    1: IncidentState.NEW,
    2: IncidentState.IN_PROGRESS,
    3: IncidentState.ON_HOLD,
    6: IncidentState.RESOLVED,
    7: IncidentState.CLOSED,
    8: IncidentState.CANCELED
};

/**
 * Reverse mapping: normalized state to ServiceNow code
 */
const NORMALIZED_TO_SERVICENOW_STATE = {
    [IncidentState.NEW]: 1,
    [IncidentState.IN_PROGRESS]: 2,
    [IncidentState.ON_HOLD]: 3,
    [IncidentState.RESOLVED]: 6,
    [IncidentState.CLOSED]: 7,
    [IncidentState.CANCELED]: 8
};

/**
 * ServiceNow priority mapping
 */
const SERVICENOW_PRIORITY_MAP = {
    1: IncidentPriority.CRITICAL,
    2: IncidentPriority.HIGH,
    3: IncidentPriority.MEDIUM,
    4: IncidentPriority.LOW,
    5: IncidentPriority.PLANNING
};

/**
 * Default fields to retrieve from ServiceNow
 */
const DEFAULT_FIELDS = [
    'sys_id', 'number', 'short_description', 'description',
    'state', 'priority', 'urgency', 'impact',
    'category', 'subcategory', 'caller_id', 'assigned_to',
    'assignment_group', 'sys_created_on', 'sys_updated_on',
    'resolved_at', 'closed_at'
];

class ServiceNowProvider extends BaseITSMProvider {
    constructor(config) {
        super({ ...config, type: ITSMProvider.SERVICENOW });
    }

    /**
     * @override
     */
    getProviderInfo() {
        return {
            type: ITSMProvider.SERVICENOW,
            name: 'ServiceNow',
            version: 'Zurich+',
            capabilities: ['incidents', 'comments', 'workNotes', 'users', 'attachments'],
            ticketPrefix: 'INC'
        };
    }

    /**
     * Get table API endpoint
     * @private
     */
    _getTableEndpoint(table) {
        return `${this.instanceUrl}/api/now/table/${table}`;
    }

    /**
     * @override
     */
    async getIncidents(options = {}) {
        const {
            callerEmail,
            state = 'open',
            priority,
            limit = 10,
            offset = 0,
            orderBy = 'sys_created_on',
            orderDir = 'desc'
        } = options;

        // Build encoded query
        const queryParts = [];

        if (callerEmail) {
            queryParts.push(`caller_id.email=${encodeURIComponent(callerEmail)}`);
        }

        if (state === 'open') {
            queryParts.push('stateNOT IN6,7,8');
        } else if (state === 'closed') {
            queryParts.push('stateIN6,7,8');
        }

        if (priority) {
            // Convert normalized priority to ServiceNow code
            const snPriority = this._normalizedPriorityToServiceNow(priority);
            if (snPriority) queryParts.push(`priority=${snPriority}`);
        }

        const queryString = queryParts.join('^');

        this.logger.info('Searching incidents', { queryString, limit, offset });

        const response = await this._request({
            method: 'GET',
            url: this._getTableEndpoint('incident'),
            params: {
                sysparm_query: queryString,
                sysparm_limit: limit,
                sysparm_offset: offset,
                sysparm_fields: DEFAULT_FIELDS.join(','),
                sysparm_display_value: 'all',
                sysparm_order_by: `${orderDir === 'desc' ? '-' : ''}${orderBy}`
            }
        });

        const incidents = (response.result || []).map(inc => this._normalizeIncident(inc));

        return {
            success: true,
            incidents,
            total: incidents.length, // Note: ServiceNow doesn't return total count by default
            limit,
            offset,
            hasMore: incidents.length === limit,
            provider: ITSMProvider.SERVICENOW
        };
    }

    /**
     * @override
     */
    async getIncident(identifier, options = {}) {
        this.logger.info('Getting incident', { identifier });

        let incident;

        // Determine if identifier is number or sys_id
        if (identifier.toUpperCase().startsWith('INC')) {
            // Query by number
            const response = await this._request({
                method: 'GET',
                url: this._getTableEndpoint('incident'),
                params: {
                    sysparm_query: `number=${identifier.toUpperCase()}`,
                    sysparm_limit: 1,
                    sysparm_fields: DEFAULT_FIELDS.join(','),
                    sysparm_display_value: 'all'
                }
            });
            incident = response.result?.[0];
        } else {
            // Direct get by sys_id
            const response = await this._request({
                method: 'GET',
                url: `${this._getTableEndpoint('incident')}/${identifier}`,
                params: {
                    sysparm_fields: DEFAULT_FIELDS.join(','),
                    sysparm_display_value: 'all'
                }
            });
            incident = response.result;
        }

        if (!incident) {
            return {
                success: true,
                found: false,
                provider: ITSMProvider.SERVICENOW
            };
        }

        const normalized = this._normalizeIncident(incident);

        return {
            success: true,
            found: true,
            incident: normalized,
            availableActions: this._getAvailableActions(normalized.state),
            provider: ITSMProvider.SERVICENOW
        };
    }

    /**
     * @override
     */
    async createIncident(data) {
        const {
            shortDescription,
            description,
            callerId,
            category,
            subcategory,
            urgency = 3,
            impact = 3,
            assignmentGroup,
            additionalComments
        } = data;

        this.logger.info('Creating incident', {
            shortDescription: shortDescription?.substring(0, 50) + '...'
        });

        // Resolve caller email to sys_id if needed
        let callerSysId = callerId;
        if (callerId && callerId.includes('@')) {
            const user = await this.validateUser(callerId);
            if (user?.sys_id) {
                callerSysId = user.sys_id;
            }
        }

        const incidentData = {
            short_description: shortDescription,
            description,
            caller_id: callerSysId,
            category: category || this.defaults.category,
            subcategory,
            urgency: String(urgency),
            impact: String(impact),
            assignment_group: assignmentGroup || this.defaults.assignmentGroup
        };

        if (additionalComments) {
            incidentData.comments = additionalComments;
        }

        const response = await this._request({
            method: 'POST',
            url: this._getTableEndpoint('incident'),
            data: incidentData
        });

        const created = response.result;

        return {
            success: true,
            id: created.sys_id,
            number: created.number,
            url: `${this.instanceUrl}/incident.do?sys_id=${created.sys_id}`,
            incident: this._normalizeIncident(created),
            provider: ITSMProvider.SERVICENOW
        };
    }

    /**
     * @override
     */
    async updateIncident(id, updateData) {
        this.logger.info('Updating incident', { id });

        const response = await this._request({
            method: 'PATCH',
            url: `${this._getTableEndpoint('incident')}/${id}`,
            data: updateData
        });

        return {
            success: true,
            incident: this._normalizeIncident(response.result),
            provider: ITSMProvider.SERVICENOW
        };
    }

    /**
     * @override
     */
    async addComment(id, comment, type = 'public') {
        this.logger.info('Adding comment', { id, type });

        // ServiceNow: 'comments' for public, 'work_notes' for internal
        const field = type === CommentType.INTERNAL ? 'work_notes' : 'comments';

        return this.updateIncident(id, { [field]: comment });
    }

    /**
     * @override
     */
    async updateState(id, state, options = {}) {
        const { closeCode, closeNotes } = options;

        this.logger.info('Updating state', { id, state });

        const snState = NORMALIZED_TO_SERVICENOW_STATE[state];
        if (!snState) {
            throw new Error(`Invalid state: ${state}`);
        }

        const updateData = { state: String(snState) };

        // Add close fields for resolution/closure
        if ([IncidentState.RESOLVED, IncidentState.CLOSED].includes(state)) {
            if (closeCode) updateData.close_code = closeCode;
            if (closeNotes) updateData.close_notes = closeNotes;
        }

        return this.updateIncident(id, updateData);
    }

    /**
     * @override
     */
    async validateUser(email) {
        if (!email) return null;

        this.logger.info('Validating user', { email });

        try {
            const response = await this._request({
                method: 'GET',
                url: this._getTableEndpoint('sys_user'),
                params: {
                    sysparm_query: `email=${encodeURIComponent(email)}`,
                    sysparm_limit: 1,
                    sysparm_fields: 'sys_id,email,name,user_name,active,department'
                }
            });

            const user = response.result?.[0];
            if (!user) return null;

            return {
                sys_id: user.sys_id,
                email: user.email,
                name: user.name,
                userName: user.user_name,
                active: user.active !== 'false',
                department: user.department?.display_value || user.department
            };
        } catch (error) {
            this.logger.error('User validation failed', { error: error.message });
            return null;
        }
    }

    // ============================================
    // PRIVATE HELPERS
    // ============================================

    /**
     * Normalize ServiceNow incident to common format
     * @private
     */
    _normalizeIncident(snIncident) {
        const stateCode = parseInt(snIncident.state?.value || snIncident.state);
        const priorityCode = parseInt(snIncident.priority?.value || snIncident.priority);

        return {
            id: snIncident.sys_id,
            number: snIncident.number,
            shortDescription: snIncident.short_description?.display_value || snIncident.short_description,
            description: snIncident.description?.display_value || snIncident.description,
            state: SERVICENOW_STATE_MAP[stateCode] || IncidentState.NEW,
            stateDisplay: snIncident.state?.display_value || 'Unknown',
            priority: SERVICENOW_PRIORITY_MAP[priorityCode] || IncidentPriority.MEDIUM,
            priorityDisplay: snIncident.priority?.display_value || 'Unknown',
            category: snIncident.category?.display_value || snIncident.category,
            subcategory: snIncident.subcategory?.display_value || snIncident.subcategory,
            caller: this._normalizeReference(snIncident.caller_id),
            assignee: this._normalizeReference(snIncident.assigned_to),
            assignmentGroup: this._normalizeReference(snIncident.assignment_group),
            createdAt: snIncident.sys_created_on?.display_value || snIncident.sys_created_on,
            updatedAt: snIncident.sys_updated_on?.display_value || snIncident.sys_updated_on,
            resolvedAt: snIncident.resolved_at?.display_value || snIncident.resolved_at,
            closedAt: snIncident.closed_at?.display_value || snIncident.closed_at,
            url: `${this.instanceUrl}/incident.do?sys_id=${snIncident.sys_id}`,
            provider: ITSMProvider.SERVICENOW,
            raw: snIncident
        };
    }

    /**
     * Normalize reference field
     * @private
     */
    _normalizeReference(field) {
        if (!field) return null;
        return {
            id: field.value || field.sys_id || field,
            name: field.display_value || field.name || 'Unknown'
        };
    }

    /**
     * Convert normalized priority to ServiceNow code
     * @private
     */
    _normalizedPriorityToServiceNow(priority) {
        const map = {
            [IncidentPriority.CRITICAL]: 1,
            [IncidentPriority.HIGH]: 2,
            [IncidentPriority.MEDIUM]: 3,
            [IncidentPriority.LOW]: 4,
            [IncidentPriority.PLANNING]: 5
        };
        return map[priority];
    }
}

module.exports = ServiceNowProvider;
```

---

### Step 4: Create Mock Provider (for Testing)

**File**: `src/Services/ITSM/providers/MockProvider.js`

```javascript
/**
 * Mock ITSM Provider
 *
 * In-memory mock provider for testing and development.
 *
 * @extends BaseITSMProvider
 * @module Services/ITSM/providers/MockProvider
 */

const BaseITSMProvider = require('./BaseITSMProvider');
const { ITSMProvider, IncidentState, IncidentPriority } = require('../types');

class MockProvider extends BaseITSMProvider {
    constructor(config = {}) {
        super({ ...config, type: ITSMProvider.MOCK, url: 'mock://localhost' });
        this.incidents = new Map();
        this.counter = 1000;
        this._seedData();
    }

    getProviderInfo() {
        return {
            type: ITSMProvider.MOCK,
            name: 'Mock Provider',
            version: '1.0.0',
            capabilities: ['incidents', 'comments', 'users'],
            ticketPrefix: 'MOCK'
        };
    }

    async getIncidents(options = {}) {
        const { callerEmail, state, limit = 10 } = options;
        let incidents = Array.from(this.incidents.values());

        if (callerEmail) {
            incidents = incidents.filter(i => i.caller?.email === callerEmail);
        }

        if (state === 'open') {
            incidents = incidents.filter(i =>
                ![IncidentState.RESOLVED, IncidentState.CLOSED, IncidentState.CANCELED].includes(i.state)
            );
        }

        return {
            success: true,
            incidents: incidents.slice(0, limit),
            total: incidents.length,
            limit,
            offset: 0,
            hasMore: incidents.length > limit,
            provider: ITSMProvider.MOCK
        };
    }

    async getIncident(identifier) {
        const incident = this.incidents.get(identifier) ||
            Array.from(this.incidents.values()).find(i => i.number === identifier);

        if (!incident) {
            return { success: true, found: false, provider: ITSMProvider.MOCK };
        }

        return {
            success: true,
            found: true,
            incident,
            availableActions: this._getAvailableActions(incident.state),
            provider: ITSMProvider.MOCK
        };
    }

    async createIncident(data) {
        const id = `mock-${++this.counter}`;
        const number = `MOCK${this.counter}`;

        const incident = {
            id,
            number,
            shortDescription: data.shortDescription,
            description: data.description,
            state: IncidentState.NEW,
            stateDisplay: 'New',
            priority: data.priority || IncidentPriority.MEDIUM,
            priorityDisplay: 'Medium',
            category: data.category,
            caller: { id: data.callerId, name: data.callerId, email: data.callerId },
            createdAt: new Date().toISOString(),
            updatedAt: new Date().toISOString(),
            url: `mock://localhost/incident/${id}`,
            provider: ITSMProvider.MOCK
        };

        this.incidents.set(id, incident);

        return {
            success: true,
            id,
            number,
            url: incident.url,
            incident,
            provider: ITSMProvider.MOCK
        };
    }

    async updateIncident(id, updateData) {
        const incident = this.incidents.get(id);
        if (!incident) throw new Error('Incident not found');

        Object.assign(incident, updateData, { updatedAt: new Date().toISOString() });
        return { success: true, incident, provider: ITSMProvider.MOCK };
    }

    async addComment(id, comment, type) {
        return this.updateIncident(id, {
            lastComment: { text: comment, type, timestamp: new Date().toISOString() }
        });
    }

    async updateState(id, state) {
        return this.updateIncident(id, { state, stateDisplay: state });
    }

    async validateUser(email) {
        return {
            sys_id: `mock-user-${email}`,
            email,
            name: email.split('@')[0],
            active: true
        };
    }

    _seedData() {
        // Add some sample incidents for testing
        const sampleIncident = {
            id: 'mock-1001',
            number: 'MOCK1001',
            shortDescription: 'Test incident for development',
            description: 'This is a mock incident for testing purposes.',
            state: IncidentState.NEW,
            stateDisplay: 'New',
            priority: IncidentPriority.MEDIUM,
            priorityDisplay: 'Medium',
            category: 'Inquiry',
            caller: { id: 'user@example.com', name: 'Test User', email: 'user@example.com' },
            createdAt: new Date().toISOString(),
            updatedAt: new Date().toISOString(),
            url: 'mock://localhost/incident/mock-1001',
            provider: ITSMProvider.MOCK
        };
        this.incidents.set('mock-1001', sampleIncident);
    }
}

module.exports = MockProvider;
```

---

### Step 5: Create ITSM Provider Factory

**File**: `src/Services/ITSM/ITSMProviderFactory.js`

```javascript
/**
 * ITSM Provider Factory
 *
 * Factory for creating and managing ITSM provider instances.
 * Supports multiple providers with configuration-based selection.
 *
 * @module Services/ITSM/ITSMProviderFactory
 * @see CB-012-ITSM-Integration-APIs
 */

const { createLogger } = require('../Logger');
const { ITSMProvider } = require('./types');

// Provider implementations
const ServiceNowProvider = require('./providers/ServiceNowProvider');
const MockProvider = require('./providers/MockProvider');
// Future: const JiraServiceProvider = require('./providers/JiraServiceProvider');

const logger = createLogger('ITSMProviderFactory');

/**
 * Registry of available providers
 */
const PROVIDER_REGISTRY = {
    [ITSMProvider.SERVICENOW]: ServiceNowProvider,
    [ITSMProvider.MOCK]: MockProvider,
    // Future providers:
    // [ITSMProvider.JIRA]: JiraServiceProvider,
    // [ITSMProvider.ZENDESK]: ZendeskProvider,
};

/**
 * Singleton provider instances (one per type)
 */
const providerInstances = new Map();

/**
 * Default/active provider type
 */
let defaultProviderType = null;

/**
 * Create provider from environment configuration
 *
 * @param {string} providerType - Provider type
 * @returns {import('./providers/BaseITSMProvider')}
 */
function createProviderFromEnv(providerType) {
    switch (providerType) {
        case ITSMProvider.SERVICENOW:
            return new ServiceNowProvider({
                url: process.env.SERVICENOW_URL,
                auth: {
                    username: process.env.SERVICENOW_USERNAME,
                    password: process.env.SERVICENOW_PASSWORD
                },
                timeout: parseInt(process.env.SERVICENOW_TIMEOUT) || 30000,
                defaults: {
                    category: process.env.SERVICENOW_DEFAULT_CATEGORY || 'Inquiry',
                    assignmentGroup: process.env.SERVICENOW_DEFAULT_ASSIGNMENT_GROUP
                }
            });

        case ITSMProvider.MOCK:
            return new MockProvider();

        // Future: case ITSMProvider.JIRA: ...

        default:
            throw new Error(`Unknown ITSM provider: ${providerType}`);
    }
}

/**
 * Get or create provider instance
 *
 * @param {string} [providerType] - Provider type (uses default if not specified)
 * @returns {import('./providers/BaseITSMProvider')}
 */
function getProvider(providerType) {
    const type = providerType || defaultProviderType || detectDefaultProvider();

    if (!providerInstances.has(type)) {
        logger.info(`Creating ITSM provider: ${type}`);
        const provider = createProviderFromEnv(type);
        providerInstances.set(type, provider);
    }

    return providerInstances.get(type);
}

/**
 * Detect default provider from environment
 *
 * @returns {string}
 */
function detectDefaultProvider() {
    // Check explicit configuration
    if (process.env.ITSM_PROVIDER) {
        defaultProviderType = process.env.ITSM_PROVIDER.toLowerCase();
        return defaultProviderType;
    }

    // Auto-detect based on configured credentials
    if (process.env.SERVICENOW_URL && process.env.SERVICENOW_USERNAME) {
        defaultProviderType = ITSMProvider.SERVICENOW;
        return defaultProviderType;
    }

    // Future: check for Jira, Zendesk, etc.

    // Fallback to mock for development
    if (process.env.NODE_ENV === 'development' || process.env.ENV === 'LOCAL') {
        logger.warn('No ITSM provider configured, using Mock provider');
        defaultProviderType = ITSMProvider.MOCK;
        return defaultProviderType;
    }

    throw new Error('No ITSM provider configured. Set ITSM_PROVIDER or configure provider credentials.');
}

/**
 * Set default provider type
 *
 * @param {string} providerType
 */
function setDefaultProvider(providerType) {
    if (!PROVIDER_REGISTRY[providerType]) {
        throw new Error(`Unknown provider: ${providerType}`);
    }
    defaultProviderType = providerType;
    logger.info(`Default ITSM provider set to: ${providerType}`);
}

/**
 * Get list of available providers
 *
 * @returns {string[]}
 */
function getAvailableProviders() {
    return Object.keys(PROVIDER_REGISTRY);
}

/**
 * Check if a provider is configured and ready
 *
 * @param {string} providerType
 * @returns {boolean}
 */
function isProviderConfigured(providerType) {
    try {
        const provider = getProvider(providerType);
        return provider.isConfigured();
    } catch {
        return false;
    }
}

/**
 * Clear all provider instances (useful for testing)
 */
function clearProviders() {
    providerInstances.clear();
    defaultProviderType = null;
}

module.exports = {
    getProvider,
    setDefaultProvider,
    getAvailableProviders,
    isProviderConfigured,
    clearProviders,
    createProviderFromEnv,
    ITSMProvider
};
```

---

### Step 6: Create IncidentService

**File**: `src/Services/ITSM/IncidentService.js`

High-level orchestration service that uses the provider factory for provider-agnostic operations.

```javascript
/**
 * Incident Service
 *
 * Orchestrates incident management operations with business logic,
 * validation, and formatting. Provider-agnostic - works with any
 * configured ITSM provider (ServiceNow, Jira, Zendesk, etc.).
 *
 * @module Services/ITSM/IncidentService
 * @see CB-012-ITSM-Integration-APIs
 */

const { createLogger } = require('../Logger');
const { getProvider } = require('./ITSMProviderFactory');
const { IncidentState, CommentType } = require('./types');
const { formatIncidentAsText } = require('../../utils/itsmFormatter');

const logger = createLogger('IncidentService');

class IncidentService {
    /**
     * Get the active ITSM provider
     * @private
     */
    _getProvider() {
        return getProvider();
    }

    /**
     * Get incidents for a user
     *
     * @param {Object} params - Query parameters
     * @param {string} [params.callerEmail] - Caller email
     * @param {string} [params.state='open'] - Filter state
     * @param {string} [params.priority] - Priority filter
     * @param {number} [params.limit=10] - Max results
     * @param {number} [params.offset=0] - Pagination offset
     * @param {string} [params.orderBy] - Sort field
     * @param {string} [params.orderDir] - Sort direction
     * @returns {Promise<Object>} Incident list result
     */
    async getIncidents(params = {}) {
        const provider = this._getProvider();

        logger.info('Getting incidents', {
            provider: provider.providerType,
            state: params.state,
            hasCallerFilter: !!params.callerEmail
        });

        const result = await provider.getIncidents(params);

        return {
            ...result,
            providerInfo: provider.getProviderInfo()
        };
    }

    /**
     * Get incident details
     *
     * @param {string} identifier - Incident number or ID
     * @param {Object} [options] - Options
     * @returns {Promise<Object>} Incident result
     */
    async getIncidentDetails(identifier, options = {}) {
        const provider = this._getProvider();

        logger.info('Getting incident details', {
            identifier,
            provider: provider.providerType
        });

        const result = await provider.getIncident(identifier, options);

        if (result.found && result.incident) {
            result.formattedText = formatIncidentAsText(result.incident);
        }

        return {
            ...result,
            providerInfo: provider.getProviderInfo()
        };
    }

    /**
     * Create new incident
     *
     * @param {Object} incidentData - Incident data
     * @returns {Promise<Object>} Created incident result
     */
    async createIncident(incidentData) {
        const provider = this._getProvider();

        logger.info('Creating incident', {
            provider: provider.providerType,
            hasDescription: !!incidentData.description
        });

        // Validate required fields
        if (!incidentData.shortDescription) {
            throw new Error('shortDescription is required');
        }
        if (!incidentData.callerId) {
            throw new Error('callerId is required');
        }

        const result = await provider.createIncident(incidentData);

        logger.info('Incident created', {
            number: result.number,
            provider: provider.providerType
        });

        return {
            ...result,
            providerInfo: provider.getProviderInfo()
        };
    }

    /**
     * Add comment to incident
     *
     * @param {string} identifier - Incident number or ID
     * @param {string} comment - Comment text
     * @param {Object} [options] - Options
     * @param {string} [options.type='public'] - Comment type
     * @returns {Promise<Object>} Result
     */
    async addComment(identifier, comment, options = {}) {
        const provider = this._getProvider();

        logger.info('Adding comment', {
            identifier,
            type: options.type || 'public',
            provider: provider.providerType
        });

        // First, get the incident to obtain its ID
        const incidentResult = await provider.getIncident(identifier);

        if (!incidentResult.found) {
            throw new Error(`Incident ${identifier} not found`);
        }

        const result = await provider.addComment(
            incidentResult.incident.id,
            comment,
            options.type || CommentType.PUBLIC
        );

        return {
            ...result,
            incidentNumber: incidentResult.incident.number,
            providerInfo: provider.getProviderInfo()
        };
    }

    /**
     * Update incident state
     *
     * @param {string} identifier - Incident number or ID
     * @param {string} state - New state (normalized)
     * @param {Object} [options] - Additional options
     * @returns {Promise<Object>} Result
     */
    async updateState(identifier, state, options = {}) {
        const provider = this._getProvider();

        logger.info('Updating state', {
            identifier,
            state,
            provider: provider.providerType
        });

        // Get incident ID
        const incidentResult = await provider.getIncident(identifier);

        if (!incidentResult.found) {
            throw new Error(`Incident ${identifier} not found`);
        }

        // Validate state transitions for closure
        if ([IncidentState.RESOLVED, IncidentState.CLOSED].includes(state)) {
            if (!options.closeNotes) {
                throw new Error('closeNotes is required when resolving or closing');
            }
        }

        const result = await provider.updateState(
            incidentResult.incident.id,
            state,
            options
        );

        return {
            ...result,
            incidentNumber: incidentResult.incident.number,
            previousState: incidentResult.incident.state,
            newState: state,
            providerInfo: provider.getProviderInfo()
        };
    }

    /**
     * Cancel incident
     *
     * @param {string} identifier - Incident number or ID
     * @param {string} [reason] - Cancellation reason
     * @returns {Promise<Object>} Result
     */
    async cancelIncident(identifier, reason) {
        const provider = this._getProvider();

        logger.info('Canceling incident', { identifier });

        // Get incident ID
        const incidentResult = await provider.getIncident(identifier);

        if (!incidentResult.found) {
            throw new Error(`Incident ${identifier} not found`);
        }

        // Add cancellation comment first
        if (reason) {
            await provider.addComment(
                incidentResult.incident.id,
                `Incident canceled: ${reason}`,
                CommentType.INTERNAL
            );
        }

        // Update state to canceled
        const result = await provider.updateState(
            incidentResult.incident.id,
            IncidentState.CANCELED,
            { closeNotes: reason || 'Canceled by user via chatbot' }
        );

        return {
            ...result,
            incidentNumber: incidentResult.incident.number,
            providerInfo: provider.getProviderInfo()
        };
    }

    /**
     * Escalate incident
     *
     * @param {string} identifier - Incident number or ID
     * @param {Object} [options] - Escalation options
     * @returns {Promise<Object>} Result
     */
    async escalateIncident(identifier, options = {}) {
        const provider = this._getProvider();

        logger.info('Escalating incident', { identifier });

        // Get incident ID
        const incidentResult = await provider.getIncident(identifier);

        if (!incidentResult.found) {
            throw new Error(`Incident ${identifier} not found`);
        }

        const result = await provider.escalateIncident(
            incidentResult.incident.id,
            options
        );

        return {
            ...result,
            incidentNumber: incidentResult.incident.number,
            providerInfo: provider.getProviderInfo()
        };
    }

    /**
     * Health check
     *
     * @returns {Promise<Object>}
     */
    async healthCheck() {
        const provider = this._getProvider();
        return provider.healthCheck();
    }

    /**
     * Get provider information
     *
     * @returns {Object}
     */
    getProviderInfo() {
        return this._getProvider().getProviderInfo();
    }
}

// Export singleton
module.exports = new IncidentService();
```

---

### Step 3: Create ITSM Formatter Utility

**File**: `src/utils/itsmFormatter.js`

```javascript
/**
 * ITSM Formatter Utilities
 *
 * Formats incident data for display and API responses.
 *
 * @module utils/itsmFormatter
 */

/**
 * State display mapping
 */
const STATE_DISPLAY = {
    1: { label: 'New', color: 'attention', icon: '🆕' },
    2: { label: 'In Progress', color: 'accent', icon: '🔄' },
    3: { label: 'On Hold', color: 'warning', icon: '⏸️' },
    6: { label: 'Resolved', color: 'good', icon: '✅' },
    7: { label: 'Closed', color: 'good', icon: '🔒' },
    8: { label: 'Canceled', color: 'default', icon: '❌' }
};

/**
 * Priority display mapping
 */
const PRIORITY_DISPLAY = {
    1: { label: 'Critical', color: 'attention' },
    2: { label: 'High', color: 'warning' },
    3: { label: 'Moderate', color: 'accent' },
    4: { label: 'Low', color: 'good' },
    5: { label: 'Planning', color: 'default' }
};

/**
 * Format single incident for display
 *
 * @param {Object} incident - Raw incident data from ServiceNow
 * @returns {Object} Formatted incident
 */
function formatIncidentForDisplay(incident) {
    if (!incident) return null;

    const stateCode = parseInt(incident.state);
    const priorityCode = parseInt(incident.priority);

    return {
        // Identifiers
        sysId: incident.sys_id,
        number: incident.number,

        // Status
        state: {
            code: stateCode,
            ...STATE_DISPLAY[stateCode] || { label: 'Unknown', color: 'default' }
        },
        priority: {
            code: priorityCode,
            ...PRIORITY_DISPLAY[priorityCode] || { label: 'Unknown', color: 'default' }
        },

        // Content
        shortDescription: incident.short_description,
        description: incident.description,
        category: incident.category,
        subcategory: incident.subcategory,

        // People
        caller: formatUserReference(incident.caller_id),
        assignedTo: formatUserReference(incident.assigned_to),
        assignmentGroup: formatReference(incident.assignment_group),

        // Dates
        createdOn: incident.sys_created_on,
        updatedOn: incident.sys_updated_on,
        resolvedAt: incident.resolved_at,
        closedAt: incident.closed_at,

        // Display helpers
        createdOnDisplay: formatDateTime(incident.sys_created_on),
        updatedOnDisplay: formatDateTime(incident.sys_updated_on),

        // ServiceNow URL
        url: `${process.env.SERVICENOW_URL}/incident.do?sys_id=${incident.sys_id}`
    };
}

/**
 * Format incident list for display
 *
 * @param {Array} incidents - Array of raw incidents
 * @returns {Array} Formatted incidents
 */
function formatIncidentList(incidents) {
    return incidents.map(inc => ({
        sysId: inc.sys_id,
        number: inc.number,
        shortDescription: inc.short_description,
        state: STATE_DISPLAY[parseInt(inc.state)] || { label: 'Unknown' },
        priority: PRIORITY_DISPLAY[parseInt(inc.priority)] || { label: 'Unknown' },
        createdOnDisplay: formatDateTime(inc.sys_created_on),
        isOpen: ![6, 7, 8].includes(parseInt(inc.state))
    }));
}

/**
 * Format incident for text display (chat message)
 *
 * @param {Object} incident - Raw incident data
 * @returns {string} Formatted text
 */
function formatIncidentAsText(incident) {
    const formatted = formatIncidentForDisplay(incident);
    if (!formatted) return 'Incident not found.';

    return `**${formatted.number}** - ${formatted.shortDescription}

📋 **Status**: ${formatted.state.icon} ${formatted.state.label}
🔴 **Priority**: ${formatted.priority.label}
📁 **Category**: ${formatted.category || 'Not specified'}
📅 **Created**: ${formatted.createdOnDisplay}
👤 **Assigned To**: ${formatted.assignedTo?.displayValue || 'Unassigned'}

🔗 [View in ServiceNow](${formatted.url})`;
}

/**
 * Format user reference field
 */
function formatUserReference(field) {
    if (!field) return null;
    if (typeof field === 'string') return { sysId: field, displayValue: field };
    return {
        sysId: field.value || field.sys_id,
        displayValue: field.display_value || field.name || 'Unknown'
    };
}

/**
 * Format generic reference field
 */
function formatReference(field) {
    if (!field) return null;
    if (typeof field === 'string') return { value: field, displayValue: field };
    return {
        value: field.value || field.sys_id,
        displayValue: field.display_value || 'Unknown'
    };
}

/**
 * Format datetime for display
 */
function formatDateTime(dateString) {
    if (!dateString) return null;
    try {
        const date = new Date(dateString);
        return date.toLocaleDateString('en-US', {
            year: 'numeric',
            month: 'short',
            day: 'numeric',
            hour: '2-digit',
            minute: '2-digit'
        });
    } catch {
        return dateString;
    }
}

module.exports = {
    STATE_DISPLAY,
    PRIORITY_DISPLAY,
    formatIncidentForDisplay,
    formatIncidentList,
    formatIncidentAsText,
    formatUserReference,
    formatReference,
    formatDateTime
};
```

---

### Step 4: Create ITSM Controller

**File**: `src/controllers/itsm.controller.js`

```javascript
/**
 * ITSM Controller
 *
 * Handles HTTP requests for ServiceNow ITSM operations.
 * Follows the same pattern as drug.controller.js and medical.controller.js.
 *
 * @module controllers/itsm
 * @see CB-012-ITSM-Integration-APIs
 */

const { createLogger } = require('../Services/Logger');
const IncidentService = require('../Services/IncidentService');
const { formatIncidentAsText } = require('../utils/itsmFormatter');

const logger = createLogger('ITSMController');

/**
 * Get list of incidents
 *
 * GET /api/itsm/incidents
 */
async function getIncidents(req, res) {
    const startTime = Date.now();

    try {
        const {
            callerId,
            state = 'open',
            priority,
            limit = 10,
            offset = 0,
            orderBy = 'sys_created_on',
            orderDir = 'desc'
        } = req.query;

        logger.info('Get incidents request', {
            callerId: callerId ? '***' : undefined,
            state,
            priority,
            limit
        });

        const result = await IncidentService.getIncidents({
            callerEmail: callerId,
            state,
            priority,
            limit: Math.min(parseInt(limit), 50),
            offset: parseInt(offset),
            orderBy,
            orderDir
        });

        return res.status(200).json({
            success: true,
            ...result,
            requestId: req.id || `itsm-${Date.now()}`,
            responseTimeMs: Date.now() - startTime
        });

    } catch (error) {
        logger.error('Get incidents error', { error: error.message });
        return handleError(res, error, startTime);
    }
}

/**
 * Get incident by identifier
 *
 * GET /api/itsm/incidents/:identifier
 */
async function getIncident(req, res) {
    const startTime = Date.now();

    try {
        const { identifier } = req.params;
        const { includeComments, includeHistory, format } = req.query;

        if (!identifier) {
            return res.status(400).json({
                success: false,
                error: 'Incident identifier is required',
                code: 'MISSING_IDENTIFIER'
            });
        }

        logger.info('Get incident request', { identifier });

        const result = await IncidentService.getIncidentDetails(identifier, {
            includeComments: includeComments === 'true',
            includeHistory: includeHistory === 'true'
        });

        if (!result.found) {
            return res.status(404).json({
                success: false,
                error: `Incident ${identifier} not found`,
                code: 'INCIDENT_NOT_FOUND'
            });
        }

        // Optional text format for chat display
        if (format === 'text') {
            result.formatted = formatIncidentAsText(result.incident);
        }

        return res.status(200).json({
            success: true,
            ...result,
            requestId: req.id || `itsm-${Date.now()}`,
            responseTimeMs: Date.now() - startTime
        });

    } catch (error) {
        logger.error('Get incident error', { error: error.message });
        return handleError(res, error, startTime);
    }
}

/**
 * Create new incident
 *
 * POST /api/itsm/incidents
 */
async function createIncident(req, res) {
    const startTime = Date.now();

    try {
        const {
            shortDescription,
            description,
            callerId,
            category,
            subcategory,
            urgency,
            impact,
            assignmentGroup,
            additionalComments,
            configurationItem
        } = req.body;

        // Validate required fields
        if (!shortDescription) {
            return res.status(400).json({
                success: false,
                error: 'shortDescription is required',
                code: 'MISSING_SHORT_DESCRIPTION'
            });
        }

        if (!callerId) {
            return res.status(400).json({
                success: false,
                error: 'callerId (email or sys_id) is required',
                code: 'MISSING_CALLER_ID'
            });
        }

        logger.info('Create incident request', {
            shortDescription: shortDescription.substring(0, 50) + '...',
            callerId: '***'
        });

        const result = await IncidentService.createIncident({
            shortDescription,
            description,
            callerId,
            category,
            subcategory,
            urgency: parseInt(urgency) || 3,
            impact: parseInt(impact) || 3,
            assignmentGroup,
            additionalComments,
            configurationItem
        });

        return res.status(201).json({
            success: true,
            ...result,
            requestId: req.id || `itsm-${Date.now()}`,
            responseTimeMs: Date.now() - startTime
        });

    } catch (error) {
        logger.error('Create incident error', { error: error.message });
        return handleError(res, error, startTime);
    }
}

/**
 * Add comment to incident
 *
 * POST /api/itsm/incidents/:identifier/comments
 */
async function addComment(req, res) {
    const startTime = Date.now();

    try {
        const { identifier } = req.params;
        const { comment, type = 'comments' } = req.body;

        if (!identifier) {
            return res.status(400).json({
                success: false,
                error: 'Incident identifier is required',
                code: 'MISSING_IDENTIFIER'
            });
        }

        if (!comment) {
            return res.status(400).json({
                success: false,
                error: 'Comment text is required',
                code: 'MISSING_COMMENT'
            });
        }

        // Validate comment type
        if (!['comments', 'work_notes'].includes(type)) {
            return res.status(400).json({
                success: false,
                error: 'Invalid comment type. Must be "comments" or "work_notes"',
                code: 'INVALID_COMMENT_TYPE'
            });
        }

        logger.info('Add comment request', { identifier, type });

        const result = await IncidentService.addComment(identifier, comment, { type });

        return res.status(200).json({
            success: true,
            message: 'Comment added successfully',
            ...result,
            requestId: req.id || `itsm-${Date.now()}`,
            responseTimeMs: Date.now() - startTime
        });

    } catch (error) {
        logger.error('Add comment error', { error: error.message });
        return handleError(res, error, startTime);
    }
}

/**
 * Update incident state
 *
 * PATCH /api/itsm/incidents/:identifier/state
 */
async function updateState(req, res) {
    const startTime = Date.now();

    try {
        const { identifier } = req.params;
        const { state, closeCode, closeNotes, reason } = req.body;

        if (!identifier) {
            return res.status(400).json({
                success: false,
                error: 'Incident identifier is required',
                code: 'MISSING_IDENTIFIER'
            });
        }

        if (state === undefined) {
            return res.status(400).json({
                success: false,
                error: 'State is required',
                code: 'MISSING_STATE'
            });
        }

        // Validate state code
        const validStates = [1, 2, 3, 6, 7, 8];
        const stateInt = parseInt(state);
        if (!validStates.includes(stateInt)) {
            return res.status(400).json({
                success: false,
                error: 'Invalid state code. Valid: 1(New), 2(In Progress), 3(On Hold), 6(Resolved), 7(Closed), 8(Canceled)',
                code: 'INVALID_STATE'
            });
        }

        // Require close notes for closure states
        if ([6, 7].includes(stateInt) && !closeNotes) {
            return res.status(400).json({
                success: false,
                error: 'closeNotes is required when resolving or closing an incident',
                code: 'MISSING_CLOSE_NOTES'
            });
        }

        logger.info('Update state request', { identifier, state: stateInt });

        const result = await IncidentService.updateState(identifier, stateInt, {
            closeCode,
            closeNotes,
            reason
        });

        return res.status(200).json({
            success: true,
            message: `Incident state updated to ${stateInt}`,
            ...result,
            requestId: req.id || `itsm-${Date.now()}`,
            responseTimeMs: Date.now() - startTime
        });

    } catch (error) {
        logger.error('Update state error', { error: error.message });
        return handleError(res, error, startTime);
    }
}

/**
 * Escalate incident
 *
 * POST /api/itsm/incidents/:identifier/escalate
 */
async function escalateIncident(req, res) {
    const startTime = Date.now();

    try {
        const { identifier } = req.params;
        const {
            reason = 'User requested escalation via chatbot',
            increasePriority = false
        } = req.body;

        if (!identifier) {
            return res.status(400).json({
                success: false,
                error: 'Incident identifier is required',
                code: 'MISSING_IDENTIFIER'
            });
        }

        logger.info('Escalate incident request', { identifier, increasePriority });

        const result = await IncidentService.escalateIncident(identifier, {
            reason,
            increasePriority
        });

        return res.status(200).json({
            success: true,
            message: 'Incident escalated successfully',
            ...result,
            requestId: req.id || `itsm-${Date.now()}`,
            responseTimeMs: Date.now() - startTime
        });

    } catch (error) {
        logger.error('Escalate incident error', { error: error.message });
        return handleError(res, error, startTime);
    }
}

/**
 * ITSM health check
 *
 * GET /api/itsm/health
 */
async function healthCheck(req, res) {
    try {
        const { serviceNowService } = require('../Services/ServiceNowService');

        const startTime = Date.now();
        let connected = false;
        let error = null;

        try {
            // Test connection by getting a single incident
            await serviceNowService.searchIncidents({ limit: 1 });
            connected = true;
        } catch (err) {
            error = err.message;
        }

        const latencyMs = Date.now() - startTime;
        const isHealthy = connected && latencyMs < 5000;

        return res.status(isHealthy ? 200 : 503).json({
            status: isHealthy ? 'healthy' : connected ? 'degraded' : 'unhealthy',
            service: 'itsm',
            servicenow: {
                connected,
                configured: serviceNowService.isConfigured,
                instanceUrl: process.env.SERVICENOW_URL || 'not configured',
                latencyMs,
                error
            },
            timestamp: new Date().toISOString()
        });

    } catch (error) {
        logger.error('ITSM health check failed', { error: error.message });

        return res.status(503).json({
            status: 'unhealthy',
            service: 'itsm',
            error: error.message,
            timestamp: new Date().toISOString()
        });
    }
}

/**
 * Handle controller errors
 */
function handleError(res, error, startTime) {
    const statusCode = error.statusCode || 500;
    const errorCode = error.code || 'ITSM_ERROR';

    return res.status(statusCode).json({
        success: false,
        error: error.message || 'Internal server error',
        code: errorCode,
        details: process.env.ENV === 'LOCAL' ? error.stack : undefined,
        responseTimeMs: Date.now() - startTime
    });
}

module.exports = {
    getIncidents,
    getIncident,
    createIncident,
    addComment,
    updateState,
    escalateIncident,
    healthCheck
};
```

---

### Step 5: Create ITSM Routes

**File**: `src/routes/itsm.routes.js`

```javascript
/**
 * ITSM Routes
 *
 * API endpoints for ServiceNow ITSM operations.
 * These endpoints are designed to be called by LangFlow flows.
 *
 * @module routes/itsm
 * @see CB-012-ITSM-Integration-APIs
 */

const express = require('express');
const itsmController = require('../controllers/itsm.controller');

const router = express.Router();

/**
 * @swagger
 * /api/itsm/incidents:
 *   get:
 *     summary: Get list of incidents
 *     description: |
 *       Retrieve incidents filtered by caller, state, priority.
 *       Designed for LangFlow integration.
 *     tags: [ITSM]
 *     parameters:
 *       - in: query
 *         name: callerId
 *         schema:
 *           type: string
 *         description: Caller email or sys_id
 *       - in: query
 *         name: state
 *         schema:
 *           type: string
 *           enum: [open, closed, all]
 *           default: open
 *         description: Filter by state
 *       - in: query
 *         name: priority
 *         schema:
 *           type: integer
 *           minimum: 1
 *           maximum: 5
 *         description: Filter by priority
 *       - in: query
 *         name: limit
 *         schema:
 *           type: integer
 *           default: 10
 *           maximum: 50
 *         description: Max results
 *       - in: query
 *         name: orderBy
 *         schema:
 *           type: string
 *           default: sys_created_on
 *         description: Sort field
 *       - in: query
 *         name: orderDir
 *         schema:
 *           type: string
 *           enum: [asc, desc]
 *           default: desc
 *         description: Sort direction
 *     responses:
 *       200:
 *         description: List of incidents
 */
router.get('/api/itsm/incidents', itsmController.getIncidents);

/**
 * @swagger
 * /api/itsm/incidents/{identifier}:
 *   get:
 *     summary: Get incident by identifier
 *     description: Retrieve incident by number (INCxxxxxxx) or sys_id
 *     tags: [ITSM]
 *     parameters:
 *       - in: path
 *         name: identifier
 *         required: true
 *         schema:
 *           type: string
 *         description: Incident number or sys_id
 *       - in: query
 *         name: includeComments
 *         schema:
 *           type: boolean
 *           default: false
 *         description: Include comments
 *       - in: query
 *         name: includeHistory
 *         schema:
 *           type: boolean
 *           default: false
 *         description: Include history
 *       - in: query
 *         name: format
 *         schema:
 *           type: string
 *           enum: [json, text]
 *           default: json
 *         description: Response format (text for chat display)
 *     responses:
 *       200:
 *         description: Incident details
 *       404:
 *         description: Incident not found
 */
router.get('/api/itsm/incidents/:identifier', itsmController.getIncident);

/**
 * @swagger
 * /api/itsm/incidents:
 *   post:
 *     summary: Create new incident
 *     description: Create a new ServiceNow incident
 *     tags: [ITSM]
 *     requestBody:
 *       required: true
 *       content:
 *         application/json:
 *           schema:
 *             type: object
 *             required:
 *               - shortDescription
 *               - callerId
 *             properties:
 *               shortDescription:
 *                 type: string
 *                 description: Brief description
 *               description:
 *                 type: string
 *                 description: Full description
 *               callerId:
 *                 type: string
 *                 description: Caller email or sys_id
 *               category:
 *                 type: string
 *               subcategory:
 *                 type: string
 *               urgency:
 *                 type: integer
 *                 minimum: 1
 *                 maximum: 3
 *                 default: 3
 *               impact:
 *                 type: integer
 *                 minimum: 1
 *                 maximum: 3
 *                 default: 3
 *               assignmentGroup:
 *                 type: string
 *               additionalComments:
 *                 type: string
 *     responses:
 *       201:
 *         description: Incident created
 *       400:
 *         description: Invalid request
 */
router.post('/api/itsm/incidents', itsmController.createIncident);

/**
 * @swagger
 * /api/itsm/incidents/{identifier}/comments:
 *   post:
 *     summary: Add comment to incident
 *     description: Add a comment or work note to existing incident
 *     tags: [ITSM]
 *     parameters:
 *       - in: path
 *         name: identifier
 *         required: true
 *         schema:
 *           type: string
 *         description: Incident number or sys_id
 *     requestBody:
 *       required: true
 *       content:
 *         application/json:
 *           schema:
 *             type: object
 *             required:
 *               - comment
 *             properties:
 *               comment:
 *                 type: string
 *                 description: Comment text
 *               type:
 *                 type: string
 *                 enum: [comments, work_notes]
 *                 default: comments
 *                 description: Comment type
 *     responses:
 *       200:
 *         description: Comment added
 *       404:
 *         description: Incident not found
 */
router.post('/api/itsm/incidents/:identifier/comments', itsmController.addComment);

/**
 * @swagger
 * /api/itsm/incidents/{identifier}/state:
 *   patch:
 *     summary: Update incident state
 *     description: Update the state of an incident (cancel, resolve, etc.)
 *     tags: [ITSM]
 *     parameters:
 *       - in: path
 *         name: identifier
 *         required: true
 *         schema:
 *           type: string
 *         description: Incident number or sys_id
 *     requestBody:
 *       required: true
 *       content:
 *         application/json:
 *           schema:
 *             type: object
 *             required:
 *               - state
 *             properties:
 *               state:
 *                 type: integer
 *                 description: State code (1=New, 2=In Progress, 3=On Hold, 6=Resolved, 7=Closed, 8=Canceled)
 *               closeCode:
 *                 type: string
 *                 description: Close code (required for resolved/closed)
 *               closeNotes:
 *                 type: string
 *                 description: Close notes (required for resolved/closed)
 *               reason:
 *                 type: string
 *                 description: Reason for state change
 *     responses:
 *       200:
 *         description: State updated
 *       400:
 *         description: Invalid state or missing close notes
 */
router.patch('/api/itsm/incidents/:identifier/state', itsmController.updateState);

/**
 * @swagger
 * /api/itsm/incidents/{identifier}/escalate:
 *   post:
 *     summary: Escalate incident
 *     description: Escalate an incident by adding escalation comment
 *     tags: [ITSM]
 *     parameters:
 *       - in: path
 *         name: identifier
 *         required: true
 *         schema:
 *           type: string
 *         description: Incident number or sys_id
 *     requestBody:
 *       content:
 *         application/json:
 *           schema:
 *             type: object
 *             properties:
 *               reason:
 *                 type: string
 *                 default: User requested escalation via chatbot
 *                 description: Escalation reason
 *               increasePriority:
 *                 type: boolean
 *                 default: false
 *                 description: Whether to increase priority
 *     responses:
 *       200:
 *         description: Incident escalated
 *       404:
 *         description: Incident not found
 */
router.post('/api/itsm/incidents/:identifier/escalate', itsmController.escalateIncident);

/**
 * @swagger
 * /api/itsm/health:
 *   get:
 *     summary: ITSM service health check
 *     description: Check ServiceNow connectivity and service health
 *     tags: [ITSM]
 *     responses:
 *       200:
 *         description: Service healthy
 *       503:
 *         description: Service unhealthy
 */
router.get('/api/itsm/health', itsmController.healthCheck);

module.exports = router;
```

---

### Step 6: Update Routes Index

**File**: `src/routes/index.js`

Add ITSM routes registration:

```javascript
// Add import
const itsmRoutes = require('./itsm.routes');

// Add registration in registerRoutes function
// ITSM routes (CB-012)
app.use(itsmRoutes);

// Add export
module.exports = {
    // ... existing exports
    itsmRoutes
};
```

---

## API Reference Summary

### Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/api/itsm/incidents` | Get list of incidents |
| `GET` | `/api/itsm/incidents/:identifier` | Get incident details |
| `POST` | `/api/itsm/incidents` | Create new incident |
| `POST` | `/api/itsm/incidents/:identifier/comments` | Add comment |
| `PATCH` | `/api/itsm/incidents/:identifier/state` | Update state |
| `POST` | `/api/itsm/incidents/:identifier/escalate` | Escalate incident |
| `GET` | `/api/itsm/health` | Health check |

### Response Format

All endpoints follow consistent response format:

```json
{
  "success": true,
  "data": { /* result data */ },
  "requestId": "itsm-1234567890",
  "responseTimeMs": 245
}
```

Error response:

```json
{
  "success": false,
  "error": "Error message",
  "code": "ERROR_CODE",
  "responseTimeMs": 100
}
```

---

## Environment Variables

Add the following to `.env.example`:

```bash
# ===========================================
# ITSM PROVIDER CONFIGURATION
# ===========================================

# Active ITSM Provider (servicenow, jira, zendesk, mock)
# If not set, auto-detects based on configured credentials
ITSM_PROVIDER=servicenow

# ===========================================
# SERVICENOW CONFIGURATION
# ===========================================
SERVICENOW_URL=https://your-instance.service-now.com
SERVICENOW_USERNAME=integration_user
SERVICENOW_PASSWORD=integration_password
SERVICENOW_TIMEOUT=30000
SERVICENOW_DEFAULT_ASSIGNMENT_GROUP=IT Support
SERVICENOW_DEFAULT_CATEGORY=Inquiry

# ===========================================
# JIRA SERVICE MANAGEMENT (Phase 2)
# ===========================================
# JIRA_URL=https://your-domain.atlassian.net
# JIRA_EMAIL=integration@example.com
# JIRA_API_TOKEN=your-api-token
# JIRA_PROJECT_KEY=SUPPORT
# JIRA_ISSUE_TYPE=Service Request

# ===========================================
# ZENDESK (Future)
# ===========================================
# ZENDESK_URL=https://your-subdomain.zendesk.com
# ZENDESK_EMAIL=integration@example.com
# ZENDESK_API_TOKEN=your-api-token

# ===========================================
# FRESHSERVICE (Future)
# ===========================================
# FRESHSERVICE_URL=https://your-domain.freshservice.com
# FRESHSERVICE_API_KEY=your-api-key
```

---

## Adding New ITSM Providers

### Step-by-Step Guide for Adding a New Provider

To add support for a new ITSM system (e.g., Jira, Zendesk):

#### 1. Create Provider Class

Create `src/Services/ITSM/providers/<ProviderName>Provider.js`:

```javascript
const BaseITSMProvider = require('./BaseITSMProvider');
const { ITSMProvider, IncidentState, IncidentPriority } = require('../types');

// Define state mappings for your provider
const STATE_MAP = {
    // Map provider-specific states to normalized states
    'open': IncidentState.NEW,
    'in progress': IncidentState.IN_PROGRESS,
    // ... etc
};

class NewProvider extends BaseITSMProvider {
    constructor(config) {
        super({ ...config, type: ITSMProvider.NEW_PROVIDER });
    }

    getProviderInfo() {
        return {
            type: ITSMProvider.NEW_PROVIDER,
            name: 'New Provider Name',
            version: 'API Version',
            capabilities: ['incidents', 'comments'],
            ticketPrefix: 'PREFIX'
        };
    }

    // Implement all abstract methods:
    async getIncidents(options) { /* ... */ }
    async getIncident(identifier) { /* ... */ }
    async createIncident(data) { /* ... */ }
    async updateIncident(id, data) { /* ... */ }
    async addComment(id, comment, type) { /* ... */ }
    async updateState(id, state, options) { /* ... */ }
    async validateUser(email) { /* ... */ }

    // Provider-specific normalization
    _normalizeIncident(rawIncident) {
        return {
            id: rawIncident.id,
            number: rawIncident.key,
            shortDescription: rawIncident.summary,
            state: STATE_MAP[rawIncident.status] || IncidentState.NEW,
            // ... map all fields to NormalizedIncident
        };
    }
}

module.exports = NewProvider;
```

#### 2. Register in Provider Factory

Update `ITSMProviderFactory.js`:

```javascript
// Add import
const NewProvider = require('./providers/NewProvider');

// Add to registry
const PROVIDER_REGISTRY = {
    // ...existing providers
    [ITSMProvider.NEW_PROVIDER]: NewProvider,
};

// Add to createProviderFromEnv
case ITSMProvider.NEW_PROVIDER:
    return new NewProvider({
        url: process.env.NEW_PROVIDER_URL,
        auth: {
            apiKey: process.env.NEW_PROVIDER_API_KEY
        },
        timeout: parseInt(process.env.NEW_PROVIDER_TIMEOUT) || 30000
    });
```

#### 3. Add Provider Type

Update `types.js`:

```javascript
const ITSMProvider = {
    // ...existing
    NEW_PROVIDER: 'new_provider'
};
```

#### 4. Add Environment Variables

Document required environment variables for the new provider.

#### 5. Write Tests

Create provider-specific tests in the test directory.

### Provider Implementation Checklist

- [ ] Create provider class extending `BaseITSMProvider`
- [ ] Implement `getProviderInfo()`
- [ ] Implement `getIncidents()` with filtering
- [ ] Implement `getIncident()` with identifier resolution
- [ ] Implement `createIncident()` with field mapping
- [ ] Implement `updateIncident()`
- [ ] Implement `addComment()` (public/internal)
- [ ] Implement `updateState()` with state mapping
- [ ] Implement `validateUser()` if supported
- [ ] Create state mapping (provider → normalized)
- [ ] Create priority mapping
- [ ] Create `_normalizeIncident()` helper
- [ ] Register in factory
- [ ] Add environment variable support
- [ ] Document configuration
- [ ] Write unit tests

---

## Testing Strategy

### Unit Tests

1. **ServiceNowService tests**:
   - `searchIncidents()` - query building, pagination
   - `getIncident()` - by number, by sys_id
   - `createIncident()` - field mapping, validation
   - `updateIncident()` - state updates, comments

2. **IncidentService tests**:
   - Validation logic
   - Error handling
   - Response formatting

3. **Controller tests**:
   - Request validation
   - Error responses
   - Success responses

### Integration Tests

1. **API endpoint tests**:
   - GET /api/itsm/incidents - various filters
   - GET /api/itsm/incidents/:identifier - valid/invalid
   - POST /api/itsm/incidents - success/validation errors
   - POST /api/itsm/incidents/:identifier/comments
   - PATCH /api/itsm/incidents/:identifier/state
   - POST /api/itsm/incidents/:identifier/escalate

### Manual Testing Checklist

| Scenario | Expected Result |
|----------|-----------------|
| Get open incidents for user | Returns filtered list |
| Get incident by INCxxxxxxx | Returns incident details |
| Get incident by sys_id | Returns incident details |
| Create incident | Returns new INC number |
| Add comment to incident | Comment visible in ServiceNow |
| Cancel incident | State changes to Canceled |
| Escalate incident | Work note added |
| Health check | Returns ServiceNow status |

---

## Implementation Timeline

### Phase 1: Core Infrastructure (ServiceNow Provider)

| Step | Tasks | Estimate |
|------|-------|----------|
| 1 | Create ITSM types and interfaces (`types.js`, `interfaces.js`) | 1 hour |
| 2 | Create `BaseITSMProvider` abstract class | 2 hours |
| 3 | Create `ServiceNowProvider` implementation | 4 hours |
| 4 | Create `MockProvider` for testing | 1.5 hours |
| 5 | Create `ITSMProviderFactory` | 2 hours |
| 6 | Create `IncidentService` orchestration | 3 hours |
| 7 | Create `itsmFormatter` utility | 1.5 hours |
| 8 | Create ITSM controller | 3 hours |
| 9 | Create ITSM routes with Swagger | 2 hours |
| 10 | Update routes index | 0.5 hours |
| 11 | Testing and debugging | 4 hours |
| **Phase 1 Total** | | **~24.5 hours** |

### Phase 2: Additional Provider (Jira Service Management)

| Step | Tasks | Estimate |
|------|-------|----------|
| 1 | Research Jira SM API | 2 hours |
| 2 | Create `JiraServiceProvider` implementation | 6 hours |
| 3 | Add Jira to factory | 1 hour |
| 4 | Testing and validation | 3 hours |
| **Phase 2 Total** | | **~12 hours** |

### Future Phases (Per Provider)

| Provider | Estimated Effort |
|----------|-----------------|
| Zendesk | ~10 hours |
| Freshservice | ~10 hours |
| BMC Remedy | ~12 hours |

---

## Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| ServiceNow rate limiting | API failures | Implement retry with backoff |
| Network timeout | Slow responses | Configure appropriate timeouts |
| Invalid caller email | Failed lookups | Validate user before operations |
| State transition violations | Update failures | Validate transitions before update |
| Concurrent updates | Data conflicts | Use ServiceNow's optimistic locking |

---

## Follow-Up Tasks

1. **LangFlow Flow Configuration** - Create flows to use these APIs
2. **Adaptive Card Templates** - Create incident display cards
3. **Live Agent Integration** - Design handoff with ticket context
4. **Caching Strategy** - Implement caching for frequently accessed incidents
5. **Webhook Support** - ServiceNow webhooks for incident updates

---

## References

### Official Documentation
- [ServiceNow Table API](https://www.servicenow.com/docs/bundle/zurich-api-reference/page/integrate/inbound-rest/concept/c_TableAPI.html)
- [ServiceNow Encoded Query Strings](https://www.servicenow.com/docs/bundle/zurich-platform-user-interface/page/use/using-lists/concept/c_EncodedQueryStrings.html)
- [ServiceNow Incident Management](https://www.servicenow.com/docs/bundle/zurich-it-service-management/page/product/incident-management/concept/c_IncidentManagement.html)

### Project Documentation
- [Drug Controller](botframework/src/controllers/drug.controller.js) - Reference pattern
- [Medical Controller](botframework/src/controllers/medical.controller.js) - Reference pattern
- [ServiceNow Integration Guidelines](/.cursor/rules/servicenow-integration.mdc)
