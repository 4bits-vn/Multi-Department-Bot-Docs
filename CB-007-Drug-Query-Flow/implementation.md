# CB-007: Drug Query Flow - Implementation

## Summary

Implemented a production-ready Drug Query Flow that normalizes drug names using RxNorm API and retrieves comprehensive drug information from MedlinePlus. The implementation follows the existing codebase patterns and integrates seamlessly with the botframework service.

## Files Created

### Services

| File | Description |
|------|-------------|
| `src/Services/RxNormService.js` | RxNorm API integration for drug name normalization |
| `src/Services/MedlinePlusService.js` | MedlinePlus Connect API for drug information |
| `src/Services/DrugQueryService.js` | Main orchestration service combining both APIs |

### Controller & Routes

| File | Description |
|------|-------------|
| `src/controllers/drug.controller.js` | HTTP request handlers for drug queries |
| `src/routes/drug.routes.js` | API route definitions |

### Modified Files

| File | Change |
|------|--------|
| `src/routes/index.js` | Added drug routes registration |
| `src/controllers/index.js` | Added drug controller export |

## API Endpoints

### 1. Query Drug by Name

```
POST /api/drug/query
```

**Request:**
```json
{
  "drugName": "metformin",
  "maxAlternatives": 3,
  "language": "en"
}
```

**Response:**
```json
{
  "success": true,
  "query": "metformin",
  "drug": {
    "rxcui": "6809",
    "name": "metformin",
    "ingredientName": "metformin",
    "matchScore": 100,
    "termType": "IN",
    "medlinePlus": {
      "title": "Metformin",
      "url": "https://medlineplus.gov/druginfo/meds/a696005.html",
      "summary": "...",
      "uses": ["..."],
      "warnings": ["..."],
      "sideEffects": ["..."]
    }
  },
  "alternativeMatches": [
    { "rxcui": "860975", "name": "Metformin 500 MG", "score": 92 }
  ]
}
```

### 2. Query by RXCUI

```
GET /api/drug/rxcui/:rxcui
```

**Example:** `GET /api/drug/rxcui/6809`

### 3. Batch Query

```
POST /api/drug/batch
```

**Request:**
```json
{
  "drugNames": ["aspirin", "ibuprofen", "metformin"]
}
```

### 4. Formatted Output

```
POST /api/drug/formatted
```

Returns human-readable markdown-formatted drug information.

### 5. Health Check

```
GET /api/drug/health
```

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Client Request                           │
│                   POST /api/drug/query                          │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                     drug.controller.js                          │
│              - Request validation                                │
│              - Error handling                                    │
│              - Response formatting                               │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                     DrugQueryService.js                         │
│              - Orchestrates the query flow                       │
│              - Combines RxNorm + MedlinePlus data               │
│              - Formats results                                   │
└─────────────────────────────────────────────────────────────────┘
                              │
              ┌───────────────┴───────────────┐
              ▼                               ▼
┌─────────────────────────┐     ┌─────────────────────────┐
│    RxNormService.js     │     │ MedlinePlusService.js   │
│  - approximateTerm      │     │   - getDrugInfo         │
│  - getRxcuiProperties   │     │   - parseResponse       │
│  - getRelatedByType     │     │   - cleanHtml           │
│  - findBestMatch        │     │                         │
└─────────────────────────┘     └─────────────────────────┘
              │                               │
              ▼                               ▼
┌─────────────────────────┐     ┌─────────────────────────┐
│    RxNorm REST API      │     │  MedlinePlus Connect    │
│ rxnav.nlm.nih.gov       │     │ connect.medlineplus.gov │
│    (No API Key)         │     │    (No API Key)         │
└─────────────────────────┘     └─────────────────────────┘
```

## Key Implementation Details

### RxNorm Integration

1. **approximateTerm Endpoint**: Best for fuzzy matching user input
   - Handles misspellings (e.g., "metphormin" → "metformin")
   - Handles abbreviations and brand names
   - Returns candidates with match scores

2. **Properties Lookup**: Gets canonical drug names and term types
   - SCD = Semantic Clinical Drug
   - SBD = Semantic Branded Drug
   - IN = Ingredient (best for MedlinePlus queries)

3. **Related Concepts**: Finds ingredient-level RXCUI from product-level
   - Improves MedlinePlus match rate

### MedlinePlus Integration

1. **Connect API**: Uses HL7 Infobutton standard
   - Requires RXCUI + RxNorm OID (2.16.840.1.113883.6.88)
   - Returns consumer-friendly drug information

2. **Response Parsing**: Extracts structured content
   - Title and URL
   - Summary text
   - Uses, warnings, side effects (when available)

3. **Fallback Strategy**: Tries multiple RXCUIs
   - Ingredient-level RXCUI first
   - Product-level RXCUI as fallback

### Error Handling

- **Retry Logic**: 2 retries with exponential backoff
- **Timeout Handling**: 10s for RxNorm, 15s for MedlinePlus
- **Graceful Degradation**: Returns partial data on failures
- **Input Validation**: Length limits, format checks

## No API Keys Required

Both RxNorm and MedlinePlus are **public APIs** provided by the National Library of Medicine. No API keys or authentication is required.

| Service | Base URL | Auth Required |
|---------|----------|---------------|
| RxNorm | `https://rxnav.nlm.nih.gov/REST` | No |
| MedlinePlus | `https://connect.medlineplus.gov` | No |

## Configuration Options

Optional environment variables:

```bash
# RxNorm timeout (default: 10000ms)
RXNORM_TIMEOUT_MS=10000

# MedlinePlus timeout (default: 15000ms)
MEDLINEPLUS_TIMEOUT_MS=15000
```

## Testing

### Manual Testing

```bash
# Start the server
cd botframework
pnpm dev

# Test drug query
curl -X POST http://localhost:3000/api/drug/query \
  -H "Content-Type: application/json" \
  -d '{"drugName": "metformin"}'

# Test with misspelling
curl -X POST http://localhost:3000/api/drug/query \
  -H "Content-Type: application/json" \
  -d '{"drugName": "metphormin"}'

# Test by RXCUI
curl http://localhost:3000/api/drug/rxcui/6809

# Test health check
curl http://localhost:3000/api/drug/health
```

### Test Cases Verified

| Test Case | Input | Expected Result |
|-----------|-------|-----------------|
| Exact match | "aspirin" | Returns aspirin with MedlinePlus info |
| Misspelling | "metphormin" | Returns metformin |
| Brand name | "Tylenol" | Returns acetaminophen |
| With dosage | "aspirin 325mg" | Returns aspirin (ignores dosage) |
| Not found | "xyzabc123" | Returns null drug with message |

## Known Limitations

1. **MedlinePlus Coverage**: Not all drugs have MedlinePlus articles
   - Newer drugs may not be available
   - Some formulations return no data

2. **Language Support**: Currently supports English (en) and Spanish (es)
   - Other languages may have limited content

3. **Rate Limits**: No explicit rate limits, but excessive requests may be throttled
   - Batch API limited to 10 drugs per request

## Future Enhancements

1. **Caching**: Redis cache for frequently queried drugs
2. **Drug Interactions**: Use RxNorm interaction API
3. **Image Support**: Drug images from RxImage API
4. **NDC Lookup**: National Drug Code support

## References

- [RxNorm API Documentation](https://lhncbc.nlm.nih.gov/RxNav/APIs/)
- [MedlinePlus Connect](https://medlineplus.gov/connect/service.html)
- [RxNorm Technical Documentation](https://www.nlm.nih.gov/research/umls/rxnorm/docs/index.html)
