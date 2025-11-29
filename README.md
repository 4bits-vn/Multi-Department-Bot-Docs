# ChatBot Platform - Documentation

> **Project**: ChatBot Platform
> **Stack**: Node.js, Bot Framework, LangFlow, Azure Services
> **Last Updated**: November 2024

---

## 📋 Table of Contents

- [Overview](#overview)
- [Quick Start](#quick-start)
- [Architecture](#architecture)
- [Ticket Index](#ticket-index)
- [Documentation Structure](#documentation-structure)
- [Ticket Naming Convention](#ticket-naming-convention)

---

## Overview

This folder contains all technical documentation for the ChatBot Platform. The platform is an enterprise conversational AI system built on Microsoft Bot Framework, integrated with LangFlow for AI-powered intent routing and natural language processing.

### Key Capabilities

| Capability | Description |
|------------|-------------|
| **Multi-Channel Support** | MS Teams (primary), DirectLine (web chat) |
| **AI-Powered Routing** | Intent classification via LangFlow ROUTER |
| **Conversation Loop** | Multi-step interactions (up to 10 iterations) |
| **Knowledge Search** | IT and Medical KB search via vector stores |
| **Ticket Management** | ServiceNow integration for ITSM |
| **Proactive Messaging** | Server-initiated messages to users |
| **Lifecycle Management** | Session timeouts, restarts, error recovery |

---

## Quick Start

1. **Understand the Architecture**: Start with [ARCHITECTURE.md](./ARCHITECTURE.md) for a high-level overview
2. **Review Core Features**: Browse the ticket folders for implementation details
3. **Check the Code**: See `botframework/src/` for the actual implementation

### Key Files

| File | Location | Description |
|------|----------|-------------|
| Main Controller | `botframework/src/controllers/bot.controller.js` | Entry point for all messages |
| LangFlow Handler | `botframework/src/Handlers/LangFlow/index.js` | AI integration orchestrator |
| User Service | `botframework/src/Services/UserService.js` | User & session management |
| State Service | `botframework/src/Services/ConversationStateService.js` | Per-message state machine |
| Lifecycle Service | `botframework/src/Services/ConversationLifecycleService.js` | Conversation lifecycle |

---

## Architecture

The platform follows the **"Asclepius Router" pattern** with a **Conversation Loop**:

```
User Message → Validate User → ROUTER Flow → Sub-Flow(s) → Response
                    ↑                              |
                    └──────── Loop if GO ──────────┘
```

For detailed architecture including data flow, storage design, and component interactions, see:

📖 **[ARCHITECTURE.md](./ARCHITECTURE.md)** - Complete technical architecture documentation

---

## Ticket Index

### ✅ Completed

| Ticket | Title | Type | Description |
|--------|-------|------|-------------|
| [CB-001](./CB-001-Consolidated-Workflow/) | Consolidated Workflow | Enhancement | Core message flow: validation → session → router → sub-flows |
| [CB-002](./CB-002-ConversationState-Refactor/) | ConversationState Refactor | Refactor | Simplified 3-state machine (IDLE → PROCESSING → ERROR) |
| [CB-003](./CB-003-Consolidated-Storage/) | Consolidated Storage | Enhancement | Single Azure Table for all user data |
| [CB-003](./CB-003-Express-Restructure/) | Express Restructure | Chore | Project structure following Express.js best practices |
| [CB-004](./CB-004-Message-Buffering/) | Message Buffering | Enhancement | Multi-input processing with debounce |
| [CB-005](./CB-005-Conversation-Loop/) | Conversation Loop | Enhancement | Multi-step interaction support with GO/COMPLETED actions |
| [CB-006](./CB-006-Conversation-Lifecycle/) | Conversation Lifecycle | Feature | Timeout, restart, error recovery management |
| [CB-007](./CB-007-Drug-Query-Flow/) | Drug Query Flow | Feature | Drug information retrieval via RxNorm & MedlinePlus |
| [CB-007](./CB-007-Dynamic-Lifecycle-Messages/) | Dynamic Lifecycle Messages | Enhancement | LangFlow-generated lifecycle messages |
| [CB-008](./CB-008-LangFlow-Handler-Refactor/) | LangFlow Handler Refactor | Refactor | Modular sub-flow handlers |
| [CB-009](./CB-009-Drug-Query-Pipeline/) | Drug Query Pipeline | Feature | RxNorm + MedlinePlus drug information retrieval |
| [CB-010](./CB-010-MedlinePlus-FreeText-Search/) | MedlinePlus Free-Text Search | Feature | Medical NER + MedlinePlus web search for health topics |

### 📊 Status Legend

| Status | Meaning |
|--------|---------|
| ✅ Completed | Implementation done and tested |
| 🔄 In Progress | Currently being implemented |
| 📋 Planning | Requirements defined, not started |
| 🔮 Proposed | Idea stage, needs requirements |

---

## Documentation Structure

Each ticket folder follows a standard structure:

```
docs/<ticket-number>/
├── requirements.md       # What we're building & acceptance criteria
├── implementation-plan.md # How we'll build it (optional)
├── implementation.md     # What was built & key decisions
└── testing.md           # Test results (if applicable)
```

### Document Descriptions

| Document | Purpose | When Created |
|----------|---------|--------------|
| `requirements.md` | Defines the problem, use cases, functional/non-functional requirements, and acceptance criteria | Before implementation |
| `implementation-plan.md` | Technical approach, file changes, dependencies, risks | Before implementation |
| `implementation.md` | Summary of what was built, files changed, key decisions | After implementation |
| `testing.md` | Test scenarios, results, edge cases tested | After testing |

---

## Ticket Naming Convention

### Format

| Type | Pattern | Example |
|------|---------|---------|
| Feature | `CB-XXX-<short-description>` | `CB-006-Conversation-Lifecycle` |
| Bug | `BUG-XXX-<short-description>` | `BUG-001-Fix-Session-Timeout` |
| Chore | `CHORE-<short-description>` | `CHORE-Update-Dependencies` |
| Integration | `INT-XXX-<short-description>` | `INT-001-ServiceNow-Ticket-Create` |

### Numbering

- Sequential numbering within each type
- Multiple related features may share a number (e.g., CB-007 covers both Drug Query and Dynamic Messages)

---

## Key Concepts

### Conversation Loop

The core pattern for handling multi-step conversations:

1. **ROUTER** classifies user intent and returns `action: GO` or `action: COMPLETED`
2. If `GO`, the appropriate sub-flow is called
3. Sub-flow returns result with `action: GO` (continue) or `action: COMPLETED` (done)
4. If sub-flow returns `GO`, loop back to ROUTER with updated context
5. Maximum 10 iterations (configurable via `MAX_CONVERSATION_ITERATIONS`)

### Conversation Lifecycle

State machine for conversation management:

```
IDLE → ACTIVE → AWAITING_INPUT → COMPLETED
  ↑       ↓           ↓              ↓
  └── ERROR ←── TIMED_OUT ←─────────┘
```

Timeouts:
- Conversation: 30 minutes
- Response: 5 minutes
- Processing: 2 minutes

### LangFlow Routes

| Route | Flow | Description |
|-------|------|-------------|
| `IT_KB_SEARCH` | IT KB Search | IT knowledge base search |
| `IT_TICKET_MGMT` | IT Ticketing | Ticket create/check/update |
| `IT_HELP_DESK` | IT Help Desk | Live agent handover |
| `MED_KB_SEARCH` | Medical KB Search | Medical knowledge search |
| `MED_DRUG_SEARCH` | Drug Search | Drug information lookup |
| `CHIT_CHAT` | Chit Chat | Casual conversation |
| `OTHER` | Other | Fallback handler |

---

## External References

### Official Documentation

| Technology | Documentation |
|------------|---------------|
| Bot Framework | [Azure Bot Service](https://learn.microsoft.com/en-us/azure/bot-service/) |
| MS Teams | [Teams Platform](https://learn.microsoft.com/en-us/microsoftteams/platform/) |
| LangFlow | [LangFlow Docs](https://docs.langflow.org/) |
| Azure Table Storage | [Azure Tables](https://learn.microsoft.com/en-us/azure/cosmos-db/table/) |
| ServiceNow | [ServiceNow API](https://www.servicenow.com/docs/) |

### Internal References

- [Project Structure Guidelines](../.cursor/rules/project-structure.mdc)
- [Coding Standards](../.cursor/rules/coding-standards.mdc)
- [Bot Framework Integration](../.cursor/rules/botframework-teams.mdc)
- [LangFlow Integration](../.cursor/rules/langflow-integration.mdc)

---

## Contributing

When creating new documentation:

1. **Create ticket folder**: `mkdir -p docs/<ticket-number>`
2. **Start with requirements**: Define the problem and acceptance criteria
3. **Create implementation plan**: Detail the technical approach
4. **Document implementation**: Record what was built and key decisions
5. **Follow templates**: Use existing tickets as reference

### Documentation Standards

- Write in **English**
- Use **Markdown format** (.md)
- Include **diagrams** where helpful (ASCII art or Mermaid)
- Link to **official documentation** for external dependencies
- Keep documents **concise and scannable**

---

<div align="center">

**Technology** | ChatBot Platform

</div>
