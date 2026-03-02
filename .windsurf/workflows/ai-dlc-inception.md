---
description: AI-DLC Inception phase — decompose business intent into validated Units, Stories, and domain models
---

# AI-DLC Inception Phase Workflow

Step-by-step workflow for the **Inception phase** where AI decomposes business intent
into actionable, validated development artifacts.

---

## Overview

```
Business Intent
      │
      ▼
AI decomposes into Units & Stories
      │
      ▼
Mob Elaboration (team validates)
      │
      ▼
Refined Requirements + Domain Model
      │
      ▼
✅ Gate: Team approves → proceed to Construction
```

---

## Steps

### 1. Capture the business intent

Create the intent document — the starting point for all AI decomposition.

```bash
mkdir -p .ai-dlc
```

```markdown
<!-- .ai-dlc/intent.md -->
# Business Intent

## Goal
[What does the business want to achieve?]

## Users
[Who are the target users? What are their personas?]

## Constraints
- Timeline: [deadline or urgency]
- Budget: [resource constraints]
- Technical: [existing systems, required integrations]
- Compliance: [regulatory requirements]

## Success Criteria
- [ ] [Measurable outcome 1]
- [ ] [Measurable outcome 2]
```

### 2. AI decomposes intent into Units of Work

Ask the AI to break down the intent into self-contained **Units** (analogous to bounded contexts).

```markdown
<!-- Prompt pattern -->
Based on the business intent in `.ai-dlc/intent.md`:

1. Decompose this into loosely-coupled Units of Work
2. For each Unit, define:
   - Name and description
   - User Stories with acceptance criteria
   - Dependencies on other Units
   - Estimated complexity (S/M/L)
3. Save each Unit as `.ai-dlc/units/unit-XXX.md`

Ask me clarifying questions before finalizing.
```

**Unit template:**
```markdown
<!-- .ai-dlc/units/unit-001.md -->
# Unit: [Name]

## Description
[What this unit accomplishes]

## Dependencies
- Depends on: [other units, if any]
- Depended by: [units that need this]

## User Stories

### Story 1: [Title]
**As a** [persona], **I want** [action], **so that** [value].

**Acceptance Criteria:**
- [ ] Given [context], when [action], then [result]
- [ ] Given [context], when [action], then [result]

### Story 2: [Title]
...

## Domain Entities
- [Entity 1]: [description]
- [Entity 2]: [description]

## Complexity: [S/M/L]
```

### 3. Conduct Mob Elaboration

This is a **collaborative team ritual** where the AI's proposals are validated.

**Participants:** Product Owner, Tech Lead, Developers, QA

**Process:**
1. AI presents the decomposed Units and Stories
2. Team reviews each Unit for:
   - [ ] Completeness — are all requirements captured?
   - [ ] Correctness — does this match the business intent?
   - [ ] Priority — is the ordering right?
   - [ ] Scope — is each Unit appropriately sized?
3. Team provides feedback — AI refines in real time
4. Repeat until team approves

**AI should ask during Mob Elaboration:**
- "I've decomposed the intent into N Units. Does this grouping make sense?"
- "Story X assumes [assumption]. Is this correct?"
- "I see a dependency between Unit A and Unit B. Should they be merged?"
- "Are there any non-functional requirements I'm missing?"

### 4. Generate the domain model

After Units are validated, AI creates the domain model.

```markdown
<!-- .ai-dlc/domain-model.md -->
# Domain Model

## Entities

### [Entity Name]
- **Properties:** [list]
- **Behaviors:** [list]
- **Invariants:** [business rules that must always hold]

## Relationships
- [Entity A] --[relationship]--> [Entity B]

## Domain Events
- [EventName]: triggered when [condition]

## Aggregates
- [Aggregate Root]: contains [entities]
```

### 5. Record Architecture Decision Records (ADRs)

Every significant decision made during Inception must be recorded.

```bash
mkdir -p .ai-dlc/architecture/decisions
```

```markdown
<!-- .ai-dlc/architecture/decisions/adr-001.md -->
# ADR-001: [Decision Title]

## Status
Accepted | Proposed | Deprecated

## Context
[Why was this decision needed?]

## Decision
[What was decided?]

## Alternatives Considered
1. [Alternative A] — [pros/cons]
2. [Alternative B] — [pros/cons]

## Consequences
- [Positive consequence]
- [Negative consequence or trade-off]
```

### 6. Gate: approve transition to Construction

**Before moving to the Construction phase, verify:**

- [ ] All Units documented in `.ai-dlc/units/`
- [ ] All Stories have acceptance criteria
- [ ] Domain model documented in `.ai-dlc/domain-model.md`
- [ ] Key ADRs recorded
- [ ] Team has approved via Mob Elaboration
- [ ] Dependencies between Units identified
- [ ] Priority order agreed

```bash
# Verify artifacts exist
ls -la .ai-dlc/intent.md
ls -la .ai-dlc/units/
ls -la .ai-dlc/domain-model.md
ls -la .ai-dlc/architecture/decisions/
```

**Only proceed to Construction after explicit team approval.**
