# CB-009: Drug Query Pipeline - RxNorm & MedlinePlus Integration

## Ticket Information
- **Ticket**: CB-009-Drug-Query-Pipeline
- **Type**: Feature Enhancement
- **Priority**: Medium
- **Created**: 2024-11-29
- **Depends On**: CB-008-LangFlow-Handler-Refactor

## Overview

Implement a comprehensive drug query pipeline that normalizes user drug queries using the RxNorm API and retrieves consumer-friendly drug information from MedlinePlus. This feature provides accurate, authoritative drug information to users.

## Target Workflow

```
User text → Drug Intent Detection → RxNorm (RXCUI) → MedlinePlus API → Structured output
```

## Current State Analysis

### Existing Components (Already Implemented)
1. **RxNormService.js** - Complete RxNorm API integration
   - `approximateTerm()` - Fuzzy search for drug names
   - `getRxcuiProperties()` - Get drug properties by RXCUI
   - `getRelatedByType()` - Find related concepts (ingredients)
   - `findBestMatch()` - Main entry point for drug normalization

2. **MedlinePlusService.js** - Complete MedlinePlus Connect API integration
   - `getDrugInfo()` - Fetch drug info by RXCUI
   - `getDrugInfoWithFallback()` - Try multiple RXCUIs
   - Response parsing with section extraction

3. **DrugQueryService.js** - Orchestration service
   - `queryDrug()` - Main entry point
   - `queryDrugsBatch()` - Batch queries
   - `queryByRxcui()` - Direct RXCUI query
   - `formatDrugInfoForDisplay()` - Formatted output

4. **Drug Controller & Routes** - REST API endpoints
   - `POST /api/drug/query` - Query by drug name
   - `GET /api/drug/rxcui/:rxcui` - Query by RXCUI
   - `POST /api/drug/batch` - Batch queries
   - `POST /api/drug/formatted` - Formatted output
   - `GET /api/drug/health` - Health check

### Missing Components (To Be Implemented)
1. **Drug Intent Detection** - Lightweight classifier to detect drug-related queries
2. **Integration with DrugSearchFlow** - Connect LangFlow handler to use DrugQueryService

## Functional Requirements

### FR-1: Drug Intent Detection
Implement a lightweight drug intent detector with:
- Common drug name suffix patterns (-pril, -statin, -mab, -olol, -caine, etc.)
- RxNorm confidence scoring for ambiguous terms
- Integration with existing intent detector framework

### FR-2: Enhanced DrugSearchFlow
Update the DrugSearchFlow handler to:
- Use DrugQueryService instead of LangFlow for drug queries
- Return structured drug information
- Support fallback to LangFlow when DrugQueryService fails

### FR-3: Structured Output Format
Standardized drug information response:
```json
{
  "drugName": "Metformin",
  "rxcui": "6809",
  "description": "Metformin is used to treat type 2 diabetes...",
  "uses": ["Improves glycemic control", "Lowers blood sugar levels"],
  "warnings": ["...", "..."],
  "sideEffects": ["...", "..."],
  "dosage": ["...", "..."],
  "precautions": ["...", "..."],
  "links": ["https://medlineplus.gov/..."]
}
```

## Non-Functional Requirements

### NFR-1: API Reliability
- Both RxNorm and MedlinePlus APIs are free, public services
- No API keys required
- Implement retry logic for transient failures
- Timeout configuration (10s RxNorm, 15s MedlinePlus)

### NFR-2: Performance
- Target response time: < 3 seconds for single drug query
- Parallel API calls where possible
- Caching consideration for frequent queries (future enhancement)

### NFR-3: Accuracy
- Use ingredient-level RXCUI for better MedlinePlus results
- Support fuzzy matching for misspelled drug names
- Provide alternative matches when exact match not found

## API Documentation

### RxNorm REST API (NLM)
- **Base URL**: `https://rxnav.nlm.nih.gov/REST`
- **Documentation**: https://lhncbc.nlm.nih.gov/RxNav/APIs/
- **Authentication**: None required (public API)

#### Key Endpoints
| Endpoint | Method | Description |
|----------|--------|-------------|
| `/approximateTerm.json?term={term}` | GET | Fuzzy search for drug names |
| `/rxcui/{rxcui}/properties.json` | GET | Get drug properties |
| `/rxcui/{rxcui}/related.json?tty=IN` | GET | Get related ingredients |

### MedlinePlus Connect API (NLM)
- **Base URL**: `https://connect.medlineplus.gov`
- **Documentation**: https://medlineplus.gov/medlineplus-connect/web-service/
- **Authentication**: None required (public API)

#### Key Endpoint
| Endpoint | Method | Description |
|----------|--------|-------------|
| `/service` | GET | Get drug/condition information |

#### Parameters
| Parameter | Value | Description |
|-----------|-------|-------------|
| `mainSearchCriteria.v.cs` | `2.16.840.1.113883.6.88` | RxNorm OID |
| `mainSearchCriteria.v.c` | `{rxcui}` | RxNorm Concept ID |
| `informationRecipient.languageCode.c` | `en` | Language code |
| `knowledgeResponseType` | `application/json` | Response format |

## Acceptance Criteria

- [ ] Drug intent detector correctly identifies drug-related queries
- [ ] DrugSearchFlow uses DrugQueryService for drug lookups
- [ ] Structured drug information returned to users
- [ ] Proper error handling with user-friendly messages
- [ ] No modifications to CB-008 refactored structure
- [ ] All existing tests continue to pass

## Dependencies

- **RxNorm API**: No dependencies, public API
- **MedlinePlus API**: No dependencies, public API
- **axios**: HTTP client (already installed)
- **Logger**: Existing logging service

## References

- RxNorm API Documentation: https://lhncbc.nlm.nih.gov/RxNav/APIs/
- MedlinePlus Connect: https://medlineplus.gov/medlineplus-connect/web-service/
- CB-008 Implementation: ../CB-008-LangFlow-Handler-Refactor/
