# Planning Rules

Planning means defining WHAT must be achieved before deciding HOW it is implemented.

## Before Planning

Read:

- AGENTS.md
- SPEC.md
- ARCHITECTURE.md
- relevant ADRs
- relevant tasks
- relevant source code when planning changes to existing behavior

## Task Decomposition

Break work into small, independently reviewable tasks.

Each task should have:

- Task ID
- Objective
- Context
- Dependencies
- Scope
- Acceptance Criteria
- Test Scenarios
- Documentation Impact

## Architecture Before Implementation

If an architectural decision is unresolved:

1. Identify it.
2. Create a decision/design task if necessary.
3. Mark it as OPEN QUESTION.
4. Do not prescribe an implementation.

Do not prematurely prescribe:

- database schemas
- ORM mappings
- APIs
- messaging systems
- frameworks
- service boundaries
- concurrency mechanisms

unless already established by the specification or an accepted ADR.

## Task Granularity

Avoid tasks combining unrelated concerns.

Split work when a task simultaneously introduces multiple independent concerns such as:

- domain model
- persistence
- messaging
- APIs
- workers
- recovery
- testing

## Planning Output

Planning agents MUST NOT:

- write implementation code
- modify source code
- silently resolve architectural questions
- claim implementation is complete

The result should be an implementation-ready plan, not implementation itself.

## Architecture Decision Boundary

Planning MUST distinguish between:

1. Requirements
2. Architectural decisions
3. Implementation design
4. Implementation tasks

Do not convert an OPEN QUESTION into an implementation decision.

If a task depends on an unresolved architectural question:

- identify the OPEN QUESTION;
- explain why it blocks or affects the task;
- propose an ADR as a decision point when appropriate;
- do not choose the technology, schema, protocol, framework, or implementation pattern on behalf of the project.

A planning task MUST NOT introduce:
- database schemas when storage design is unresolved;
- ORM or framework choices when not decided;
- API contracts when API design is unresolved;
- messaging technology when queue/messaging strategy is unresolved;
- concurrency mechanisms when the consistency model is unresolved.

Architectural decisions must be resolved before implementation tasks depend on them.

## Task Boundary

Separate conceptual/design work from implementation work.

Prefer:

Design/Decision Task
    ↓
Accepted Decision / Approved Design
    ↓
Implementation Task
    ↓
Tests
    ↓
Documentation

Do not combine architectural decision-making and implementation in a single task unless the relevant architecture is already explicitly decided.