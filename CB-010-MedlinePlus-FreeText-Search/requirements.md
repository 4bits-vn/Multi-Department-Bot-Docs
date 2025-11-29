# CB-009: MedlinePlus Free-Text Search with Medical NER

## Ticket Information
- **Ticket**: CB-009-MedlinePlus-FreeText-Search
- **Type**: Feature / Enhancement
- **Priority**: High
- **Created**: 2024-11-29
- **Dependencies**: CB-007 (Drug Query Flow), CB-008 (LangFlow Handler Refactor)

## Overview

Implement a comprehensive medical information query system that:
1. Extracts key medical terms from user questions using Named Entity Recognition (NER)
2. Attempts to normalize drug terms using RxNorm API
3. Falls back to MedlinePlus free-text search for broader medical queries
4. Parses and returns structured, easy-to-use health information

This extends the existing drug query functionality (CB-007) to handle general medical questions beyond just drug names.

## Current State

### Existing Services
- `RxNormService.js` - Drug name normalization via RxNorm API
- `MedlinePlusService.js` - Drug info via RXCUI code-based queries
- `DrugQueryService.js` - Orchestrates RxNorm + MedlinePlus for drug queries

### Limitation
Current implementation only supports drug queries via RXCUI codes. It cannot handle:
- General medical questions ("What is diabetes?")
- Symptom queries ("Why do my eyes feel dry?")
- Condition explanations ("Explain insulin resistance")
- Vague or complex medical questions

## Target Workflow

```
User question
      ↓
Extract key medical terms (NER)
      ↓
Classify entity types (drug, condition, symptom, etc.)
      ↓
For drugs: Try RxNorm normalization
      ↓
If RxNorm succeeds → Use RXCUI-based query (existing flow)
      ↓
If no drug match OR non-drug query → MedlinePlus free-text search
      ↓
Parse MedlinePlus response
      ↓
Return structured, user-friendly payload
```

## Functional Requirements

### FR-1: Medical Named Entity Recognition (NER)
Extract medical entities from user input text:

**Entity Types:**
| Entity Type | Examples | Normalizer |
|-------------|----------|------------|
| `drug` | metformin, aspirin, tylenol | RxNorm API |
| `condition` | diabetes, asthma, hypertension | Free-text search |
| `symptom` | headache, dry eyes, fatigue | Free-text search |
| `measurement` | blood pressure, A1C | Free-text search |
| `anatomy` | heart, lungs, liver | Free-text search |
| `procedure` | MRI, colonoscopy | Free-text search |

**NER Output Structure:**
```json
{
  "terms": [
    { "text": "metformin", "type": "drug", "confidence": 0.95 },
    { "text": "type 2 diabetes", "type": "condition", "confidence": 0.92 }
  ],
  "originalQuery": "What are the side effects of metformin for type 2 diabetes?"
}
```

**NER Implementation Options (in order of preference):**
1. **LLM-based extraction**: Use existing LangFlow integration with a medical NER prompt
2. **Azure Text Analytics for Health**: Requires Azure Language Service API key
3. **Keyword-based fallback**: Simple regex/dictionary matching

### FR-2: Drug Term Normalization
For entities classified as `drug`:
1. Call RxNorm `approximateTerm` API (already implemented)
2. If match found with score > 70 → use RXCUI-based MedlinePlus query
3. If no good match → fall back to free-text search

### FR-3: MedlinePlus Free-Text Search
New API integration for free-text medical queries:

**API Endpoint:**
```
https://wsearch.nlm.nih.gov/ws/query?db=healthTopics&term=<query>&format=json
```

**Parameters:**
| Parameter | Required | Description |
|-----------|----------|-------------|
| `db` | Yes | Database: `healthTopics` (English) or `healthTopicsSpanish` |
| `term` | Yes | URL-encoded search query |
| `format` | Yes | Response format: `json` |
| `retmax` | No | Maximum results (default: 10) |

**Response Parsing:**
Extract from MedlinePlus response:
- Title
- Summary/description
- URL to full article
- Categories/tags
- Health topic sections (if available)

### FR-4: Structured Response Format
Return consistent, structured responses:

```json
{
  "success": true,
  "query": "What are the symptoms of asthma?",
  "source": "medlineplus_freetext",
  "result": {
    "title": "Asthma",
    "summary": "Asthma is a chronic lung disease...",
    "url": "https://medlineplus.gov/asthma.html",
    "categories": ["Respiratory Diseases", "Lung Diseases"],
    "sections": {
      "symptoms": ["Wheezing", "Shortness of breath", "Chest tightness"],
      "causes": ["Allergens", "Exercise", "Cold air"],
      "treatment": ["Inhalers", "Medications"],
      "whenToSeeDoctor": ["Severe breathing difficulty"]
    }
  },
  "alternativeTopics": [...],
  "processingTimeMs": 245
}
```

### FR-5: Caching Strategy
Implement caching for performance:

| Cache Type | TTL | Key Format |
|------------|-----|------------|
| NER results | 1 hour | `ner:${hash(query)}` |
| RxNorm results | 24 hours | `rxnorm:${term}` |
| MedlinePlus free-text | 7 days | `medline:freetext:${term}` |
| MedlinePlus RXCUI | 7 days | `medline:rxcui:${rxcui}` |

**Cache Implementation:**
- Use existing Redis service if available
- Fallback to in-memory LRU cache

## Non-Functional Requirements

### NFR-1: Performance
- NER extraction: < 500ms (LLM) or < 50ms (keyword-based)
- RxNorm lookup: < 2s
- MedlinePlus search: < 3s
- End-to-end response: < 5s (with caching: < 500ms)

### NFR-2: Reliability
- Graceful fallbacks when APIs are unavailable
- Retry logic with exponential backoff
- Circuit breaker pattern for external APIs

### NFR-3: No Breaking Changes
- Existing drug query APIs unchanged
- New functionality exposed via new endpoints
- All existing CB-008 handler logic preserved

### NFR-4: Rate Limiting
- MedlinePlus: No official rate limit, but respect 1 req/sec guideline
- RxNorm: No rate limit, but use caching
- Implement request throttling if needed

## API Endpoints

### New Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/medical/search` | Main medical query endpoint |
| POST | `/api/medical/ner` | Extract medical entities only |
| GET | `/api/medical/topic/:topic` | Get specific health topic |
| GET | `/api/medical/health` | Health check for medical services |

### Example Requests

**Medical Search:**
```bash
POST /api/medical/search
{
  "query": "What are the side effects of ibuprofen?",
  "language": "en"
}
```

**NER Extraction:**
```bash
POST /api/medical/ner
{
  "text": "I have diabetes and take metformin"
}
```

## Acceptance Criteria

- [ ] Medical NER extracts drug, condition, symptom entities
- [ ] Drug terms successfully normalized via RxNorm when possible
- [ ] MedlinePlus free-text search returns relevant health topics
- [ ] Structured response format consistent across all query types
- [ ] Caching reduces repeat query response times by >80%
- [ ] All existing drug query APIs continue to work
- [ ] New endpoints documented and tested
- [ ] No modifications to CB-008 LangFlow handler structure

## Dependencies

### External APIs (No API Key Required)
- **MedlinePlus Web Service**: `https://wsearch.nlm.nih.gov/ws/query`
- **MedlinePlus Connect**: `https://connect.medlineplus.gov/service` (existing)
- **RxNorm REST API**: `https://rxnav.nlm.nih.gov/REST` (existing)

### Optional (Requires API Key)
- **Azure Text Analytics for Health**: For advanced medical NER
  - Requires: `AZURE_LANGUAGE_KEY` and `AZURE_LANGUAGE_ENDPOINT`
  - Falls back to keyword-based NER if not configured

### Internal Dependencies
- `RxNormService.js` (existing)
- `MedlinePlusService.js` (existing, will extend)
- `DrugQueryService.js` (existing)
- `Logger.js` (existing)
- Redis service (if available)

## References

### Official Documentation
- **MedlinePlus Web Services**: https://medlineplus.gov/about/developers/webservices/
- **MedlinePlus Connect**: https://medlineplus.gov/medlineplus-connect/technical-information/
- **RxNorm REST API**: https://lhncbc.nlm.nih.gov/RxNav/APIs/
- **Azure Text Analytics for Health**: https://learn.microsoft.com/en-us/azure/ai-services/language-service/text-analytics-for-health/overview

### Related Tickets
- CB-007: Drug Query Flow
- CB-008: LangFlow Handler Refactor
