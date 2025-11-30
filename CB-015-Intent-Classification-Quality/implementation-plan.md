# CB-015: Intent Classification Quality - Implementation Plan

## Overview

This document outlines the implementation plan for enhancing the intent classification system to achieve higher accuracy and reliability.

## Technical Approach

### Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      ENHANCED CLASSIFICATION PIPELINE                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  User Input: "my screen is low"                                             │
│       │                                                                     │
│       ▼                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ LAYER 1: Keyword Matching (Fast Path)                                │   │
│  │ - Multilingual patterns (EN, FR, VI)                                 │   │
│  │ - If STRONG match → Return immediately (score: 1.0)                  │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│       │ No match                                                            │
│       ▼                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ LAYER 2: Out-of-Scope Detection (NEW)                                │   │
│  │ - Quick check: Is this in-scope or random?                           │
│  │ - Labels: ["IT query", "Medical query", "Greeting", "Unrelated"]     │
│  │ - If "Unrelated" + high confidence → Return OTHER                    │
│  └─────────────────────────────────────────────────────────────────────┘   │
│       │ In-scope                                                            │
│       ▼                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ LAYER 3: Ensemble Classification (NEW)                               │   │
│  │ ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐         │   │
│  │ │ Zero-Shot NLI   │ │ Sentence Embed  │ │ Domain Heuristics│         │   │
│  │ │ (Hierarchical)  │ │ (KNN Similarity)│ │ (Keywords+Rules) │         │   │
│  │ │ Weight: 0.4     │ │ Weight: 0.35    │ │ Weight: 0.25     │         │   │
│  │ └────────┬────────┘ └────────┬────────┘ └────────┬────────┘         │   │
│  │          │                   │                   │                   │   │
│  │          └───────────────────┴───────────────────┘                   │   │
│  │                              │                                       │   │
│  │                     Weighted Vote                                    │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│       │                                                                     │
│       ▼                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ LAYER 4: Confidence Calibration & Thresholds                         │   │
│  │ - Route-specific thresholds                                          │   │
│  │ - Temperature scaling for score calibration                          │   │
│  │ - Final route decision                                               │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│       │                                                                     │
│       ▼                                                                     │
│  Result: { route: "IT_KB_SEARCH", confidence: "high", score: 0.87 }        │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Implementation Phases

### Phase 1: Foundation (FR-2, FR-4, FR-6)

**Duration**: 1-2 days

#### Step 1.1: Dynamic Confidence Thresholds

**File**: `src/config/intentClasses.js`

Add route-specific threshold configuration:

```javascript
const ROUTE_CONFIDENCE_THRESHOLDS = {
    IT_TICKET_MGMT: { high: 0.75, medium: 0.50, low: 0.30 },
    IT_KB_SEARCH:   { high: 0.60, medium: 0.40, low: 0.25 },
    IT_HELP_DESK:   { high: 0.80, medium: 0.60, low: 0.40 },
    MED_KB_SEARCH:  { high: 0.70, medium: 0.50, low: 0.35 },
    MED_DRUG_SEARCH:{ high: 0.70, medium: 0.50, low: 0.35 },
    CHIT_CHAT:      { high: 0.80, medium: 0.60, low: 0.45 },
    OTHER:          { high: 0.50, medium: 0.35, low: 0.20 },
};
```

**File**: `src/Services/IntentClassificationService.js`

Update `determineConfidence()` to use route-specific thresholds.

#### Step 1.2: Enhanced Negative Labels

**File**: `src/config/intentClasses.js`

Update ROUTE_DEFINITIONS with explicit negatives:

```javascript
const ROUTE_DEFINITIONS = [
    {
        name: "IT_KB_SEARCH",
        description: "IT troubleshooting for computers, software, networks. NOT medical, NOT health, NOT drugs, NOT greetings.",
        negatives: ["medical", "health", "drug", "medication", "greeting", "hello"]
    },
    // ...
];
```

#### Step 1.3: Classification Logging

**File**: `src/Services/IntentClassificationService.js`

Add structured logging for all classification decisions:

```javascript
function logClassificationResult(input, result, metadata) {
    logger.info("Classification completed", {
        input: input.substring(0, 100),
        route: result.route,
        confidence: result.confidence,
        score: result.score,
        method: result.classifiedBy,
        domain: result.domain,
        processingTimeMs: metadata.processingTimeMs,
        timestamp: new Date().toISOString()
    });
}
```

---

### Phase 2: Out-of-Scope Detection (FR-1)

**Duration**: 1 day

#### Step 2.1: OOS Detection Labels

**File**: `src/config/intentClasses.js`

```javascript
const OOS_DETECTION_LABELS = [
    "IT or technology support request",
    "Medical or health related question",
    "Casual greeting or conversation",
    "Completely unrelated or random query"  // OOS indicator
];

const OOS_LABEL_TO_SCOPE = {
    "IT or technology support request": "IN_SCOPE_IT",
    "Medical or health related question": "IN_SCOPE_MEDICAL",
    "Casual greeting or conversation": "IN_SCOPE_CHITCHAT",
    "Completely unrelated or random query": "OUT_OF_SCOPE"
};
```

#### Step 2.2: OOS Detection Function

**File**: `src/Services/IntentClassificationService.js`

```javascript
/**
 * Detect if query is out-of-scope before domain classification
 * @param {string} text - User input
 * @returns {Promise<Object>} OOS detection result
 */
async function detectOutOfScope(text) {
    const result = await classify(text, OOS_DETECTION_LABELS);

    const topLabel = result.labels[0];
    const topScore = result.scores[0];
    const scope = OOS_LABEL_TO_SCOPE[topLabel];

    return {
        isOutOfScope: scope === "OUT_OF_SCOPE" && topScore > 0.5,
        scope,
        score: topScore,
        confidence: determineConfidence(topScore, topScore - result.scores[1])
    };
}
```

#### Step 2.3: Integrate OOS into classifyRoute

Update `classifyRoute()` to check OOS first:

```javascript
async function classifyRoute(text, options = {}) {
    // ... keyword matching first ...

    // NEW: Check if out-of-scope before domain classification
    const oosResult = await detectOutOfScope(text);
    if (oosResult.isOutOfScope && oosResult.confidence === "high") {
        return {
            success: true,
            route: "OTHER",
            confidence: oosResult.confidence,
            classifiedBy: "oos-detection",
            oosScore: oosResult.score
        };
    }

    // Continue with domain classification...
}
```

---

### Phase 3: Sentence Embedding Similarity (FR-3)

**Duration**: 2-3 days

#### Step 3.1: Create Embedding Service

**File**: `src/Services/SentenceEmbeddingService.js` (NEW)

```javascript
/**
 * Sentence Embedding Service
 * Uses sentence-transformers for similarity-based classification
 */

const { pipeline } = require("@huggingface/transformers");

class SentenceEmbeddingService {
    constructor() {
        this.embedder = null;
        this.routeEmbeddings = new Map();
        this.isInitialized = false;
    }

    async initialize() {
        this.embedder = await pipeline(
            "feature-extraction",
            "Xenova/all-MiniLM-L6-v2"
        );
        await this.precomputeRouteEmbeddings();
        this.isInitialized = true;
    }

    async embed(text) {
        const output = await this.embedder(text, { pooling: "mean" });
        return Array.from(output.data);
    }

    async precomputeRouteEmbeddings() {
        for (const [route, examples] of Object.entries(ROUTE_EXAMPLES)) {
            const embeddings = await Promise.all(
                examples.map(ex => this.embed(ex))
            );
            this.routeEmbeddings.set(route, embeddings);
        }
    }

    async findNearestRoute(text) {
        const queryEmbed = await this.embed(text);
        let bestRoute = "OTHER";
        let bestScore = -1;

        for (const [route, embeddings] of this.routeEmbeddings) {
            const maxSim = Math.max(
                ...embeddings.map(e => this.cosineSimilarity(queryEmbed, e))
            );
            if (maxSim > bestScore) {
                bestScore = maxSim;
                bestRoute = route;
            }
        }

        return { route: bestRoute, score: bestScore };
    }

    cosineSimilarity(a, b) {
        const dot = a.reduce((sum, val, i) => sum + val * b[i], 0);
        const normA = Math.sqrt(a.reduce((sum, val) => sum + val * val, 0));
        const normB = Math.sqrt(b.reduce((sum, val) => sum + val * val, 0));
        return dot / (normA * normB);
    }
}

module.exports = new SentenceEmbeddingService();
```

#### Step 3.2: Route Examples Configuration

**File**: `src/config/routeExamples.js` (NEW)

```javascript
/**
 * Example queries for each route
 * Used for sentence embedding similarity classification
 */

const ROUTE_EXAMPLES = {
    IT_TICKET_MGMT: [
        "check my ticket status",
        "create a new incident",
        "update ticket INC0012345",
        "what's the status of my request",
        "I want to open a new ticket",
        "cancel my ticket",
        "escalate my incident",
        "show my open tickets"
    ],
    IT_KB_SEARCH: [
        "my computer is slow",
        "Teams keeps crashing",
        "wifi not working",
        "screen is flickering",
        "outlook won't open",
        "password reset",
        "VPN not connecting",
        "printer not working",
        "laptop won't start",
        "Excel freezes"
    ],
    IT_HELP_DESK: [
        "talk to an agent",
        "speak with a human",
        "I need live support",
        "connect me to a real person",
        "transfer to support agent"
    ],
    MED_KB_SEARCH: [
        "what causes diabetes",
        "symptoms of flu",
        "how to treat headache",
        "what is high blood pressure",
        "signs of depression",
        "how to prevent heart disease"
    ],
    MED_DRUG_SEARCH: [
        "side effects of aspirin",
        "what is ibuprofen used for",
        "drug interactions with warfarin",
        "dosage for acetaminophen",
        "is paracetamol safe"
    ],
    CHIT_CHAT: [
        "hello",
        "hi there",
        "good morning",
        "how are you",
        "tell me a joke",
        "what's your name"
    ],
    OTHER: [
        "thank you",
        "buy a car",
        "who is the president",
        "what's 2+2",
        "weather today"
    ]
};

module.exports = { ROUTE_EXAMPLES };
```

---

### Phase 4: Ensemble Voting System (FR-5)

**Duration**: 1-2 days

#### Step 4.1: Ensemble Classifier

**File**: `src/Services/EnsembleClassificationService.js` (NEW)

```javascript
/**
 * Ensemble Classification Service
 * Combines multiple classification methods with weighted voting
 */

const { preClassifyByKeywords, classifyDomain, classifyITSubdomain, classifyMedicalSubdomain } = require("./IntentClassificationService");
const sentenceEmbeddingService = require("./SentenceEmbeddingService");

const ENSEMBLE_WEIGHTS = {
    keyword: 0.30,      // High precision, limited coverage
    zeroShot: 0.40,     // Good coverage, variable precision
    embedding: 0.30     // Good for short queries
};

async function classifyWithEnsemble(text) {
    const results = [];

    // Method 1: Keyword matching
    const keywordResult = preClassifyByKeywords(text);
    if (keywordResult) {
        results.push({
            method: "keyword",
            route: keywordResult.route,
            score: keywordResult.confidence === "high" ? 1.0 : 0.7,
            weight: ENSEMBLE_WEIGHTS.keyword
        });
    }

    // Method 2: Zero-shot hierarchical
    const zeroShotResult = await classifyWithZeroShot(text);
    results.push({
        method: "zeroShot",
        route: zeroShotResult.route,
        score: zeroShotResult.score,
        weight: ENSEMBLE_WEIGHTS.zeroShot
    });

    // Method 3: Sentence embedding similarity
    if (sentenceEmbeddingService.isInitialized) {
        const embeddingResult = await sentenceEmbeddingService.findNearestRoute(text);
        results.push({
            method: "embedding",
            route: embeddingResult.route,
            score: embeddingResult.score,
            weight: ENSEMBLE_WEIGHTS.embedding
        });
    }

    // Weighted voting
    return aggregateVotes(results);
}

function aggregateVotes(results) {
    const routeScores = {};

    for (const result of results) {
        const route = result.route;
        const weightedScore = result.score * result.weight;

        if (!routeScores[route]) {
            routeScores[route] = { totalScore: 0, votes: [] };
        }
        routeScores[route].totalScore += weightedScore;
        routeScores[route].votes.push(result);
    }

    // Find winner
    let bestRoute = "OTHER";
    let bestScore = 0;

    for (const [route, data] of Object.entries(routeScores)) {
        if (data.totalScore > bestScore) {
            bestScore = data.totalScore;
            bestRoute = route;
        }
    }

    return {
        route: bestRoute,
        score: bestScore,
        votes: routeScores,
        methods: results.map(r => r.method)
    };
}

module.exports = { classifyWithEnsemble, ENSEMBLE_WEIGHTS };
```

#### Step 4.2: Update classifyRoute to Use Ensemble

Update `classifyRoute()` to optionally use ensemble:

```javascript
async function classifyRoute(text, options = {}) {
    const { useEnsemble = false, skipKeywords = false } = options;

    if (useEnsemble) {
        return await classifyWithEnsemble(text);
    }

    // ... existing implementation ...
}
```

---

## Testing Strategy

### Unit Tests

| Test Case | Input | Expected Route | Method |
|-----------|-------|----------------|--------|
| IT device query | "my screen is low" | IT_KB_SEARCH | keyword/hierarchical |
| Random query | "buy a bike" | OTHER | oos-detection |
| Medical query | "what causes diabetes" | MED_KB_SEARCH | keyword |
| Short IT query | "screen" | IT_KB_SEARCH | embedding |
| Ticket query | "check ticket INC001" | IT_TICKET_MGMT | keyword |
| Greeting FR | "bonjour" | CHIT_CHAT | keyword |

### Integration Tests

1. Test OOS detection layer independently
2. Test ensemble voting with all methods
3. Test fallback when embedding service unavailable
4. Test performance under load (100 requests)

### Manual Testing

1. Test with real user queries from production logs
2. Verify no regression on existing test cases
3. Test edge cases and ambiguous queries

---

## Configuration

### Environment Variables

```bash
# Feature flags
ENABLE_OOS_DETECTION=true
ENABLE_ENSEMBLE_CLASSIFICATION=true
ENABLE_SENTENCE_EMBEDDINGS=true

# Weights
ENSEMBLE_WEIGHT_KEYWORD=0.30
ENSEMBLE_WEIGHT_ZEROSHOT=0.40
ENSEMBLE_WEIGHT_EMBEDDING=0.30

# Thresholds
OOS_CONFIDENCE_THRESHOLD=0.5
```

### Config File Updates

**File**: `src/config/index.js`

```javascript
classification: {
    enabled: process.env.ENABLE_CLASSIFICATION_API === "true",
    // ... existing ...

    // New settings
    oosDetection: {
        enabled: process.env.ENABLE_OOS_DETECTION === "true",
        threshold: parseFloat(process.env.OOS_CONFIDENCE_THRESHOLD) || 0.5
    },
    ensemble: {
        enabled: process.env.ENABLE_ENSEMBLE_CLASSIFICATION === "true",
        weights: {
            keyword: parseFloat(process.env.ENSEMBLE_WEIGHT_KEYWORD) || 0.30,
            zeroShot: parseFloat(process.env.ENSEMBLE_WEIGHT_ZEROSHOT) || 0.40,
            embedding: parseFloat(process.env.ENSEMBLE_WEIGHT_EMBEDDING) || 0.30
        }
    },
    sentenceEmbedding: {
        enabled: process.env.ENABLE_SENTENCE_EMBEDDINGS === "true",
        model: "Xenova/all-MiniLM-L6-v2"
    }
}
```

---

## Files to Create/Modify

### New Files

| File | Description |
|------|-------------|
| `src/Services/SentenceEmbeddingService.js` | Sentence embedding for similarity search |
| `src/Services/EnsembleClassificationService.js` | Ensemble voting system |
| `src/config/routeExamples.js` | Example queries for each route |

### Modified Files

| File | Changes |
|------|---------|
| `src/config/intentClasses.js` | Add OOS labels, route thresholds, negative labels |
| `src/config/index.js` | Add new configuration options |
| `src/Services/IntentClassificationService.js` | Add OOS detection, integrate ensemble |
| `src/server.js` | Initialize sentence embedding service on startup |

---

## Rollout Plan

### Phase 1: Foundation (Low Risk)
- Deploy dynamic thresholds
- Deploy enhanced logging
- Monitor classification accuracy

### Phase 2: OOS Detection (Medium Risk)
- Deploy behind feature flag
- A/B test with 10% traffic
- Monitor false positive rate

### Phase 3: Sentence Embeddings (Medium Risk)
- Deploy behind feature flag
- Measure performance impact
- Validate example quality

### Phase 4: Full Ensemble (Higher Risk)
- Enable ensemble mode
- Monitor accuracy improvement
- Tune weights based on results

---

## Success Metrics

| Metric | Current | Target |
|--------|---------|--------|
| Accuracy on test set | ~85% | > 92% |
| OOS detection precision | N/A | > 90% |
| Classification latency (P95) | ~400ms | < 500ms |
| Misclassification rate | ~15% | < 8% |

---

## Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| Sentence embedding model load time | Slow startup | Preload in background, cache embeddings |
| Ensemble increases latency | Slower responses | Parallel execution, caching |
| OOS detection too aggressive | Good queries rejected | Tune threshold, add more examples |
| Memory increase | Resource usage | Lazy loading, model quantization |

---

## Timeline

| Phase | Duration | Dependencies |
|-------|----------|--------------|
| Phase 1: Foundation | 1-2 days | None |
| Phase 2: OOS Detection | 1 day | Phase 1 |
| Phase 3: Sentence Embeddings | 2-3 days | Phase 1 |
| Phase 4: Ensemble | 1-2 days | Phase 2, 3 |
| Testing & Tuning | 2 days | All phases |
| **Total** | **7-10 days** | |
