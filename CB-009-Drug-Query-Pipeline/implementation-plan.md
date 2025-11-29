# CB-009: Implementation Plan

## Architecture Overview

### End-to-End Drug Query Pipeline

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         DRUG QUERY PIPELINE                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌──────────────┐                                                           │
│  │  User Input  │  "What is metformin used for?"                            │
│  └──────┬───────┘                                                           │
│         │                                                                   │
│         ▼                                                                   │
│  ┌──────────────────────────────────────┐                                   │
│  │   Step 1: Drug Intent Detection      │                                   │
│  │   ─────────────────────────────────  │                                   │
│  │   • Check drug suffix patterns       │                                   │
│  │   • Check common drug names          │                                   │
│  │   • RxNorm confidence scoring        │                                   │
│  │   • Return: { isDrug, confidence }   │                                   │
│  └──────────────┬───────────────────────┘                                   │
│                 │                                                           │
│                 ▼                                                           │
│  ┌──────────────────────────────────────┐                                   │
│  │   Step 2: Drug Name Normalization    │ ◄── RxNorm REST API               │
│  │   ─────────────────────────────────  │     https://rxnav.nlm.nih.gov     │
│  │   • approximateTerm (fuzzy match)    │                                   │
│  │   • Get best RXCUI candidate         │                                   │
│  │   • Resolve to ingredient RXCUI      │                                   │
│  │   • Return: { rxcui, name, score }   │                                   │
│  └──────────────┬───────────────────────┘                                   │
│                 │                                                           │
│                 ▼                                                           │
│  ┌──────────────────────────────────────┐                                   │
│  │   Step 3: Drug Info Retrieval        │ ◄── MedlinePlus Connect API       │
│  │   ─────────────────────────────────  │     https://connect.medlineplus   │
│  │   • Query with RXCUI + RxNorm OID    │     .gov                          │
│  │   • Parse JSON response              │                                   │
│  │   • Extract structured sections      │                                   │
│  │   • Return: { title, uses, ... }     │                                   │
│  └──────────────┬───────────────────────┘                                   │
│                 │                                                           │
│                 ▼                                                           │
│  ┌──────────────────────────────────────┐                                   │
│  │   Step 4: Response Formatting        │                                   │
│  │   ─────────────────────────────────  │                                   │
│  │   • Combine RxNorm + MedlinePlus     │                                   │
│  │   • Format for display               │                                   │
│  │   • Add disclaimer                   │                                   │
│  │   • Return to user                   │                                   │
│  └──────────────────────────────────────┘                                   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Component Architecture

```
botframework/src/
├── Services/
│   ├── RxNormService.js          ✅ EXISTING - RxNorm API integration
│   ├── MedlinePlusService.js     ✅ EXISTING - MedlinePlus API integration
│   └── DrugQueryService.js       ✅ EXISTING - Orchestration service
│
├── utils/
│   └── intentDetectors.js        🔄 ENHANCE - Add drug intent detection
│
├── controllers/
│   └── drug.controller.js        ✅ EXISTING - HTTP API endpoints
│
├── routes/
│   └── drug.routes.js            ✅ EXISTING - Route definitions
│
└── Handlers/
    └── LangFlow/
        └── flows/
            └── DrugSearchFlow.js  🔄 ENHANCE - Use DrugQueryService
```

## Implementation Steps

### Step 1: Add Drug Intent Detector (intentDetectors.js)

Add a lightweight drug intent detector with:

```javascript
// Drug suffix patterns (pharmaceutical naming conventions)
const DRUG_SUFFIXES = [
    '-pril',      // ACE inhibitors (lisinopril, enalapril)
    '-statin',    // Statins (atorvastatin, simvastatin)
    '-mab',       // Monoclonal antibodies (adalimumab, infliximab)
    '-olol',      // Beta blockers (metoprolol, atenolol)
    '-caine',     // Local anesthetics (lidocaine, novocaine)
    '-azole',     // Antifungals (fluconazole, ketoconazole)
    '-cillin',    // Penicillins (amoxicillin, ampicillin)
    '-mycin',     // Antibiotics (azithromycin, erythromycin)
    '-prazole',   // PPIs (omeprazole, esomeprazole)
    '-sartan',    // ARBs (losartan, valsartan)
    '-dipine',    // Calcium channel blockers (amlodipine)
    '-pine',      // Antipsychotics (quetiapine, olanzapine)
    '-ine',       // Various (morphine, aspirin is special case)
    '-ide',       // Various (furosemide, glipizide)
    '-formin',    // Biguanides (metformin)
    '-artan',     // ARBs (alternative suffix)
    '-conazole',  // Antifungals
    '-cycline',   // Tetracyclines
    '-floxacin',  // Fluoroquinolones
];

// Common drug names (brand and generic)
const COMMON_DRUG_NAMES = [
    'aspirin', 'tylenol', 'advil', 'ibuprofen', 'acetaminophen',
    'metformin', 'lisinopril', 'amlodipine', 'atorvastatin',
    'omeprazole', 'losartan', 'gabapentin', 'hydrochlorothiazide',
    'sertraline', 'montelukast', 'escitalopram', 'rosuvastatin',
    'prednisone', 'levothyroxine', 'pantoprazole', 'tramadol',
    'alprazolam', 'duloxetine', 'carvedilol', 'trazodone',
    // ... more common drugs
];

// Drug-related query patterns
const DRUG_QUERY_PATTERNS = [
    /what\s+is\s+(\w+)\s+(used\s+for|for)/i,
    /tell\s+me\s+about\s+(\w+)/i,
    /information\s+(on|about)\s+(\w+)/i,
    /side\s+effects\s+of\s+(\w+)/i,
    /how\s+does\s+(\w+)\s+work/i,
    /dosage\s+(of|for)\s+(\w+)/i,
    /interactions\s+(with|of)\s+(\w+)/i,
];
```

**Functions to add:**
- `detectDrugIntent(text)` - Returns `{ isDrug, confidence, extractedDrug }`
- `extractDrugName(text)` - Extracts potential drug name from query
- `hasDrugSuffix(word)` - Checks if word has pharmaceutical suffix
- `isCommonDrugName(word)` - Checks against known drug list

### Step 2: Enhance DrugSearchFlow (DrugSearchFlow.js)

Update to use DrugQueryService:

```javascript
class DrugSearchFlow extends BaseFlowHandler {
    async execute(params) {
        const { query, sessionId, userEmail } = params;

        // Use DrugQueryService for drug lookups
        const drugResult = await DrugQueryService.queryDrug(query);

        if (drugResult.success && drugResult.drug) {
            // Format the drug information for display
            const formattedText = DrugQueryService.formatDrugInfoForDisplay(drugResult);
            return {
                success: true,
                text: formattedText,
                raw: drugResult,
                source: 'DrugQueryService'
            };
        }

        // Fallback to LangFlow if DrugQueryService fails
        return this.fallbackToLangFlow(params);
    }
}
```

### Step 3: API Endpoint Verification

Verify all endpoints work correctly:

| Endpoint | Test Command |
|----------|-------------|
| GET /api/drug/query | `curl "http://localhost:3978/api/drug/query?drugName=metformin"` |
| GET /api/drug/rxcui/:rxcui | `curl "http://localhost:3978/api/drug/rxcui/6809"` |
| POST /api/drug/batch | `curl -X POST http://localhost:3978/api/drug/batch -H "Content-Type: application/json" -d '{"drugNames":["aspirin","ibuprofen"]}'` |
| GET /api/drug/formatted | `curl "http://localhost:3978/api/drug/formatted?drugName=metformin"` |
| GET /api/drug/health | `curl "http://localhost:3978/api/drug/health"` |

## API Call Details

### RxNorm API Calls

#### 1. Approximate Term Search
```
GET https://rxnav.nlm.nih.gov/REST/approximateTerm.json?term=metformin&maxEntries=5

Response:
{
  "approximateGroup": {
    "candidate": [
      {
        "rxcui": "6809",
        "score": "100",
        "rank": "1",
        "name": "metformin"
      }
    ]
  }
}
```

#### 2. Get RXCUI Properties
```
GET https://rxnav.nlm.nih.gov/REST/rxcui/6809/properties.json

Response:
{
  "properties": {
    "rxcui": "6809",
    "name": "metformin",
    "synonym": "",
    "tty": "IN",
    "language": "ENG"
  }
}
```

#### 3. Get Related Concepts (Ingredients)
```
GET https://rxnav.nlm.nih.gov/REST/rxcui/6809/related.json?tty=IN

Response:
{
  "relatedGroup": {
    "conceptGroup": [
      {
        "tty": "IN",
        "conceptProperties": [
          {
            "rxcui": "6809",
            "name": "metformin"
          }
        ]
      }
    ]
  }
}
```

### MedlinePlus API Call

```
GET https://connect.medlineplus.gov/service?mainSearchCriteria.v.cs=2.16.840.1.113883.6.88&mainSearchCriteria.v.c=6809&informationRecipient.languageCode.c=en&knowledgeResponseType=application/json

Response:
{
  "feed": {
    "title": "MedlinePlus Connect",
    "entry": [
      {
        "title": "Metformin",
        "link": [{"href": "https://medlineplus.gov/druginfo/meds/a696005.html"}],
        "summary": "Metformin is used alone or with other medications..."
      }
    ]
  }
}
```

## Error Handling Strategy

### RxNorm Errors
| Error | Handling |
|-------|----------|
| No matches found | Return user-friendly "drug not found" message with suggestions |
| API timeout | Retry up to 2 times with exponential backoff |
| Network error | Log error, return generic error message |

### MedlinePlus Errors
| Error | Handling |
|-------|----------|
| No drug info available | Return RxNorm data only with note about missing info |
| API timeout | Retry up to 2 times |
| Invalid RXCUI | Try ingredient-level RXCUI as fallback |

## Testing Strategy

### Unit Tests
- [ ] Drug intent detector correctly identifies drug queries
- [ ] Drug suffix matching works correctly
- [ ] Common drug name matching works
- [ ] RxNormService.findBestMatch returns correct results
- [ ] MedlinePlusService.getDrugInfo parses response correctly

### Integration Tests
- [ ] Full pipeline: drug name → RXCUI → drug info
- [ ] API endpoint responses are correct
- [ ] Error handling works as expected

### Manual Testing
```bash
# Test drug query endpoint
curl "http://localhost:3978/api/drug/query?drugName=metformin"

# Test with misspelling
curl "http://localhost:3978/api/drug/query?drugName=metphormin"

# Test formatted drug info
curl "http://localhost:3978/api/drug/formatted?drugName=aspirin"

# Test health check
curl "http://localhost:3978/api/drug/health"
```

## Checklist

- [ ] Create CB-009 documentation folder
- [ ] Create requirements.md
- [ ] Create implementation-plan.md
- [ ] Add drug intent detector to intentDetectors.js
- [ ] Enhance DrugSearchFlow to use DrugQueryService
- [ ] Test all API endpoints
- [ ] Create implementation.md with final documentation
- [ ] No modifications to CB-008 structure

## Risk Mitigation

### Risk 1: API Rate Limiting
- **Risk**: NLM APIs may have rate limits
- **Mitigation**: Implement request throttling and caching

### Risk 2: API Unavailability
- **Risk**: External APIs may be temporarily unavailable
- **Mitigation**: Implement graceful fallback to LangFlow

### Risk 3: Drug Name Ambiguity
- **Risk**: Some drug names may match multiple drugs
- **Mitigation**: Return alternatives and let user select

## References

- RxNorm API: https://lhncbc.nlm.nih.gov/RxNav/APIs/
- MedlinePlus Connect: https://medlineplus.gov/medlineplus-connect/web-service/
- NLM API Terms: https://lhncbc.nlm.nih.gov/RxNav/TermOfService.html
