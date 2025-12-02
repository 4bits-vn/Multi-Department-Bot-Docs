# CB-015: Intent Classification Quality - Implementation

## Summary

Implemented Phase 1 and Phase 2 of the intent classification quality improvements to achieve higher accuracy and reliability.

## Changes Implemented

### Phase 1: Foundation

#### 1.1 Route-Specific Confidence Thresholds

Added `ROUTE_CONFIDENCE_THRESHOLDS` in `src/config/intentClasses.js`:

```javascript
const ROUTE_CONFIDENCE_THRESHOLDS = {
    IT_TICKET_MGMT: { high: 0.75, medium: 0.50, low: 0.30 },
    IT_KB_SEARCH:   { high: 0.55, medium: 0.35, low: 0.20 },  // More lenient - primary IT route
    IT_HELP_DESK:   { high: 0.80, medium: 0.60, low: 0.40 },  // Very strict
    MED_KB_SEARCH:  { high: 0.65, medium: 0.45, low: 0.30 },
    MED_DRUG_SEARCH:{ high: 0.65, medium: 0.45, low: 0.30 },
    CHIT_CHAT:      { high: 0.75, medium: 0.55, low: 0.40 },  // Strict
    OTHER:          { high: 0.50, medium: 0.35, low: 0.20 },  // Most lenient
};
```

#### 1.2 Enhanced ROUTE_DEFINITIONS with Negative Labels

Updated route definitions to include explicit negatives for better classification:

```javascript
{
    name: "IT_KB_SEARCH",
    description: "IT troubleshooting... NOT for medical, NOT for drugs, NOT for tickets, NOT for greetings.",
    negatives: ["medical", "health", "drug", "medication", ...],
    keywords: ["computer", "laptop", "software", ...],
}
```

#### 1.3 Updated determineConfidence() Function

Modified to support route-specific thresholds:

```javascript
function determineConfidence(score, gap, route = null) {
    const thresholds = route && ROUTE_CONFIDENCE_THRESHOLDS[route]
        ? ROUTE_CONFIDENCE_THRESHOLDS[route]
        : { high: 0.8, medium: 0.5, low: 0.3 };
    // ...
}
```

#### 1.4 Structured Classification Logging

Added `logClassificationResult()` function for consistent logging:

```javascript
function logClassificationResult(input, result, metadata = {}) {
    logger.info("Classification result", {
        input, route, confidence, score, classifiedBy, domain, appliedRule, timestamp
    });
}
```

### Phase 2: Out-of-Scope Detection

#### 2.1 OOS Detection Labels

Added in `src/config/intentClasses.js`:

```javascript
const OOS_DETECTION_LABELS = [
    "IT or technology support request",
    "Medical or health related question",
    "Casual greeting or conversation",
    "Completely unrelated or random query",
];

const OOS_LABEL_TO_SCOPE = {
    "IT or technology support request": "IN_SCOPE_IT",
    "Medical or health related question": "IN_SCOPE_MEDICAL",
    "Casual greeting or conversation": "IN_SCOPE_CHITCHAT",
    "Completely unrelated or random query": "OUT_OF_SCOPE",
};

const OOS_THRESHOLD = 0.5;
```

#### 2.2 detectOutOfScope() Function

New function in `IntentClassificationService.js`:

```javascript
async function detectOutOfScope(text) {
    const result = await classify(text, OOS_DETECTION_LABELS);
    const scope = OOS_LABEL_TO_SCOPE[result.labels[0]];
    const isOutOfScope = scope === "OUT_OF_SCOPE" && score >= OOS_THRESHOLD;
    return { success, isOutOfScope, scope, label, score, confidence };
}
```

#### 2.3 Integration into classifyRoute()

OOS detection now runs before domain classification:

```
User Input
    │
    ▼
┌─────────────────────┐
│ 1. Keyword Matching │ ← Fast path (existing)
└─────────────────────┘
    │ No match
    ▼
┌─────────────────────┐
│ 2. OOS Detection    │ ← NEW: Filter random queries
└─────────────────────┘
    │ In-scope
    ▼
┌─────────────────────┐
│ 3. Hierarchical     │ ← Domain → Sub-domain
│    Classification   │
└─────────────────────┘
    │
    ▼
  Result
```

## Files Modified

| File | Changes |
|------|---------|
| `src/config/intentClasses.js` | Added ROUTE_CONFIDENCE_THRESHOLDS, OOS_DETECTION_LABELS, OOS_LABEL_TO_SCOPE, OOS_THRESHOLD, updated ROUTE_DEFINITIONS with negatives |
| `src/Services/IntentClassificationService.js` | Added detectOutOfScope(), logClassificationResult(), updated determineConfidence(), integrated OOS into classifyRoute(), updated exports |

## Test Results

| # | Query | Expected | Actual | Status |
|---|-------|----------|--------|--------|
| 1 | "buy a bike" | OTHER | OTHER | ✅ |
| 2 | "who is the president" | CHIT_CHAT/OTHER | CHIT_CHAT | ✅ |
| 3 | "my screen is low" | IT_KB_SEARCH | IT_KB_SEARCH | ✅ |
| 4 | "check my mac" | IT_KB_SEARCH | IT_KB_SEARCH | ✅ |
| 5 | "ticket status please" | IT_TICKET_MGMT | IT_TICKET_MGMT | ✅ |
| 6 | "what causes diabetes" | MED_KB_SEARCH | MED_KB_SEARCH | ✅ |
| 7 | "side effects of aspirin" | MED_DRUG_SEARCH | MED_DRUG_SEARCH | ✅ |
| 8 | "Bonjour" (French) | CHIT_CHAT | CHIT_CHAT | ✅ |
| 9 | "kiểm tra ticket" (Vietnamese) | IT_TICKET_MGMT | IT_TICKET_MGMT | ✅ |
| 10 | "hello" | CHIT_CHAT | CHIT_CHAT | ✅ |
| 11 | "talk to an agent" | IT_HELP_DESK | IT_HELP_DESK | ✅ |

## New API Exports

```javascript
// Out-of-Scope Detection
detectOutOfScope(text)              // Detect if query is out-of-scope

// Helpers
logClassificationResult(input, result, metadata)  // Structured logging

// Configuration
ROUTE_CONFIDENCE_THRESHOLDS         // Route-specific thresholds
OOS_DETECTION_LABELS                // OOS detection labels
OOS_LABEL_TO_SCOPE                  // OOS label mapping
OOS_THRESHOLD                       // OOS detection threshold
```

### Phase 3: Sentence Embedding Similarity

#### 3.1 Route Examples Configuration

Created `src/config/routeExamples.js` with multilingual examples for each route.

#### 3.2 Sentence Embedding Service

Created `src/Services/SentenceEmbeddingService.js`:

```javascript
// Key features:
- Uses Xenova/all-MiniLM-L6-v2 model (384-dimensional embeddings)
- Precomputes embeddings for all route examples on initialization
- Finds nearest route using cosine similarity
- Supports dynamic example addition for feedback learning
```

### Phase 4: Ensemble Voting System

#### 4.1 Ensemble Classification Service

Created `src/Services/EnsembleClassificationService.js`:

```javascript
const ENSEMBLE_WEIGHTS = {
    keyword: 0.35,      // High precision, limited coverage
    zeroShot: 0.40,     // Good coverage, variable precision
    embedding: 0.25,    // Good for short queries
};
```

#### 4.2 API Endpoints

Added new endpoints in `routes/classification.routes.js`:

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/classification/ensemble` | POST | Classify using ensemble voting |
| `/api/classification/ensemble/init` | POST | Initialize embedding service |
| `/api/classification/ensemble/stats` | GET | Get ensemble service stats |

#### 4.3 Integration with classifyRoute

Added `useEnsemble` option to `classifyRoute()`:

```javascript
// Enable via option
classifyRoute(text, { useEnsemble: true })

// Or enable globally via environment
CLASSIFICATION_ENSEMBLE_ENABLED=true
```

## Ensemble Test Results

| # | Query | Route | Confidence | Unanimous |
|---|-------|-------|------------|-----------|
| 1 | "buy a bike" | OTHER | medium | ✅ |
| 2 | "Teams keeps crashing" | IT_KB_SEARCH | medium | ✅ |
| 3 | "check my mac" | IT_KB_SEARCH | medium | ✅ |
| 4 | "ticket status please" | IT_TICKET_MGMT | medium | ✅ |
| 5 | "my screen is low" | IT_KB_SEARCH | medium | ✅ |
| 6 | "what causes diabetes" | MED_KB_SEARCH | medium | ✅ |
| 7 | "side effects of aspirin" | MED_DRUG_SEARCH | medium | ✅ |
| 8 | "hello" | CHIT_CHAT | medium | ✅ |
| 9 | "talk to an agent" | IT_HELP_DESK | medium | ✅ |

## Bug Fix: Application Settings/Mode Queries

### Issue

User intent **"offline mode in outlook"** was incorrectly classified as `OTHER` instead of `IT_KB_SEARCH`.

### Root Cause Analysis

1. **Keyword patterns missing**: Existing IT_KB_SEARCH patterns expected `outlook` followed by problem words (e.g., "outlook is not working"). But queries like "offline mode in outlook" have the application name at the end.

2. **Route examples gap**: All embedding examples were problem statements ("Teams keeps crashing"), not settings/configuration queries.

3. **Pattern coverage**: No patterns for application settings, mode, or configuration queries.

### Fix Applied

#### 1. New Strong Keyword Patterns (English)

```javascript
// Application settings/mode/feature queries
/\b(offline|online|cached|sync)\s+(mode|settings?|option)\s+(in|for|on|with)\s+(outlook|teams|excel|word|office|sharepoint|onedrive)/i,
/\b(outlook|teams|excel|word|office|sharepoint|onedrive)\s+(offline|online|settings?|configuration|setup|mode)/i,
/\b(how\s+to|enable|disable|turn\s+on|turn\s+off|configure|setup|set\s+up)\s+.{0,20}(outlook|teams|excel|word|office|vpn|wifi)/i,
```

#### 2. Application Names as Moderate Signal

```javascript
// Application name alone should be a moderate IT signal
/\b(outlook|teams|excel|word|office|sharepoint|onedrive|vpn|wifi|printer)\b/i,
```

#### 3. Multilingual Support (French & Vietnamese)

Added equivalent patterns for French and Vietnamese to maintain multilingual consistency.

#### 4. Route Examples Updated

Added new examples to `src/config/routeExamples.js`:
- "offline mode in outlook"
- "outlook offline settings"
- "how to enable offline mode"
- "configure outlook settings"
- "mode hors ligne outlook" (French)
- "cài đặt outlook" (Vietnamese)

### Test Results After Fix

| # | Query | Expected | Actual | Status |
|---|-------|----------|--------|--------|
| 1 | "offline mode in outlook" | IT_KB_SEARCH | IT_KB_SEARCH | ✅ |
| 2 | "outlook offline settings" | IT_KB_SEARCH | IT_KB_SEARCH | ✅ |
| 3 | "how to enable offline mode in teams" | IT_KB_SEARCH | IT_KB_SEARCH | ✅ |
| 4 | "configure outlook settings" | IT_KB_SEARCH | IT_KB_SEARCH | ✅ |
| 5 | "teams settings not working" | IT_KB_SEARCH | IT_KB_SEARCH | ✅ |
| 6 | "mode hors ligne outlook" (FR) | IT_KB_SEARCH | IT_KB_SEARCH | ✅ |
| 7 | "cài đặt outlook" (VI) | IT_KB_SEARCH | IT_KB_SEARCH | ✅ |

### Files Modified

| File | Changes |
|------|---------|
| `src/Services/IntentClassificationService.js` | Added application settings/mode patterns for EN, FR, VI |
| `src/config/routeExamples.js` | Added settings/configuration examples for embedding similarity |

---

## Future Enhancements (Pending)

- **Phase 5**: Continuous Learning / Feedback Loop

## References

- [CB-015 Requirements](./requirements.md)
- [CB-015 Implementation Plan](./implementation-plan.md)
- [CB-014 Zero-Shot Classification](../CB-014-Zero-Shot-Classification/)
