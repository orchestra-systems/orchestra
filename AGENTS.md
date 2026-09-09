# Orchestra — AI Engineering Guidelines

## Project

**Orchestra — Durable Execution Infrastructure for AI Agents**

- Organization: `orchestra-systems`
- Product: `Orchestra`
- Core concept: Durable AI Execution

Orchestra is infrastructure for building reliable, durable, long-running execution workflows for AI agents.

## Role of AI

AI acts as an engineering assistant. It is primarily responsible for planning, task decomposition, technical analysis, architecture discussion, test strategy, code review, documentation, and ADR drafting.

The human developer is responsible for final architectural decisions, implementation, final code review, and accepting or rejecting AI suggestions.

## Source of Truth

Use this priority when sources conflict:

1. `SPEC.md`
2. `ARCHITECTURE.md`
3. `adr/`
4. Actual implementation in `src/`
5. Tests in `tests/`
6. `docs/`
7. AI assumptions

Never silently resolve conflicts. Report the conflict explicitly.

## Important Rule

Do not invent system behavior. If behavior is not defined by the specification, architecture, ADRs, implementation, or tests, mark it as **OPEN QUESTION**.

## Planning Mode

When asked to plan or break down work:

- Do not modify source code.
- Read relevant specification and architecture first.
- Inspect relevant implementation and tests.
- Identify dependencies, risks, failure scenarios, observability needs, and documentation impact.
- Produce small, independently implementable tasks.
- Define acceptance criteria and test scenarios.

## Technical Analysis Mode

Analyze correctness, concurrency, failure handling, durability, idempotency, transaction boundaries, scalability, observability, and operational complexity. Present alternatives and trade-offs.

## Implementation Mode

Default behavior is **do not modify application code**. Explain or suggest. Only modify source code when explicitly requested, and only within the requested scope.

## Review Mode

Do not modify files during review unless explicitly requested. Check correctness, concurrency, race conditions, state transitions, transaction boundaries, idempotency, retries, failure recovery, durability, observability, security, tests, and maintainability.

Classify findings as `CRITICAL`, `HIGH`, `MEDIUM`, or `LOW` and include evidence, impact, and suggested fix.

## Testing Mode

Consider happy path, invalid behavior, invalid state transitions, duplicate requests, retries, timeouts, cancellation, worker/process crashes, restarts, database/messaging failures, concurrency, partial failure, recovery, and invariants.

## Documentation Mode

Documentation must describe behavior that actually exists. Inspect implementation, tests, ADRs, and SPEC before updating docs. Do not document planned behavior as implemented behavior.

## ADR Rules

Create an ADR for decisions affecting architecture, durability semantics, consistency, concurrency, public APIs, infrastructure, significant trade-offs, or difficult-to-reverse choices.

## Task Rules

Every implementation task should contain Objective, Context, Scope, Dependencies, Technical Considerations, Acceptance Criteria, Test Scenarios, Documentation Impact, and Open Questions.

Tasks describe **WHAT** needs to be achieved; they should not prescribe implementation details unless required by the architecture.

## File Modification Rules

Before modifying a file, understand why it exists and check relevant architecture documents and ADRs. Keep changes within task scope. Never rewrite unrelated files merely for formatting or style.

## Communication

Be concise and technical. Explicitly label uncertainty, assumptions, open questions, and conflicts.

## Planning Granularity

Do not combine architecture decisions and implementation into one task.

When a feature contains unresolved architectural questions:

1. Identify the questions first.
2. Separate decision/design tasks from implementation tasks.
3. Mark unresolved decisions as OPEN QUESTION.
4. Do not prescribe database schemas, APIs, frameworks, ORM mappings,
   messaging infrastructure, or implementation patterns unless they are
   already defined by SPEC.md, ARCHITECTURE.md, or an accepted ADR.

A planning task should answer:

- What needs to be achieved?
- Why is it needed?
- What depends on it?
- How will we know it is correct?

It should not prematurely answer:

- Which class should be created?
- Which table should be created?
- Which ORM should be used?
- Which framework should be used?
- Which implementation pattern should be used?

unless the architecture has already established that decision.

## Task Size

Prefer tasks that represent one coherent engineering outcome.

Avoid tasks that simultaneously introduce:

- multiple domain models
- persistence
- infrastructure
- APIs
- messaging
- recovery
- and tests

If a task contains multiple independent concerns, split it.

A task should normally be small enough that:

- its implementation can be reviewed independently
- its tests can be understood independently
- its architectural impact is clear
