# Review Rules

Review implementation against the repository source of truth.

Inspect:

1. SPEC.md
2. ARCHITECTURE.md
3. relevant ADRs
4. relevant task
5. implementation
6. tests

## Review Priorities

Check:

- correctness
- specification compliance
- architectural invariants
- state transitions
- concurrency
- idempotency
- failure handling
- retry behavior
- recovery behavior
- security
- test coverage
- hidden coupling

## Findings

Classify findings as:

- BLOCKER
- HIGH
- MEDIUM
- LOW
- QUESTION

Every finding should explain:

- what is wrong
- why it matters
- supporting repository evidence
- suggested direction when appropriate

Do not modify code during review unless explicitly requested.

Do not treat a proposed solution as a requirement.