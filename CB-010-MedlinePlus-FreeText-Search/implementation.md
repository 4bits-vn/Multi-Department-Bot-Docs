# CB-009: Implementation Summary

## Overview

Successfully implemented a comprehensive medical information query system that:
1. Extracts medical entities from user questions using Named Entity Recognition (NER)
2. Normalizes drug terms using RxNorm API
3. Falls back to MedlinePlus free-text search for broader medical queries
4. Provides structured, cacheable responses

This implementation does **not** modify any CB-008 LangFlow Handler logic.

## Files Created

### Directory Structure
```
botframework/src/
├── Services/
│   └── Medical/
│       ├── index.js                        (42 lines)  - Module exports
│       ├── MedicalCacheService.js          (320 lines) - Caching layer
│       ├── MedicalNERService.js            (485 lines) - Entity extraction
│       ├── MedlinePlusWebSearchService.js  (420 lines) - Free-text search
│       └── MedicalQueryOrchestrator.js     (465 lines) - Main orchestrator
│
├── controllers/
│   └── medical.controller.js               (230 lines) - API controller
│
└── routes/
    └── medical.routes.js                   (170 lines) - API routes
```

### Updated Files
- `routes/index.js` - Added medical routes registration

### Total: ~2,132 lines across 7 new files

## Module Responsibilities

### MedlinePlusWebSearchService.js
- **Purpose**: Free-text search against MedlinePlus Health Topics
- **API**: `https://wsearch.nlm.nih.gov/ws/query?db=healthTopics&term=<query>`
- **Response Format**: XML (parsed using `fast-xml-parser`)
- **Features**:
  - Search health topics by keyword
  - Parse XML response and convert to JSON
  - Extract structured sections (symptoms, causes, treatment, etc.)
  - Support for English and Spanish databases
  - Retry logic with exponential backoff

> **Important**: The MedlinePlus Web Service API only returns XML format. There is no JSON option. The service uses `fast-xml-parser` to convert XML to JSON.

### MedicalNERService.js
- **Purpose**: Extract medical entities from user text
- **Entity Types**:
  - `drug` - Medication names (detected via brand names + suffix patterns)
  - `condition` - Diseases, disorders (-itis, -osis suffixes)
  - `symptom` - Signs of illness (pain, fever, etc.)
  - `anatomy` - Body parts (heart, lungs, etc.)
  - `procedure` - Medical tests (MRI, blood test, etc.)
  - `measurement` - Medical measurements (blood pressure, A1C, etc.)
- **Implementation**: Keyword/pattern-based (no external dependencies)

### MedicalCacheService.js
- **Purpose**: Cache medical query results for performance
- **Features**:
  - Redis primary (if `REDIS_URL` configured)
  - In-memory LRU fallback (max 1000 entries)
  - TTL management per cache type
- **Cache TTLs**:
  - NER results: 1 hour
  - RxNorm results: 24 hours
  - MedlinePlus searches: 7 days

### MedicalQueryOrchestrator.js
- **Purpose**: Main orchestrator coordinating all services
- **Flow**:
  1. Check cache for existing result
  2. Extract medical entities (NER)
  3. For drugs: Try RxNorm → RXCUI-based MedlinePlus
  4. For non-drugs: Use free-text search
  5. Cache and return results
- **Exports**:
  - `queryMedical()` - Main query function
  - `extractMedicalEntities()` - NER only
  - `isMedicalQuery()` - Check if query is medical
  - `getHealthTopic()` - Get specific topic
  - `formatResultForDisplay()` - Format for chat
  - `getHealthStatus()` - Service health check

## API Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/medical/search` | Main medical query endpoint |
| GET | `/api/medical/ner` | Extract medical entities only |
| GET | `/api/medical/topic/:topic` | Get specific health topic |
| GET | `/api/medical/topics` | Search health topics (free-text) |
| GET | `/api/medical/formatted` | Get formatted display text |
| GET | `/api/medical/health` | Health check for services |

## API Usage Examples

### Main Medical Search
```bash
# Simple query
curl "http://localhost:3978/api/medical/search?query=What%20are%20the%20side%20effects%20of%20metformin"

# With optional parameters
curl "http://localhost:3978/api/medical/search?query=What%20is%20diabetes&language=en&skipCache=false"
```

**Response:**
```json
{
  "success": true,
  "query": "What are the side effects of metformin?",
  "source": "rxcui",
  "result": {
    "title": "Metformin",
    "url": "https://medlineplus.gov/druginfo/meds/a696005.html",
    "summary": "Metformin is used to treat type 2 diabetes...",
    "sections": {
      "sideEffects": ["nausea", "diarrhea", "stomach upset"]
    }
  },
  "drugInfo": {
    "rxcui": "6809",
    "name": "metformin",
    "matchScore": 11
  },
  "entities": [
    { "text": "metformin", "type": "drug", "confidence": 0.95 }
  ],
  "processingTimeMs": 245
}
```

> **Note**: The `matchScore` is a Lucene score from RxNorm (typically 5-15 for good matches), NOT a percentage (0-100). The orchestrator uses a minimum threshold of 5.

### NER Extraction
```bash
curl "http://localhost:3978/api/medical/ner?text=I%20take%20metformin%20for%20diabetes%20and%20have%20headaches"
```

**Response:**
```json
{
  "success": true,
  "terms": [
    { "text": "metformin", "type": "drug", "confidence": 0.95 },
    { "text": "diabetes", "type": "condition", "confidence": 0.9 },
    { "text": "headaches", "type": "symptom", "confidence": 0.85 }
  ],
  "originalQuery": "I take metformin for diabetes and have headaches",
  "extractionMethod": "keyword"
}
```

### Free-Text Topic Search
```bash
curl "http://localhost:3978/api/medical/topics?q=diabetes"
```

### Formatted Medical Info
```bash
curl "http://localhost:3978/api/medical/formatted?query=What%20is%20diabetes"
```

### Health Check
```bash
curl "http://localhost:3978/api/medical/health"
```

## External APIs Used

### MedlinePlus Web Service (FREE - No API Key Required)
- **URL**: `https://wsearch.nlm.nih.gov/ws/query`
- **Parameters**: `db=healthTopics`, `term=<query>`, `retmax=<limit>`
- **Response Format**: XML only (no JSON option)
- **Parser**: `fast-xml-parser` to convert XML to JSON
- **Documentation**: https://medlineplus.gov/about/developers/webservices/

### RxNorm REST API (FREE - No API Key Required)
- **URL**: `https://rxnav.nlm.nih.gov/REST`
- **Already Implemented**: `RxNormService.js`
- **Documentation**: https://lhncbc.nlm.nih.gov/RxNav/APIs/

### MedlinePlus Connect (FREE - No API Key Required)
- **URL**: `https://connect.medlineplus.gov/service`
- **Already Implemented**: `MedlinePlusService.js`
- **Documentation**: https://medlineplus.gov/medlineplus-connect/technical-information/

## Environment Variables

### New (Optional)
```bash
# MedlinePlus Web Search timeout
MEDLINEPLUS_SEARCH_TIMEOUT_MS=15000

# Redis for caching (optional - falls back to in-memory)
REDIS_URL=redis://localhost:6379
```

### Existing (Used by Service)
```bash
# RxNorm timeout (existing)
RXNORM_TIMEOUT_MS=10000

# MedlinePlus Connect timeout (existing)
MEDLINEPLUS_TIMEOUT_MS=15000
```

## Query Flow Diagram

```
User Question: "What are the side effects of metformin?"
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│ Step 1: Check Cache                                     │
│ Key: medical:query:<hash>                               │
│ If found → Return cached result                         │
└─────────────────────────────────────────────────────────┘
         │ (cache miss)
         ▼
┌─────────────────────────────────────────────────────────┐
│ Step 2: Extract Medical Entities (NER)                  │
│ Result: [{ text: "metformin", type: "drug" }]           │
└─────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│ Step 3: Drug detected → Try RxNorm                      │
│ GET https://rxnav.nlm.nih.gov/REST/approximateTerm.json │
│ Result: RXCUI = 6809, matchScore = 100                  │
└─────────────────────────────────────────────────────────┘
         │ (match found)
         ▼
┌─────────────────────────────────────────────────────────┐
│ Step 4: Query MedlinePlus Connect (RXCUI-based)         │
│ GET https://connect.medlineplus.gov/service             │
│ ?mainSearchCriteria.v.c=6809                            │
│ &mainSearchCriteria.v.cs=2.16.840.1.113883.6.88         │
└─────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│ Step 5: Parse & Format Response                         │
│ Extract: title, summary, sections, URL                  │
└─────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│ Step 6: Cache Result                                    │
│ Key: medical:query:<hash>, TTL: 4 hours                 │
└─────────────────────────────────────────────────────────┘
         │
         ▼
   Return Structured Response
```

## Integration with Existing Services

This implementation:
- ✅ Uses existing `RxNormService.js` for drug normalization
- ✅ Uses existing `MedlinePlusService.js` for RXCUI-based queries
- ✅ Uses existing `Logger.js` for consistent logging
- ✅ Does NOT modify CB-008 LangFlow Handler structure
- ✅ Does NOT modify existing drug query endpoints

## Testing Checklist

### Manual Testing
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
- [ ] NER extraction: `/api/medical/ner?text=...`
- [ ] Topic search: `/api/medical/topics?q=asthma`
- [ ] Health check: `/api/medical/health`

### Performance Testing
- [ ] First query response < 5s
- [ ] Cached query response < 500ms
- [ ] Memory cache working (Redis unavailable)

## Known Limitations

1. **NER Accuracy**: Keyword-based NER is fast but may miss edge cases
   - Mitigation: Fallback to free-text search for unknown terms

2. **No Spanish NER**: Entity extraction only works for English
   - Mitigation: Spanish free-text search still works

3. **MedlinePlus Rate Limits**: No official limits, but respect 1 req/sec
   - Mitigation: Aggressive caching (7-day TTL)

4. **MedlinePlus Connect Sections**: The MedlinePlus Connect API returns a summary but not structured side effects, warnings, etc.
   - Mitigation: URL to full article provided for detailed information

## Bug Fixes Applied

### XML Response Handling (Fixed)
**Issue**: MedlinePlus Web Service only returns XML, not JSON.
**Fix**: Added `fast-xml-parser` to parse XML responses and convert to JSON.

### RxNorm Score Threshold (Fixed)
**Issue**: `RXNORM_MIN_SCORE = 70` was too high. RxNorm uses Lucene scoring (5-15 for good matches), not percentages.
**Fix**: Changed threshold to `RXNORM_MIN_SCORE = 5` in `MedicalQueryOrchestrator.js`.

## Future Improvements

1. **LLM-based NER**: Use LangFlow for more accurate entity extraction
2. **Azure Text Analytics for Health**: Advanced medical NER with UMLS linking
3. **Multi-language NER**: Extend patterns for Spanish terms
4. **Response Enhancement**: Use LLM to summarize MedlinePlus content

## References

- [MedlinePlus Web Services](https://medlineplus.gov/about/developers/webservices/)
- [MedlinePlus Connect](https://medlineplus.gov/medlineplus-connect/technical-information/)
- [RxNorm REST API](https://lhncbc.nlm.nih.gov/RxNav/APIs/)
- [CB-007 Drug Query Flow](../CB-007-Drug-Query-Flow/)
- [CB-008 LangFlow Handler Refactor](../CB-008-LangFlow-Handler-Refactor/)
