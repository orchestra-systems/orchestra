# Orchestra Project Rules

## Source of Truth

The repository specification is the highest-level source of truth.

Before making recommendations or changes:

1. Read the relevant specification.
2. Inspect relevant architecture documentation.
3. Inspect accepted ADRs.
4. Inspect the current implementation and tests when relevant.

Never silently modify requirements to justify an implementation.

## Requirement Fidelity

Do not strengthen, weaken, or reinterpret requirements.

Examples:

- "monotonically increasing" does not automatically mean "gapless"
- "ordered" does not automatically mean "globally ordered"
- "durable" does not automatically specify a persistence mechanism

When the source does not define a behavior, mark it:

OPEN QUESTION

Do not invent a decision.

## Architecture

Preserve existing architectural invariants.

When an architectural decision is unresolved:

- identify the decision
- explain relevant trade-offs
- propose options when requested
- mark it as OPEN QUESTION

Do not present a proposal as an accepted decision.

## Verification

Never claim that something works without verification.

Prefer:

- tests
- repository inspection
- build results
- runtime evidence

over assumptions.

## Evidence Discipline

Never describe repository-specific behavior based only on:
- file names
- directory names
- naming conventions
- assumptions
- general software conventions

Do not use speculative language such as:
- likely
- probably
- presumably
- typically
- should contain

when describing facts about this repository.

If the relevant source has not been inspected:
1. inspect it using the available repository tools, or
2. explicitly mark the information as UNVERIFIED.

Never present an assumption as repository evidence.