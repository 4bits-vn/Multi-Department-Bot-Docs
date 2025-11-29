# CB-008: Implementation Plan

## Overview

This plan breaks down `LangFlowHandler.js` (2127 lines) into smaller, maintainable modules following the Single Responsibility Principle.

## Proposed Directory Structure

```
botframework/src/
├── Handlers/
│   └── LangFlow/
│       ├── index.js                    # Main orchestrator (~350 lines)
│       ├── config.js                   # Configuration & constants (~80 lines)
│       ├── responseParser.js           # Response parsing utilities (~300 lines)
│       ├── flows/
│       │   ├── index.js                # Flow exports
│       │   ├── BaseFlowHandler.js      # Base class for flows (~100 lines)
│       │   ├── RouterFlow.js           # Router flow handler (~200 lines)
│       │   ├── ExceptionFlow.js        # Exception flow handler (~150 lines)
│       │   ├── ITKBSearchFlow.js       # IT KB search flow (~80 lines)
│       │   ├── TicketingFlow.js        # Ticketing flow (~80 lines)
│       │   ├── ITHelpDeskFlow.js       # IT Help Desk flow (~80 lines)
│       │   ├── MedKBSearchFlow.js      # Medical KB search flow (~80 lines)
│       │   ├── DrugSearchFlow.js       # Drug search flow (~80 lines)
│       │   ├── ChitChatFlow.js         # Chit chat flow (~80 lines)
│       │   └── OtherFlow.js            # Other/fallback flow (~80 lines)
│       └── lifecycleMessageGenerator.js # Lifecycle message generation (~200 lines)
│
├── Handlers/
│   └── LangFlowHandler.js             # DEPRECATED - Re-exports from LangFlow/index.js
```

## Implementation Steps

### Step 1: Create LangFlow Directory Structure
- Create `Handlers/LangFlow/` directory
- Create `Handlers/LangFlow/flows/` subdirectory

### Step 2: Extract Configuration Module (`config.js`)
Extract:
- `LANGFLOW_URL`, `LANGFLOW_API_KEY`, `REQUEST_TIMEOUT`
- `ROUTER_FLOW`, `EXCEPTION_FLOW`, and all flow IDs
- `ROUTING_RESULTS` constant
- `ROUTER_ACTIONS` constant
- `EXCEPTION_ACTIONS` constant

### Step 3: Extract Response Parser Module (`responseParser.js`)
Extract methods:
- `extractOutputText()`
- `stripMarkdownCodeBlock()`
- `parseRouterResponse()`
- `tryParseJSON()`
- `extractJSONFromText()`
- `extractRouterFieldsWithRegex()`
- `parseExceptionResponse()`
- `parseSubFlowResult()`

### Step 4: Create Base Flow Handler (`BaseFlowHandler.js`)
Create a base class with:
- Common HTTP request logic
- Error handling
- Retry logic
- Logging

### Step 5: Extract Individual Flow Handlers
Each flow handler will:
- Extend `BaseFlowHandler`
- Implement flow-specific logic
- Export a singleton instance

#### 5.1: RouterFlow.js
- `sendToRouter()`
- `fallbackToExceptionFlow()`

#### 5.2: ExceptionFlow.js
- `sendToExceptionFlow()`

#### 5.3: ITKBSearchFlow.js
- `callKBSearchFlow()`

#### 5.4: TicketingFlow.js
- `callTicketingFlow()`

#### 5.5: ITHelpDeskFlow.js
- `callITHelpDeskFlow()`

#### 5.6: MedKBSearchFlow.js
- `callMedKBSearchFlow()`

#### 5.7: DrugSearchFlow.js
- `callMedDrugSearchFlow()`

#### 5.8: ChitChatFlow.js
- `callChitChatFlow()`

#### 5.9: OtherFlow.js
- `callOtherFlow()`

### Step 6: Extract Lifecycle Message Generator (`lifecycleMessageGenerator.js`)
Extract methods:
- `generateLifecycleMessage()`
- `generateConversationalMessage()`
- `generateErrorMessage()`

### Step 7: Create Main Orchestrator (`index.js`)
The main `LangFlowHandler` class will:
- Import all flow handlers
- Import response parser
- Import config
- Orchestrate routing via `processRoutingResult()`
- Expose public API

### Step 8: Update Original File for Backward Compatibility
- Replace `LangFlowHandler.js` content with re-exports from new location
- Maintain all existing exports

### Step 9: Update Imports in Consumers
- Verify all consumers still work
- No changes should be required due to backward compatibility

## File Creation Order

1. `config.js` - No dependencies
2. `responseParser.js` - Depends on config.js
3. `BaseFlowHandler.js` - Depends on config.js, responseParser.js
4. Individual flow handlers - Depend on BaseFlowHandler.js
5. `flows/index.js` - Aggregates flow exports
6. `lifecycleMessageGenerator.js` - Depends on flows
7. `index.js` - Main orchestrator

## Risk Mitigation

### Risk 1: Breaking Existing Consumers
**Mitigation**: Keep all exports in original location via re-exports

### Risk 2: Circular Dependencies
**Mitigation**: Clear dependency hierarchy, config at bottom

### Risk 3: Missing Functionality
**Mitigation**: Comprehensive testing after refactoring

## Testing Strategy

1. Ensure dev server starts without errors
2. Test each flow handler individually
3. Test routing through main orchestrator
4. Verify all exports work correctly

## Checklist

- [ ] Create directory structure
- [ ] Extract config.js
- [ ] Extract responseParser.js
- [ ] Create BaseFlowHandler.js
- [ ] Extract RouterFlow.js
- [ ] Extract ExceptionFlow.js
- [ ] Extract ITKBSearchFlow.js
- [ ] Extract TicketingFlow.js
- [ ] Extract ITHelpDeskFlow.js
- [ ] Extract MedKBSearchFlow.js
- [ ] Extract DrugSearchFlow.js
- [ ] Extract ChitChatFlow.js
- [ ] Extract OtherFlow.js
- [ ] Create flows/index.js
- [ ] Extract lifecycleMessageGenerator.js
- [ ] Create main index.js orchestrator
- [ ] Update original LangFlowHandler.js for backward compatibility
- [ ] Test all functionality
- [ ] No linter errors
