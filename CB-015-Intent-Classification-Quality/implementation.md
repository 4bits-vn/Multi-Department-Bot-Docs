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

## Future Enhancements (Pending)

From the implementation plan, the following phases are still pending:

- **Phase 3**: Sentence Embedding Similarity (SetFit-like approach)
- **Phase 4**: Ensemble Voting System
- **Phase 5**: Continuous Learning / Feedback Loop

## References

- [CB-015 Requirements](./requirements.md)
- [CB-015 Implementation Plan](./implementation-plan.md)
- [CB-014 Zero-Shot Classification](../CB-014-Zero-Shot-Classification/)
