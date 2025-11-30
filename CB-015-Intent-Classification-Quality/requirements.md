# CB-015: Intent Classification Quality Improvements

## Ticket Information

| Field | Value |
|-------|-------|
| **Ticket ID** | CB-015 |
| **Title** | Intent Classification Quality Improvements |
| **Type** | Enhancement |
| **Priority** | High |
| **Status** | Planning |
| **Created** | 2024-11-30 |
| **Assignee** | TBD |

## Overview

Enhance the intent classification system (`classifyRoute`) to achieve higher accuracy and reliability for production use. The current implementation uses a hybrid approach (keyword matching + hierarchical zero-shot classification), but still has quality gaps in certain scenarios.

### Current State

- **Keyword Matching**: Fast, high accuracy for common patterns
- **Hierarchical Zero-Shot**: Domain → Sub-domain classification using DeBERTa model
- **Business Rules**: Fallback logic for low-confidence cases
- **Multi-language**: English, French, Vietnamese keyword support

### Problems to Address

1. **Overconfident/underconfident scores** - Zero-shot model scores are not well-calibrated
2. **Out-of-scope detection** - Random queries still get forced into a route
3. **Similar domain confusion** - IT vs Medical confusion for ambiguous queries
4. **Short query handling** - Brief inputs like "screen" are hard to classify
5. **No learning mechanism** - Static model doesn't improve from mistakes

## Functional Requirements

### FR-1: Out-of-Scope (OOS) Detection Layer

**Priority**: High

- Add explicit OOS detection as the first classification step
- Detect truly unrelated queries before domain classification
- Return `OTHER` with high confidence for random/unrelated queries
- Reduce false positives for in-scope domains

**Acceptance Criteria**:
- "buy a bike" → OTHER with confidence > 0.7
- "what's your favorite color" → OTHER or CHIT_CHAT
- "my computer is slow" → Still routes to IT_KB_SEARCH correctly

### FR-2: Dynamic Confidence Thresholds per Route

**Priority**: High

- Implement route-specific confidence thresholds
- Account for varying classification difficulty per domain
- Allow configuration via environment variables or config file

**Acceptance Criteria**:
- Different thresholds for IT, Medical, Chitchat, Other routes
- Configurable threshold values
- Improved precision/recall balance

### FR-3: Sentence Embedding Similarity (Example-Based)

**Priority**: Medium

- Add sentence embedding-based classification using example similarity
- Pre-compute embeddings for representative queries per route
- Use as secondary signal alongside zero-shot classification

**Acceptance Criteria**:
- Fast inference (< 50ms for embedding lookup)
- At least 10 example queries per route
- Improves accuracy on short/ambiguous queries

### FR-4: Negative Class Labels Enhancement

**Priority**: Medium

- Add explicit negative context to classification labels
- Improve decision boundaries between similar domains
- Reduce IT vs Medical confusion

**Acceptance Criteria**:
- Labels include explicit exclusions (NOT medical, NOT IT, etc.)
- Reduced misclassification between similar domains
- No regression on existing test cases

### FR-5: Ensemble Voting System

**Priority**: Medium

- Combine multiple classification methods with weighted voting
- Include: Keyword, Zero-shot NLI, Sentence Embedding
- Configurable weights per method

**Acceptance Criteria**:
- Aggregates results from 3+ methods
- Configurable weights
- Higher overall accuracy than single method

### FR-6: Classification Logging & Analytics

**Priority**: Low

- Log all classification decisions with details
- Track confidence distribution over time
- Enable analysis of misclassifications

**Acceptance Criteria**:
- Structured logging of all classification results
- Include: input, route, confidence, method, timestamp
- Exportable for analysis

## Non-Functional Requirements

### NFR-1: Performance

- Classification latency < 500ms (P95)
- No significant increase in memory usage
- Efficient caching of embeddings

### NFR-2: Reliability

- Graceful degradation if embedding service fails
- Fallback to simpler methods on error
- No crashes on malformed input

### NFR-3: Maintainability

- Well-documented code with JSDoc
- Unit tests for each classification method
- Easy to add new routes/examples

### NFR-4: Configurability

- Environment variables for thresholds
- Feature flags for each improvement
- Easy to enable/disable features

## Dependencies

- Existing: `@huggingface/transformers` (zero-shot classification)
- Existing: `IntentClassificationService.js`
- Existing: `ZeroShotClassificationService.js`
- New: Sentence embedding model (e.g., `Xenova/all-MiniLM-L6-v2`)

## Out of Scope

- Fine-tuning custom models
- Multi-intent detection (future enhancement)
- Context-aware classification with conversation history (future enhancement)
- Continuous learning / feedback loop (future enhancement)

## References

- [Zero-Shot Classification Best Practices](https://huggingface.co/facebook/bart-large-mnli)
- [SetFit: Few-Shot Learning](https://github.com/huggingface/setfit)
- [Confidence Calibration for BERT](https://medium.com/expedia-group-tech/calibrating-bert-based-intent-classification-models-part-2-a44752fc655a)
- [Out-of-Scope Detection](https://rasa.com/docs/rasa/2.x/fallback-handoff/)
- Current implementation: `src/Services/IntentClassificationService.js`
