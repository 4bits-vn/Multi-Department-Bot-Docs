# CB-011: Message Streaming Implementation Plan

## Technical Overview

### Current Architecture

```
User Message → Bot Controller → LangFlow Handler → MSTeamsService.sendProactiveMessage()
                    ↓
              Typing Indicator (simple)
                    ↓
              Wait for full response from LangFlow
                    ↓
              Send complete message
```

### Target Architecture with End-to-End Streaming

```
User Message → Bot Controller → StreamingService.start()
                    ↓
              📡 Informative Update: "Understanding your request..."
                    ↓
              LangFlow Router (non-streaming)
                    ↓
              📡 Informative Update: "Searching knowledge base..."
                    ↓
              LangFlow Sub-flow (STREAMING ENABLED)
                    ↓
              📡 Stream LLM tokens → Teams in real-time
                    ↓
              📡 Final Message (with attachments)
```

## LangFlow Streaming Support

### Flows with Streaming Enabled

| Flow | Environment Variable | Streaming Support |
|------|---------------------|-------------------|
| IT_KB_SEARCH_FLOW | `IT_KB_SEARCH_FLOW` | ✅ Yes |
| MED_KB_SEARCH_FLOW | `MED_KB_SEARCH_FLOW` | ✅ Yes |
| CHIT_CHAT_FLOW | `CHIT_CHAT_FLOW` | ✅ Yes |
| EXCEPTION_FLOW | `EXCEPTION_FLOW` | ✅ Yes |
| OTHER_FLOW | `OTHER_FLOW` | ✅ Yes |
| ROUTER_FLOW | `ROUTER_FLOW` | ❌ No (needs structured JSON) |
| IT_TICKET_MGMT_FLOW | `IT_TICKET_MGMT_FLOW` | ❌ No (action-based) |
| IT_HELP_DESK_FLOW | `IT_HELP_DESK_FLOW` | ❌ No (escalation) |
| MED_DRUG_SEARCH_FLOW | `MED_DRUG_SEARCH_FLOW` | ❌ No (API aggregation) |

### LangFlow Streaming API

From [LangFlow Documentation](https://docs.langflow.org/api-flows-run):

**Endpoint:** `POST /api/v1/run/{flow_id}?stream=true`

**SSE Response Events:**
```json
// Initial message
{"event": "add_message", "data": {...}}

// LLM tokens as they're generated
{"event": "token", "data": {"chunk": " Have", "id": "...", "timestamp": "..."}}

// More tokens...
{"event": "token", "data": {"chunk": " you", "id": "...", "timestamp": "..."}}

// Final result
{"event": "end", "data": {"result": {"session_id": "...", "message": "Complete response..."}}}
```

---

## Implementation Strategy

We will implement **end-to-end streaming** that:
1. Uses **Teams REST API** for streaming to users
2. Uses **LangFlow SSE streaming** for real-time LLM output
3. Connects both streams to deliver **real-time LLM responses** to Teams users

---

## Implementation Steps

### Step 1: Create LangFlow Streaming Client

**File:** `src/Services/LangFlowStreamingClient.js`

A new service to handle SSE streaming from LangFlow:

```javascript
/**
 * LangFlow Streaming Client
 *
 * Handles Server-Sent Events (SSE) streaming from LangFlow API.
 * Parses token events and emits chunks for downstream consumption.
 *
 * @see https://docs.langflow.org/api-flows-run
 */
const { EventEmitter } = require('events');
const axios = require('axios');

class LangFlowStreamingClient extends EventEmitter {
    constructor(config) {
        super();
        this.baseUrl = config.baseUrl;
        this.apiKey = config.apiKey;
        this.timeout = config.timeout || 120000;
    }

    /**
     * Execute a flow with streaming enabled
     *
     * @param {string} flowId - Flow ID
     * @param {Object} payload - Request payload
     * @returns {Promise<Object>} Final result
     *
     * Events emitted:
     * - 'token': {chunk: string} - LLM token received
     * - 'message': {text: string} - Complete message (add_message event)
     * - 'end': {result: Object} - Stream completed
     * - 'error': {error: Error} - Stream error
     */
    async runWithStreaming(flowId, payload) {
        const url = `${this.baseUrl}/api/v1/run/${flowId}?stream=true`;

        return new Promise((resolve, reject) => {
            let fullResponse = '';
            let finalResult = null;

            axios({
                method: 'POST',
                url,
                headers: {
                    'Content-Type': 'application/json',
                    'Accept': 'text/event-stream',
                    ...(this.apiKey && { 'x-api-key': this.apiKey })
                },
                data: payload,
                responseType: 'stream',
                timeout: this.timeout
            })
            .then(response => {
                const stream = response.data;

                stream.on('data', (chunk) => {
                    const lines = chunk.toString().split('\n');

                    for (const line of lines) {
                        if (line.startsWith('data: ')) {
                            try {
                                const data = JSON.parse(line.slice(6));
                                this.handleSSEEvent(data, (token) => {
                                    fullResponse += token;
                                    this.emit('token', { chunk: token, accumulated: fullResponse });
                                });

                                if (data.event === 'end') {
                                    finalResult = data.data?.result;
                                }
                            } catch (e) {
                                // Skip malformed JSON
                            }
                        }
                    }
                });

                stream.on('end', () => {
                    this.emit('end', { result: finalResult, fullResponse });
                    resolve({ success: true, text: fullResponse, result: finalResult });
                });

                stream.on('error', (error) => {
                    this.emit('error', { error });
                    reject(error);
                });
            })
            .catch(reject);
        });
    }

    /**
     * Handle individual SSE event
     * @private
     */
    handleSSEEvent(data, onToken) {
        switch (data.event) {
            case 'token':
                if (data.data?.chunk) {
                    onToken(data.data.chunk);
                }
                break;

            case 'add_message':
                this.emit('message', { text: data.data?.text });
                break;

            case 'end':
                // Handled in stream.on('end')
                break;

            case 'error':
                this.emit('error', { error: new Error(data.data?.message || 'Stream error') });
                break;
        }
    }
}

module.exports = { LangFlowStreamingClient };
```

---

### Step 2: Create MessageStreamingService

**File:** `src/Services/MessageStreamingService.js`

Main orchestration service that bridges LangFlow streaming to Teams streaming:

```javascript
/**
 * Message Streaming Service
 *
 * Orchestrates end-to-end streaming from LangFlow to MS Teams.
 * - Manages Teams streaming sessions
 * - Consumes LangFlow SSE tokens
 * - Buffers and delivers to Teams with rate limiting
 *
 * @see https://learn.microsoft.com/en-us/microsoftteams/platform/bots/streaming-ux
 */
const { createLogger } = require('./Logger');
const { LangFlowStreamingClient } = require('./LangFlowStreamingClient');
const StreamSession = require('./StreamSession');
const { STREAMING_MESSAGES, FLOW_TO_MESSAGE } = require('../config/streamingMessages');

const logger = createLogger('MessageStreamingService');

class MessageStreamingService {
    constructor(msTeamsService, config = {}) {
        this.msTeamsService = msTeamsService;
        this.activeSessions = new Map();

        // Configuration
        this.config = {
            enabled: config.enabled !== false,
            minUpdateIntervalMs: config.minUpdateIntervalMs || 1500,
            maxDurationMs: config.maxDurationMs || 120000,
            tokenBufferSize: config.tokenBufferSize || 30, // Characters to buffer before sending
            tokenBufferTimeoutMs: config.tokenBufferTimeoutMs || 1500
        };

        // LangFlow streaming client
        this.langFlowClient = new LangFlowStreamingClient({
            baseUrl: config.langflowUrl || process.env.LANGFLOW_URL,
            apiKey: config.langflowApiKey || process.env.LANGFLOW_API_KEY
        });

        logger.info('MessageStreamingService initialized', {
            enabled: this.config.enabled,
            minUpdateIntervalMs: this.config.minUpdateIntervalMs
        });
    }

    /**
     * Check if streaming is supported for this conversation
     */
    isStreamingSupported(activity) {
        if (!this.config.enabled) return false;

        // Only one-on-one chats support streaming
        const isTeams = activity.channelId === 'msteams';
        const isPersonal = activity.conversation?.conversationType === 'personal';

        return isTeams && isPersonal;
    }

    /**
     * Start a new streaming session
     *
     * @param {Object} params
     * @param {string} params.conversationId
     * @param {string} params.serviceUrl
     * @param {string} params.initialMessage - First informative message
     * @returns {Promise<StreamSession>}
     */
    async startStream(params) {
        const { conversationId, serviceUrl, initialMessage } = params;

        // Create session
        const session = new StreamSession({
            conversationId,
            serviceUrl,
            minUpdateIntervalMs: this.config.minUpdateIntervalMs,
            maxDurationMs: this.config.maxDurationMs
        });

        // Start Teams streaming with informative message
        const result = await this.msTeamsService.startStreamingActivity(
            serviceUrl,
            conversationId,
            initialMessage || STREAMING_MESSAGES.PROCESSING,
            'informative'
        );

        session.streamId = result.streamId;
        session.sequence = 1;
        session.isActive = true;
        session.lastUpdateAt = Date.now();

        // Store session
        this.activeSessions.set(conversationId, session);

        logger.info('Streaming session started', {
            conversationId: conversationId?.substring(0, 20),
            streamId: session.streamId
        });

        return session;
    }

    /**
     * Send informative update (status message)
     */
    async sendInformativeUpdate(session, message) {
        if (!session.isActive || session.isEnded) return;

        // Check rate limit
        if (!session.canSendUpdate()) {
            logger.debug('Rate limited - skipping informative update');
            return;
        }

        const sequence = session.nextSequence();

        await this.msTeamsService.continueStreaming(
            session.serviceUrl,
            session.conversationId,
            session.streamId,
            sequence,
            message,
            'informative'
        );

        session.lastUpdateAt = Date.now();

        logger.debug('Informative update sent', {
            message: message.substring(0, 50),
            sequence
        });
    }

    /**
     * Execute LangFlow flow with streaming and pipe to Teams
     *
     * This is the main method that connects LangFlow streaming to Teams streaming.
     *
     * @param {StreamSession} session - Active streaming session
     * @param {string} flowId - LangFlow flow ID
     * @param {Object} payload - LangFlow request payload
     * @returns {Promise<Object>} Final result from LangFlow
     */
    async streamFromLangFlow(session, flowId, payload) {
        if (!session.isActive || session.isEnded) {
            throw new Error('Streaming session is not active');
        }

        return new Promise((resolve, reject) => {
            let tokenBuffer = '';
            let bufferTimeout = null;
            let finalResult = null;

            // Flush buffer to Teams
            const flushBuffer = async () => {
                if (tokenBuffer.length === 0) return;
                if (!session.canSendUpdate()) return;

                const textToSend = session.appendText(tokenBuffer);
                tokenBuffer = '';

                try {
                    const sequence = session.nextSequence();
                    await this.msTeamsService.continueStreaming(
                        session.serviceUrl,
                        session.conversationId,
                        session.streamId,
                        sequence,
                        textToSend,
                        'streaming'
                    );
                    session.lastUpdateAt = Date.now();
                } catch (error) {
                    logger.warn('Failed to send streaming content', { error: error.message });
                }
            };

            // Handle incoming tokens
            this.langFlowClient.on('token', async ({ chunk }) => {
                tokenBuffer += chunk;

                // Flush when buffer is large enough
                if (tokenBuffer.length >= this.config.tokenBufferSize) {
                    clearTimeout(bufferTimeout);
                    await flushBuffer();
                } else {
                    // Set timeout to flush smaller buffers
                    clearTimeout(bufferTimeout);
                    bufferTimeout = setTimeout(flushBuffer, this.config.tokenBufferTimeoutMs);
                }
            });

            // Handle stream end
            this.langFlowClient.on('end', async ({ result, fullResponse }) => {
                clearTimeout(bufferTimeout);
                finalResult = { success: true, text: fullResponse, result };

                // Flush any remaining buffer
                if (tokenBuffer.length > 0) {
                    session.appendText(tokenBuffer);
                }

                // Remove listeners
                this.langFlowClient.removeAllListeners();

                resolve(finalResult);
            });

            // Handle errors
            this.langFlowClient.on('error', ({ error }) => {
                clearTimeout(bufferTimeout);
                this.langFlowClient.removeAllListeners();
                reject(error);
            });

            // Start streaming from LangFlow
            this.langFlowClient.runWithStreaming(flowId, payload).catch(reject);
        });
    }

    /**
     * End streaming with final message
     *
     * @param {StreamSession} session
     * @param {string} finalText
     * @param {Array} [attachments]
     */
    async endStream(session, finalText, attachments = null) {
        if (session.isEnded) return;

        session.isEnded = true;
        session.isActive = false;

        try {
            await this.msTeamsService.endStreaming(
                session.serviceUrl,
                session.conversationId,
                session.streamId,
                finalText,
                attachments
            );

            logger.info('Streaming session ended', {
                conversationId: session.conversationId?.substring(0, 20),
                totalSequence: session.sequence
            });
        } finally {
            // Cleanup
            this.activeSessions.delete(session.conversationId);
        }
    }

    /**
     * Cancel streaming and fall back to standard message
     */
    async cancelAndFallback(session, message, attachments = null) {
        if (session.isEnded) return;

        session.isEnded = true;
        session.isActive = false;
        this.activeSessions.delete(session.conversationId);

        // Send as standard proactive message
        await this.msTeamsService.sendProactiveMessage({
            serviceUrl: session.serviceUrl,
            conversationId: session.conversationId,
            message,
            attachments
        });

        logger.info('Streaming cancelled, sent fallback message', {
            conversationId: session.conversationId?.substring(0, 20)
        });
    }

    /**
     * Get session for conversation
     */
    getSession(conversationId) {
        return this.activeSessions.get(conversationId);
    }
}

module.exports = { MessageStreamingService };
```

---

### Step 3: Create StreamSession Class

**File:** `src/Services/StreamSession.js`

```javascript
/**
 * Stream Session
 *
 * Represents an active streaming session between bot and Teams user.
 * Manages state, rate limiting, and text accumulation.
 */
class StreamSession {
    constructor(params) {
        this.conversationId = params.conversationId;
        this.serviceUrl = params.serviceUrl;
        this.streamId = null;
        this.sequence = 0;
        this.startedAt = Date.now();
        this.lastUpdateAt = null;
        this.accumulatedText = '';
        this.isActive = false;
        this.isEnded = false;

        // Rate limiting configuration
        this.minUpdateIntervalMs = params.minUpdateIntervalMs || 1500;
        this.maxDurationMs = params.maxDurationMs || 120000;
    }

    /**
     * Check if enough time has passed for next update (rate limiting)
     */
    canSendUpdate() {
        if (!this.lastUpdateAt) return true;
        return (Date.now() - this.lastUpdateAt) >= this.minUpdateIntervalMs;
    }

    /**
     * Get next sequence number
     */
    nextSequence() {
        return ++this.sequence;
    }

    /**
     * Append text for streaming (Teams requires cumulative content)
     */
    appendText(newText) {
        this.accumulatedText += newText;
        return this.accumulatedText;
    }

    /**
     * Check if session has exceeded max duration (2 minutes)
     */
    hasTimedOut() {
        return (Date.now() - this.startedAt) > this.maxDurationMs;
    }

    /**
     * Get elapsed time in milliseconds
     */
    getElapsedMs() {
        return Date.now() - this.startedAt;
    }
}

module.exports = StreamSession;
```

---

### Step 4: Create Streaming Message Constants

**File:** `src/config/streamingMessages.js`

```javascript
/**
 * Streaming Message Constants
 *
 * Informative messages displayed during different processing phases.
 * These appear as a blue progress bar in Teams.
 */

const STREAMING_MESSAGES = {
    // Router phase
    UNDERSTANDING_REQUEST: "Understanding your request...",
    ANALYZING_INTENT: "Analyzing your question...",

    // IT flows
    SEARCHING_IT_KB: "Searching IT knowledge base...",
    CHECKING_TICKETS: "Looking up your tickets...",
    CONNECTING_SUPPORT: "Connecting to support system...",

    // Medical flows
    SEARCHING_MEDICAL_KB: "Looking up health information...",
    SEARCHING_DRUG_INFO: "Searching drug database...",
    CHECKING_MEDLINEPLUS: "Checking MedlinePlus...",
    NORMALIZING_DRUG: "Looking up drug information...",

    // General
    PROCESSING: "Processing your request...",
    GENERATING_RESPONSE: "Generating response...",
    FINALIZING: "Preparing your answer..."
};

/**
 * Map routing results to informative messages
 */
const FLOW_TO_MESSAGE = {
    'ROUTER': STREAMING_MESSAGES.UNDERSTANDING_REQUEST,
    'IT_KB_SEARCH': STREAMING_MESSAGES.SEARCHING_IT_KB,
    'IT_TICKET_MGMT': STREAMING_MESSAGES.CHECKING_TICKETS,
    'IT_HELP_DESK': STREAMING_MESSAGES.CONNECTING_SUPPORT,
    'MED_KB_SEARCH': STREAMING_MESSAGES.SEARCHING_MEDICAL_KB,
    'MED_DRUG_SEARCH': STREAMING_MESSAGES.SEARCHING_DRUG_INFO,
    'CHIT_CHAT': STREAMING_MESSAGES.GENERATING_RESPONSE,
    'EXCEPTION': STREAMING_MESSAGES.PROCESSING,
    'OTHER': STREAMING_MESSAGES.PROCESSING
};

/**
 * Flows that support LangFlow streaming
 * These flows have streaming enabled in LangFlow
 */
const STREAMING_ENABLED_FLOWS = new Set([
    'IT_KB_SEARCH',
    'MED_KB_SEARCH',
    'CHIT_CHAT',
    'EXCEPTION',
    'OTHER'
]);

/**
 * Check if a flow supports LangFlow streaming
 */
function isFlowStreamingEnabled(flowRoute) {
    return STREAMING_ENABLED_FLOWS.has(flowRoute);
}

module.exports = {
    STREAMING_MESSAGES,
    FLOW_TO_MESSAGE,
    STREAMING_ENABLED_FLOWS,
    isFlowStreamingEnabled
};
```

---

### Step 5: Add Streaming Methods to MSTeamsService

**File:** `src/Services/MSTeamsService.js` (additions)

```javascript
// ============================================
// STREAMING METHODS
// ============================================

/**
 * Start a streaming session
 *
 * @param {string} serviceUrl - Bot connector service URL
 * @param {string} conversationId - Conversation ID
 * @param {string} initialMessage - First message to display
 * @param {string} streamType - 'informative' or 'streaming'
 * @returns {Promise<{streamId: string}>}
 */
async startStreamingActivity(serviceUrl, conversationId, initialMessage, streamType = 'informative') {
    const activity = {
        type: 'typing',
        text: initialMessage,
        entities: [{
            type: 'streaminfo',
            streamType: streamType,
            streamSequence: 1
        }]
    };

    const result = await this.sendStreamingActivity(serviceUrl, conversationId, activity);
    return { streamId: result.id };
}

/**
 * Continue streaming with informative or content update
 *
 * @param {string} serviceUrl
 * @param {string} conversationId
 * @param {string} streamId - From initial request
 * @param {number} sequence - Incremental sequence number
 * @param {string} text - Message content
 * @param {string} streamType - 'informative' or 'streaming'
 */
async continueStreaming(serviceUrl, conversationId, streamId, sequence, text, streamType = 'streaming') {
    const activity = {
        type: 'typing',
        text: text,
        entities: [{
            type: 'streaminfo',
            streamId: streamId,
            streamType: streamType,
            streamSequence: sequence
        }]
    };

    return this.sendStreamingActivity(serviceUrl, conversationId, activity);
}

/**
 * End streaming with final message
 *
 * @param {string} serviceUrl
 * @param {string} conversationId
 * @param {string} streamId
 * @param {string} finalText
 * @param {Array} [attachments]
 */
async endStreaming(serviceUrl, conversationId, streamId, finalText, attachments = null) {
    const activity = {
        type: 'message',
        text: finalText,
        entities: [{
            type: 'streaminfo',
            streamId: streamId,
            streamType: 'final'
        }]
    };

    if (attachments && attachments.length > 0) {
        activity.attachments = attachments;
    }

    return this.sendStreamingActivity(serviceUrl, conversationId, activity);
}

/**
 * Send streaming activity (low-level)
 * @private
 */
async sendStreamingActivity(serviceUrl, conversationId, activity) {
    if (!this.isConfigured) {
        logger.info('Would send streaming activity (mock)', {
            conversationId,
            type: activity.type,
            streamType: activity.entities?.[0]?.streamType
        });
        return { success: true, mock: true, id: `mock-stream-${Date.now()}` };
    }

    try {
        const token = await this.getAccessToken();
        const baseUrl = serviceUrl.endsWith('/') ? serviceUrl.slice(0, -1) : serviceUrl;
        const url = `${baseUrl}/v3/conversations/${encodeURIComponent(conversationId)}/activities`;

        // Transform emoji codes
        const transformedActivity = { ...activity };
        if (transformedActivity.text) {
            transformedActivity.text = transformEmojiCodes(transformedActivity.text);
        }

        const response = await axios({
            method: 'POST',
            url,
            headers: {
                'Content-Type': 'application/json',
                'Authorization': `Bearer ${token}`
            },
            data: {
                ...transformedActivity,
                serviceUrl,
                conversation: {
                    id: conversationId,
                    conversationType: 'personal'
                }
            }
        });

        logger.debug('Streaming activity sent', {
            conversationId: conversationId?.substring(0, 20),
            activityId: response.data?.id,
            streamType: activity.entities?.[0]?.streamType
        });

        return {
            success: true,
            id: response.data?.id
        };

    } catch (error) {
        logger.error('Failed to send streaming activity', {
            conversationId: conversationId?.substring(0, 20),
            status: error.response?.status,
            errorCode: error.response?.data?.error?.code,
            error: error.message
        });

        // Check for specific streaming errors
        if (error.response?.data?.error?.code === 'ContentStreamNotAllowed') {
            throw new StreamingError('Streaming not allowed', error.response?.data?.error);
        }

        throw error;
    }
}

/**
 * Custom error for streaming failures
 */
class StreamingError extends Error {
    constructor(message, details) {
        super(message);
        this.name = 'StreamingError';
        this.details = details;
    }
}
```

---

### Step 6: Update LangFlow Config

**File:** `src/Handlers/LangFlow/config.js` (additions)

```javascript
/**
 * Flows that support streaming output
 * These flows have been configured in LangFlow with streaming enabled
 */
const STREAMING_ENABLED_FLOW_IDS = {
    itKbSearch: IT_KB_SEARCH_FLOW,
    medKbSearch: MED_KB_SEARCH_FLOW,
    chitChat: CHIT_CHAT_FLOW,
    exception: EXCEPTION_FLOW,
    other: OTHER_FLOW
};

/**
 * Check if a flow supports streaming
 * @param {string} flowId - Flow ID or route name
 * @returns {boolean}
 */
function isFlowStreamingEnabled(flowIdOrRoute) {
    // Check by route name
    const routeToFlowId = {
        'IT_KB_SEARCH': IT_KB_SEARCH_FLOW,
        'MED_KB_SEARCH': MED_KB_SEARCH_FLOW,
        'CHIT_CHAT': CHIT_CHAT_FLOW,
        'EXCEPTION': EXCEPTION_FLOW,
        'OTHER': OTHER_FLOW
    };

    const flowId = routeToFlowId[flowIdOrRoute] || flowIdOrRoute;

    return Object.values(STREAMING_ENABLED_FLOW_IDS).includes(flowId);
}

/**
 * Build streaming API URL
 */
function buildStreamingApiUrl(flowId) {
    return `${LANGFLOW_URL}/api/v1/run/${flowId}?stream=true`;
}

module.exports = {
    // ... existing exports ...
    STREAMING_ENABLED_FLOW_IDS,
    isFlowStreamingEnabled,
    buildStreamingApiUrl
};
```

---

### Step 7: Update Bot Controller

**File:** `src/controllers/bot.controller.js` (key changes)

```javascript
// Add imports
const { MessageStreamingService } = require('../Services/MessageStreamingService');
const { STREAMING_MESSAGES, FLOW_TO_MESSAGE, isFlowStreamingEnabled } = require('../config/streamingMessages');
const config = require('../config');

// Initialize streaming service
const messageStreamingService = new MessageStreamingService(msTeamsService, {
    enabled: config.streaming?.enabled,
    minUpdateIntervalMs: config.streaming?.minUpdateIntervalMs,
    maxDurationMs: config.streaming?.maxDurationMs,
    langflowUrl: config.services.langflow.url,
    langflowApiKey: config.services.langflow.apiKey
});

/**
 * Handle router response with streaming support
 */
async function handleRouterResponse({ routerResult, userEmail, channel, sessionId, conversationId, serviceUrl, originalInput }) {
    // ... existing setup code ...

    // Check if streaming is supported
    const supportsStreaming = messageStreamingService.isStreamingSupported({
        channelId: channel,
        conversation: { conversationType: 'personal' }
    });

    let streamSession = null;

    // Start streaming session if supported
    if (supportsStreaming) {
        try {
            streamSession = await messageStreamingService.startStream({
                conversationId,
                serviceUrl,
                initialMessage: STREAMING_MESSAGES.UNDERSTANDING_REQUEST
            });

            logger.info('Streaming session started', {
                userEmail,
                streamId: streamSession.streamId
            });
        } catch (error) {
            logger.warn('Failed to start streaming, using standard messages', {
                error: error.message
            });
        }
    }

    // CONVERSATION LOOP
    while (iteration < MAX_ITERATIONS) {
        // ... existing iteration logic ...

        // Send informative update based on route
        if (streamSession) {
            const informativeMessage = FLOW_TO_MESSAGE[route] || STREAMING_MESSAGES.PROCESSING;
            await messageStreamingService.sendInformativeUpdate(streamSession, informativeMessage);
        } else {
            await msTeamsService.sendTypingIndicator(serviceUrl, conversationId);
        }

        // Check if this flow supports streaming
        const flowSupportsStreaming = isFlowStreamingEnabled(route);

        if (streamSession && flowSupportsStreaming) {
            // Use streaming for this flow
            try {
                const subFlowResult = await executeFlowWithStreaming(
                    streamSession,
                    route,
                    sessionId,
                    userEmail,
                    originalInput,
                    metadata
                );

                // Stream completed - end session with final message
                await messageStreamingService.endStream(
                    streamSession,
                    subFlowResult.text,
                    null // attachments if any
                );

                // Exit loop
                break;

            } catch (streamError) {
                logger.error('Streaming error, falling back', { error: streamError.message });

                // Fall back to standard message
                const fallbackResult = await langFlowHandler.processRoutingResult({
                    routingResult: route,
                    sessionId,
                    userEmail,
                    originalInput,
                    metadata
                });

                await messageStreamingService.cancelAndFallback(
                    streamSession,
                    extractMessage(fallbackResult.text || fallbackResult.message)
                );

                break;
            }
        } else {
            // Non-streaming flow - use existing logic
            // ... existing sub-flow processing ...

            // Send final message
            if (streamSession) {
                await messageStreamingService.endStream(
                    streamSession,
                    extractedMessage,
                    null
                );
            } else {
                await msTeamsService.sendProactiveMessage({
                    serviceUrl,
                    conversationId,
                    message: extractedMessage
                });
            }
        }
    }

    // ... existing cleanup ...
}

/**
 * Execute a flow with streaming output
 */
async function executeFlowWithStreaming(streamSession, route, sessionId, userEmail, originalInput, metadata) {
    const flowIdMap = {
        'IT_KB_SEARCH': process.env.IT_KB_SEARCH_FLOW,
        'MED_KB_SEARCH': process.env.MED_KB_SEARCH_FLOW,
        'CHIT_CHAT': process.env.CHIT_CHAT_FLOW,
        'EXCEPTION': process.env.EXCEPTION_FLOW,
        'OTHER': process.env.OTHER_FLOW
    };

    const flowId = flowIdMap[route];
    if (!flowId) {
        throw new Error(`No flow ID for route: ${route}`);
    }

    const payload = {
        input_value: metadata.keyword || originalInput,
        output_type: 'chat',
        input_type: 'chat',
        session_id: sessionId,
        tweaks: {
            sender_name: userEmail
        }
    };

    return messageStreamingService.streamFromLangFlow(streamSession, flowId, payload);
}
```

---

### Step 8: Update Configuration

**File:** `src/config/index.js` (additions)

```javascript
/**
 * Message Streaming Configuration
 */
streaming: {
    // Enable/disable streaming feature
    enabled: process.env.ENABLE_MESSAGE_STREAMING !== 'false',

    // Minimum interval between streaming updates (ms) - Teams rate limit
    minUpdateIntervalMs: parseInt(process.env.STREAMING_UPDATE_INTERVAL_MS) || 1500,

    // Maximum streaming duration before force-end (ms) - Teams limit is 2 minutes
    maxDurationMs: parseInt(process.env.STREAMING_MAX_DURATION_MS) || 120000,

    // Token buffer size (characters) before sending to Teams
    tokenBufferSize: parseInt(process.env.STREAMING_TOKEN_BUFFER_SIZE) || 30,

    // Token buffer timeout (ms) - flush buffer if no new tokens
    tokenBufferTimeoutMs: parseInt(process.env.STREAMING_TOKEN_BUFFER_TIMEOUT_MS) || 1500
}
```

---

## File Changes Summary

| File | Change Type | Description |
|------|-------------|-------------|
| `src/Services/LangFlowStreamingClient.js` | **NEW** | SSE client for LangFlow streaming |
| `src/Services/MessageStreamingService.js` | **NEW** | Main streaming orchestration |
| `src/Services/StreamSession.js` | **NEW** | Stream session state management |
| `src/config/streamingMessages.js` | **NEW** | Informative message constants |
| `src/Services/MSTeamsService.js` | **MODIFY** | Add Teams streaming API methods |
| `src/Handlers/LangFlow/config.js` | **MODIFY** | Add streaming-enabled flow config |
| `src/controllers/bot.controller.js` | **MODIFY** | Integrate streaming into conversation loop |
| `src/config/index.js` | **MODIFY** | Add streaming configuration |

---

## Data Flow Diagram

```
┌─────────────┐     ┌──────────────────┐     ┌─────────────────┐     ┌──────────────┐
│   MS Teams  │     │  Bot Controller  │     │ StreamingService│     │   LangFlow   │
│    User     │     │                  │     │                 │     │              │
└─────┬───────┘     └────────┬─────────┘     └────────┬────────┘     └──────┬───────┘
      │                      │                        │                     │
      │  User Message        │                        │                     │
      │─────────────────────>│                        │                     │
      │                      │                        │                     │
      │                      │  startStream()         │                     │
      │                      │───────────────────────>│                     │
      │                      │                        │                     │
      │  "Understanding..."  │<───────────────────────│                     │
      │<─────────────────────│  informative update    │                     │
      │                      │                        │                     │
      │                      │  streamFromLangFlow()  │                     │
      │                      │───────────────────────>│  POST ?stream=true  │
      │                      │                        │────────────────────>│
      │                      │                        │                     │
      │                      │                        │  SSE: token events  │
      │                      │                        │<────────────────────│
      │  Streaming text...   │<───────────────────────│  buffer & send      │
      │<─────────────────────│                        │                     │
      │                      │                        │                     │
      │  More text...        │<───────────────────────│  more tokens        │
      │<─────────────────────│                        │<────────────────────│
      │                      │                        │                     │
      │                      │                        │  SSE: end event     │
      │                      │                        │<────────────────────│
      │                      │  endStream()           │                     │
      │                      │───────────────────────>│                     │
      │  Final message       │<───────────────────────│                     │
      │<─────────────────────│                        │                     │
      │                      │                        │                     │
```

---

## Testing Strategy

### Unit Tests

1. **LangFlowStreamingClient**
   - SSE parsing
   - Token event handling
   - Error event handling

2. **StreamSession**
   - Rate limiting
   - Text accumulation
   - Timeout detection

3. **MessageStreamingService**
   - Session lifecycle
   - LangFlow → Teams bridging
   - Fallback handling

### Integration Tests

| Test Case | Expected Behavior |
|-----------|-------------------|
| IT KB search with streaming | Informative → LLM tokens stream → Final message |
| Medical KB search with streaming | Informative → LLM tokens stream → Final message |
| Chit-chat with streaming | Quick informative → Stream response |
| Non-streaming flow (Drug Search) | Informative → Standard final message |
| Streaming timeout | Falls back to standard message |
| User cancels streaming | Graceful cleanup, no error |

### Manual Testing Checklist

- [ ] One-on-one Teams chat shows streaming
- [ ] Informative messages appear (blue progress bar)
- [ ] LLM response streams incrementally
- [ ] Final message is complete and correct
- [ ] Attachments work in final message
- [ ] Stop button works
- [ ] Group chats fall back correctly
- [ ] Error recovery works

---

## Rollout Plan

### Phase 1: Foundation (Day 1-2)
- [ ] Create `LangFlowStreamingClient`
- [ ] Create `StreamSession`
- [ ] Create `MessageStreamingService`
- [ ] Add streaming methods to `MSTeamsService`
- [ ] Add configuration

### Phase 2: Integration (Day 3-4)
- [ ] Update LangFlow config for streaming flows
- [ ] Add streaming message constants
- [ ] Integrate into bot controller
- [ ] Add fallback handling

### Phase 3: Testing & Refinement (Day 5)
- [ ] End-to-end testing with all streaming flows
- [ ] Performance tuning (buffer sizes, intervals)
- [ ] Error handling edge cases
- [ ] Documentation

---

## Environment Variables

```bash
# Streaming Feature
ENABLE_MESSAGE_STREAMING=true

# Rate Limiting (ms)
STREAMING_UPDATE_INTERVAL_MS=1500

# Max Duration (ms) - Teams limit is 2 minutes
STREAMING_MAX_DURATION_MS=120000

# Token Buffering
STREAMING_TOKEN_BUFFER_SIZE=30
STREAMING_TOKEN_BUFFER_TIMEOUT_MS=1500
```

---

## References

- [Microsoft Teams Streaming Bot Messages](https://learn.microsoft.com/en-us/microsoftteams/platform/bots/streaming-ux?tabs=csharp#stream-message-through-rest-api)
- [LangFlow Streaming API](https://docs.langflow.org/api-flows-run)
- [Bot Framework REST API](https://learn.microsoft.com/en-us/azure/bot-service/rest-api/bot-framework-rest-connector-api-reference)
