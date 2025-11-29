# CB-008: LangFlow Handler Refactoring

## Ticket Information
- **Ticket**: CB-008-LangFlow-Handler-Refactor
- **Type**: Refactoring / Technical Debt
- **Priority**: Medium
- **Created**: 2024-11-28

## Overview

The `LangFlowHandler.js` file has grown to 2127 lines and contains multiple responsibilities that should be separated into smaller, more maintainable modules. This refactoring will improve code organization, testability, and maintainability without changing functionality.

## Current State Analysis

### File: `botframework/src/Handlers/LangFlowHandler.js`
- **Lines**: 2127
- **Responsibilities**:
  1. Configuration and constants management
  2. LangFlow API communication
  3. Response parsing and extraction
  4. Router flow orchestration
  5. Sub-flow handlers (KB Search, Ticketing, Help Desk, Medical, Drug Search, Chit Chat, Other)
  6. Lifecycle message generation
  7. Error handling and retry logic

### Problems with Current Structure
1. **Single Responsibility Violation**: One file handles multiple concerns
2. **Hard to Test**: Large file makes unit testing difficult
3. **Hard to Navigate**: 2000+ lines makes finding code difficult
4. **Tight Coupling**: All flow handlers in one class
5. **Code Duplication**: Similar patterns across flow handlers

## Functional Requirements

### FR-1: Separate Configuration Module
Extract all configuration constants to a dedicated module:
- Flow IDs
- Routing results constants
- Router actions
- Exception actions
- API configuration (URLs, timeouts)

### FR-2: Separate Response Parser Module
Extract all response parsing logic:
- `extractOutputText()`
- `stripMarkdownCodeBlock()`
- `parseRouterResponse()`
- `tryParseJSON()`
- `extractJSONFromText()`
- `extractRouterFieldsWithRegex()`
- `parseExceptionResponse()`
- `parseSubFlowResult()`

### FR-3: Separate Sub-Flow Handlers
Extract individual flow handlers into separate modules:
- IT KB Search Flow
- Ticketing Flow
- IT Help Desk Flow
- Medical KB Search Flow
- Drug Search Flow
- Chit Chat Flow
- Other Flow

### FR-4: Separate Lifecycle Message Service
Already partially extracted to `LifecycleMessageService.js`, ensure proper integration.

### FR-5: Refactored Main Handler
Keep the main `LangFlowHandler.js` as an orchestrator that:
- Initializes all services
- Routes to appropriate sub-handlers
- Maintains backward compatibility with existing API

## Non-Functional Requirements

### NFR-1: Backward Compatibility
- All existing exports must continue to work
- No changes to external API signatures
- Existing consumers should not require modifications

### NFR-2: Performance
- No degradation in response times
- Lazy loading of sub-handlers where appropriate

### NFR-3: Testability
- Each module should be independently testable
- Clear interfaces between modules

### NFR-4: Maintainability
- Each file should be under 300 lines
- Clear separation of concerns
- Consistent naming conventions

## Acceptance Criteria

- [ ] `LangFlowHandler.js` reduced to <400 lines (orchestration only)
- [ ] All sub-flows extracted to separate modules
- [ ] Response parsing extracted to dedicated module
- [ ] Configuration extracted to dedicated module
- [ ] All existing functionality works unchanged
- [ ] All exports maintained for backward compatibility
- [ ] No linter errors or warnings

## Dependencies

- Node.js
- axios
- Existing Logger service
- Existing utils/lifecycleMessages module

## References

- LangFlow API Documentation: https://docs.langflow.org/api-reference-api-examples
- Project Structure Guidelines: `.cursor/rules/project-structure.mdc`
- Coding Standards: `.cursor/rules/coding-standards.mdc`
