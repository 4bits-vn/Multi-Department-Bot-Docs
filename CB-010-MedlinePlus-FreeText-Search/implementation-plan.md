# CB-009: Implementation Plan

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          Medical Query Flow                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  User Question                                                               │
│       │                                                                      │
│       ▼                                                                      │
│  ┌──────────────────┐                                                        │
│  │ MedicalQueryOrch │ ◄── Main orchestrator                                  │
│  │    estrator      │                                                        │
│  └────────┬─────────┘                                                        │
│           │                                                                  │
│           ▼                                                                  │
│  ┌──────────────────┐     ┌─────────────────┐                               │
│  │ MedicalNERService│────►│ LLM (LangFlow)  │  (Primary)                    │
│  │                  │     │ or Keyword      │  (Fallback)                   │
│  └────────┬─────────┘     └─────────────────┘                               │
│           │                                                                  │
│           ▼                                                                  │
│  ┌──────────────────┐                                                        │
│  │ Entity Type?     │                                                        │
│  └────────┬─────────┘                                                        │
│           │                                                                  │
│     ┌─────┴─────────────────────┐                                           │
│     │                           │                                           │
│     ▼                           ▼                                           │
│  Drug Entity              Non-Drug Entity                                   │
│     │                           │                                           │
│     ▼                           │                                           │
│  ┌──────────────────┐           │                                           │
│  │ RxNormService    │           │                                           │
│  │ (approximateTerm)│           │                                           │
│  └────────┬─────────┘           │                                           │
│           │                     │                                           │
│     ┌─────┴─────┐               │                                           │
│     │           │               │                                           │
│  RXCUI Found  Not Found         │                                           │
│     │           │               │                                           │
│     ▼           └───────────────┤                                           │
│  ┌──────────────────┐           │                                           │
│  │ MedlinePlus      │           │                                           │
│  │ (RXCUI-based)    │           │                                           │
│  │ [Existing]       │           │                                           │
│  └────────┬─────────┘           │                                           │
│           │                     │                                           │
│           │                     ▼                                           │
│           │           ┌──────────────────┐                                  │
│           │           │ MedlinePlus      │                                  │
│           │           │ WebSearchService │                                  │
│           │           │ [New]            │                                  │
│           │           └────────┬─────────┘                                  │
│           │                    │                                            │
│           └──────────┬─────────┘                                            │
│                      │                                                      │
│                      ▼                                                      │
│           ┌──────────────────┐                                              │
│           │ Response Parser  │                                              │
│           │ & Formatter      │                                              │
│           └────────┬─────────┘                                              │
│                    │                                                        │
│                    ▼                                                        │
│           ┌──────────────────┐                                              │
│           │ Cache Service    │                                              │
│           │ (Redis/Memory)   │                                              │
│           └────────┬─────────┘                                              │
│                    │                                                        │
│                    ▼                                                        │
│           Structured Response                                               │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Proposed Directory Structure

```
botframework/src/
├── Services/
│   ├── Medical/
│   │   ├── index.js                      # Exports all medical services
│   │   ├── MedicalNERService.js          # Medical entity extraction
│   │   ├── MedlinePlusWebSearchService.js # Free-text search
│   │   ├── MedicalQueryOrchestrator.js   # Main orchestrator
│   │   └── MedicalCacheService.js        # Caching layer
│   ├── RxNormService.js                  # (Existing - no changes)
│   ├── MedlinePlusService.js             # (Existing - no changes)
│   └── DrugQueryService.js               # (Existing - no changes)
│
├── controllers/
│   ├── medical.controller.js             # New medical API controller
│   └── drug.controller.js                # (Existing - no changes)
│
└── routes/
    ├── medical.routes.js                 # New medical API routes
    └── drug.routes.js                    # (Existing - no changes)
```

## Implementation Steps

### Step 1: Create MedlinePlus Web Search Service

**File:** `Services/Medical/MedlinePlusWebSearchService.js`

**API Details:**
```
Base URL: https://wsearch.nlm.nih.gov/ws/query
Method: GET
Response: JSON

Parameters:
- db: "healthTopics" (English) or "healthTopicsSpanish" (Spanish)
- term: URL-encoded search query
- format: "json" (required for JSON response)
- retmax: Maximum results (optional, default ~10)
```

**Example Request:**
```
GET https://wsearch.nlm.nih.gov/ws/query?db=healthTopics&term=diabetes&format=json
```

**Response Structure (to parse):**
```json
{
  "feed": {
    "title": "MedlinePlus Health Topics",
    "xsi:noNamespaceSchemaLocation": "...",
    "base": "https://wsearch.nlm.nih.gov/ws/query",
    "entry": [
      {
        "id": "https://medlineplus.gov/diabetes.html",
        "title": { "_value": "Diabetes" },
        "link": [{ "href": "https://medlineplus.gov/diabetes.html" }],
        "summary": { "_value": "Diabetes is a disease..." },
        "content": { "_value": "<full HTML content>" },
        "category": [
          { "scheme": "urn:medlineplus:mesh", "term": "Diabetes Mellitus" }
        ]
      }
    ]
  }
}
```

### Step 2: Create Medical NER Service

**File:** `Services/Medical/MedicalNERService.js`

**Implementation Options:**

#### Option A: LLM-based (via LangFlow)
```javascript
const NER_PROMPT = `
Extract medical entities from the following text.
Return a JSON object with the following structure:
{
  "terms": [
    { "text": "<entity>", "type": "<type>", "confidence": <0-1> }
  ]
}

Entity types: drug, condition, symptom, measurement, anatomy, procedure

Text: "{userInput}"
`;
```

#### Option B: Keyword-based (Fallback)
Use medical term dictionaries and regex patterns:
- Common drug suffixes: -in, -ol, -ide, -ine, -ate
- Condition keywords: disease, syndrome, disorder, -itis, -osis
- Symptom keywords: pain, ache, swelling, rash, fever

### Step 3: Create Medical Query Orchestrator

**File:** `Services/Medical/MedicalQueryOrchestrator.js`

**Main Flow:**
```javascript
async function queryMedical(userQuery, options = {}) {
    // 1. Check cache
    const cached = await cacheService.get(`medical:${hash(userQuery)}`);
    if (cached) return cached;

    // 2. Extract medical entities
    const nerResult = await medicalNERService.extractEntities(userQuery);

    // 3. Determine best search strategy
    if (nerResult.terms.length === 0) {
        // No specific medical terms - use full query for free-text search
        return await searchMedlinePlusFreeText(userQuery);
    }

    // 4. Process each entity type
    const results = [];
    for (const term of nerResult.terms) {
        if (term.type === 'drug') {
            // Try RxNorm normalization first
            const rxResult = await rxNormService.findBestMatch(term.text);
            if (rxResult.found && rxResult.bestMatch.matchScore > 70) {
                // Use RXCUI-based query
                const drugInfo = await medlinePlusService.getDrugInfo(
                    rxResult.bestMatch.ingredientRxcui
                );
                results.push({ type: 'drug', source: 'rxcui', data: drugInfo });
            } else {
                // Fallback to free-text
                const ftResult = await searchMedlinePlusFreeText(term.text);
                results.push({ type: 'drug', source: 'freetext', data: ftResult });
            }
        } else {
            // Non-drug entities - use free-text search
            const ftResult = await searchMedlinePlusFreeText(term.text);
            results.push({ type: term.type, source: 'freetext', data: ftResult });
        }
    }

    // 5. Combine and format results
    const response = formatResults(results, userQuery);

    // 6. Cache result
    await cacheService.set(`medical:${hash(userQuery)}`, response, { ttl: 3600 });

    return response;
}
```

### Step 4: Create Medical Cache Service

**File:** `Services/Medical/MedicalCacheService.js`

**Features:**
- Redis primary (if configured)
- In-memory LRU fallback
- TTL management per cache type

```javascript
const CACHE_TTL = {
    ner: 60 * 60,           // 1 hour
    rxnorm: 24 * 60 * 60,   // 24 hours
    freetext: 7 * 24 * 60 * 60  // 7 days
};
```

### Step 5: Create API Controller & Routes

**File:** `controllers/medical.controller.js`

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/medical/search` | GET | Main medical query |
| `/api/medical/ner` | GET | Extract entities only |
| `/api/medical/topic/:topic` | GET | Get specific topic |
| `/api/medical/topics` | GET | Search health topics |
| `/api/medical/formatted` | GET | Get formatted display text |
| `/api/medical/health` | GET | Health check |

### Step 6: Register Routes

Update `routes/index.js` to include medical routes.

## API Reference

### MedlinePlus Web Service

**Official Documentation:** https://medlineplus.gov/about/developers/webservices/

**Endpoint:** `https://wsearch.nlm.nih.gov/ws/query`

**Parameters:**
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `db` | string | Yes | Database to search |
| `term` | string | Yes | Search term (URL-encoded) |
| `format` | string | Yes | Response format (`json`) |
| `retmax` | number | No | Max results to return |

**Database Options:**
- `healthTopics` - English health topics
- `healthTopicsSpanish` - Spanish health topics

**No API Key Required** ✓

### RxNorm REST API (Existing)

**Official Documentation:** https://lhncbc.nlm.nih.gov/RxNav/APIs/

**Endpoint:** `https://rxnav.nlm.nih.gov/REST`

**No API Key Required** ✓

### MedlinePlus Connect (Existing)

**Official Documentation:** https://medlineplus.gov/medlineplus-connect/technical-information/

**Endpoint:** `https://connect.medlineplus.gov/service`

**No API Key Required** ✓

## File Creation Order

1. `Services/Medical/MedicalCacheService.js` - No dependencies
2. `Services/Medical/MedlinePlusWebSearchService.js` - Depends on cache
3. `Services/Medical/MedicalNERService.js` - Standalone NER
4. `Services/Medical/MedicalQueryOrchestrator.js` - Depends on all above
5. `Services/Medical/index.js` - Aggregates exports
6. `controllers/medical.controller.js` - Uses orchestrator
7. `routes/medical.routes.js` - Defines endpoints
8. Update `routes/index.js` - Register routes

## Error Handling

```javascript
class MedicalServiceError extends Error {
    constructor(message, code, details = {}) {
        super(message);
        this.code = code;
        this.details = details;
    }
}

// Error codes
const ERROR_CODES = {
    NER_FAILED: 'NER_EXTRACTION_FAILED',
    RXNORM_UNAVAILABLE: 'RXNORM_SERVICE_UNAVAILABLE',
    MEDLINEPLUS_UNAVAILABLE: 'MEDLINEPLUS_SERVICE_UNAVAILABLE',
    CACHE_ERROR: 'CACHE_SERVICE_ERROR',
    INVALID_QUERY: 'INVALID_QUERY'
};
```

## Testing Strategy

### Unit Tests
- MedicalNERService.extractEntities() with various inputs
- MedlinePlusWebSearchService.search() response parsing
- MedicalCacheService get/set operations
- MedicalQueryOrchestrator flow logic

### Integration Tests
- Full flow: query → NER → search → response
- Cache hit/miss scenarios
- API timeout handling
- Fallback behavior

### Manual Testing Checklist
```bash
# Drug query
curl "http://localhost:3978/api/medical/search?query=What%20are%20the%20side%20effects%20of%20metformin"

# Condition query
curl "http://localhost:3978/api/medical/search?query=What%20is%20diabetes"

# NER extraction
curl "http://localhost:3978/api/medical/ner?text=I%20take%20metformin%20for%20diabetes"

# Topic search
curl "http://localhost:3978/api/medical/topics?q=asthma"

# Health check
curl "http://localhost:3978/api/medical/health"
```

- [ ] Drug query: "What are the side effects of metformin?"
- [ ] Condition query: "What is diabetes?"
- [ ] Symptom query: "Why do my eyes feel dry?"
- [ ] Vague query: "What is a stent?"
- [ ] Multi-entity query: "I take metformin for diabetes"
- [ ] Misspelled drug: "What is tylenall?"
- [ ] Spanish query (if enabled)

## Environment Variables

```bash
# Optional - Azure Text Analytics for Health (for advanced NER)
AZURE_LANGUAGE_KEY=<your-key>
AZURE_LANGUAGE_ENDPOINT=<your-endpoint>

# Optional - Redis (for caching)
REDIS_URL=redis://localhost:6379

# Timeouts
MEDLINEPLUS_TIMEOUT_MS=15000  # (existing)
RXNORM_TIMEOUT_MS=10000       # (existing)
NER_TIMEOUT_MS=5000           # (new)
```

## Risk Mitigation

| Risk | Impact | Mitigation |
|------|--------|------------|
| MedlinePlus API down | High | Graceful degradation, return cached results |
| NER extraction fails | Medium | Fallback to keyword-based extraction |
| Slow response times | Medium | Aggressive caching, parallel requests |
| Rate limiting | Low | Request throttling, respect 1 req/sec |

## Checklist

### Implementation
- [ ] Create Medical directory structure
- [ ] Implement MedicalCacheService
- [ ] Implement MedlinePlusWebSearchService
- [ ] Implement MedicalNERService
- [ ] Implement MedicalQueryOrchestrator
- [ ] Create medical.controller.js
- [ ] Create medical.routes.js
- [ ] Register routes in index.js
- [ ] Add environment variables to config

### Testing
- [ ] Unit tests pass
- [ ] Integration tests pass
- [ ] Manual testing complete
- [ ] Performance acceptable (<5s)

### Documentation
- [ ] API documentation complete
- [ ] Implementation summary written
- [ ] Environment variables documented
