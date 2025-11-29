# CB-007: Drug Query Flow - Implementation Plan

## Technical Approach

### Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        Client Request                           │
│                   POST /api/drug/query                          │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      drug.controller.js                         │
│              - Request validation                                │
│              - Response formatting                               │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                     DrugQueryService.js                         │
│              - Orchestrates the query flow                       │
│              - Combines RxNorm + MedlinePlus data               │
└─────────────────────────────────────────────────────────────────┘
                              │
              ┌───────────────┴───────────────┐
              ▼                               ▼
┌─────────────────────────┐     ┌─────────────────────────┐
│      RxNormService      │     │   MedlinePlusService    │
│  - approximateTerm      │     │   - getDrugInfo         │
│  - getRxcuiProperty     │     │   - parseResponse       │
│  - findBestMatch        │     │                         │
└─────────────────────────┘     └─────────────────────────┘
              │                               │
              ▼                               ▼
┌─────────────────────────┐     ┌─────────────────────────┐
│    RxNorm REST API      │     │  MedlinePlus Connect    │
│ rxnav.nlm.nih.gov       │     │ connect.medlineplus.gov │
└─────────────────────────┘     └─────────────────────────┘
```

### Data Flow

1. **User Input**: Raw drug name (e.g., "metphormin", "advil", "tylenol 500")
2. **Step 1**: Normalize via RxNorm `approximateTerm` → Get candidate RXCUIs
3. **Step 2**: Select best match based on score/rank
4. **Step 3**: Get canonical name via RxNorm property lookup
5. **Step 4**: Query MedlinePlus with RXCUI
6. **Step 5**: Parse and structure the response
7. **Output**: Clean JSON with drug information

## Implementation Steps

### Step 1: Create RxNormService
- Implement `approximateTerm` endpoint call
- Implement `getRxcuiProperty` for canonical names
- Implement best match selection logic
- Handle no-match scenarios

### Step 2: Create MedlinePlusService
- Implement drug info retrieval by RXCUI
- Parse HL7 C-CDA JSON response
- Extract: title, summary, uses, warnings, side effects, links

### Step 3: Create DrugQueryService
- Orchestrate the complete flow
- Combine data from both services
- Format final response

### Step 4: Create Controller & Routes
- `drug.controller.js` with query handler
- `drug.routes.js` with endpoint definitions
- Register routes in main router

## API Specification

### POST /api/drug/query

**Request Body:**
```json
{
  "drugName": "metformin",
  "maxResults": 3
}
```

**Success Response (200):**
```json
{
  "success": true,
  "query": "metformin",
  "drug": {
    "rxcui": "6809",
    "name": "Metformin",
    "normalizedName": "metformin hydrochloride",
    "matchScore": 100,
    "medlinePlus": {
      "title": "Metformin",
      "url": "https://medlineplus.gov/druginfo/meds/a696005.html",
      "summary": "...",
      "uses": ["..."],
      "warnings": ["..."],
      "sideEffects": ["..."],
      "dosage": "..."
    }
  },
  "alternativeMatches": [
    { "rxcui": "860975", "name": "Metformin 500 MG", "score": 92 }
  ]
}
```

**Not Found Response (200):**
```json
{
  "success": true,
  "query": "xyzabc123",
  "drug": null,
  "message": "No matching drug found for 'xyzabc123'"
}
```

**Error Response (500):**
```json
{
  "success": false,
  "error": "Failed to query drug information",
  "details": "RxNorm API timeout"
}
```

## External API Details

### RxNorm API

**Base URL:** `https://rxnav.nlm.nih.gov/REST`

**Endpoints Used:**

1. **Approximate Term Match**
   ```
   GET /approximateTerm.json?term={drugName}&maxEntries={max}
   ```

2. **Get RXCUI Properties**
   ```
   GET /rxcui/{rxcui}/properties.json
   ```

**Response Parsing:**
- `approximateGroup.candidate[]` - array of matches
- Each candidate has: `rxcui`, `score`, `rank`, `name`

### MedlinePlus Connect API

**Base URL:** `https://connect.medlineplus.gov`

**Endpoint:**
```
GET /service?mainSearchCriteria.v.c={rxcui}&mainSearchCriteria.v.cs=2.16.840.1.113883.6.88&knowledgeResponseType=application/json
```

**Response Structure:**
- `feed.entry[]` - array of matching articles
- Each entry has: `title`, `summary`, `link[]`

## Error Handling Strategy

| Scenario | Handling |
|----------|----------|
| Drug not found in RxNorm | Return success with null drug, helpful message |
| RxNorm API timeout | Retry 2x, then return error |
| MedlinePlus no data | Return drug info from RxNorm only |
| MedlinePlus timeout | Retry 2x, then return partial data |
| Invalid drug name | Return validation error |

## Testing Strategy

### Manual Testing
1. Common drugs: "aspirin", "ibuprofen", "metformin"
2. Misspellings: "metphormin", "advl", "tyelnol"
3. Brand names: "Tylenol", "Advil", "Lipitor"
4. Edge cases: empty string, special characters, very long names

### Test Cases
- [ ] Exact match returns correct drug
- [ ] Misspelling returns correct normalized drug
- [ ] Brand name resolves to generic
- [ ] Non-existent drug returns graceful not-found
- [ ] API timeout triggers retry
- [ ] Malformed input is rejected

## Files to Create/Modify

### New Files
- `src/Services/DrugQueryService.js` - Main orchestration service
- `src/Services/RxNormService.js` - RxNorm API integration
- `src/Services/MedlinePlusService.js` - MedlinePlus API integration
- `src/controllers/drug.controller.js` - Request handling
- `src/routes/drug.routes.js` - Route definitions

### Modified Files
- `src/routes/index.js` - Register drug routes
- `src/controllers/index.js` - Export drug controller
