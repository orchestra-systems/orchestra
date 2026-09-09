# Documentation Rules

Documentation must describe behavior that actually exists.

## Source Priority

Prefer:

1. implementation
2. tests
3. accepted ADRs
4. SPEC.md

Do not document planned behavior as implemented behavior.

## Documentation Changes

When implementation changes behavior:

- identify affected documentation
- update only relevant documentation
- preserve existing terminology
- avoid speculative details

## ADRs

Use ADRs for significant architectural decisions.

An ADR should capture:

- Context
- Problem
- Decision
- Alternatives considered
- Consequences

Do not create an ADR for an unresolved question.

Mark unresolved architectural issues as:

OPEN QUESTION

## Verification

Before documenting a behavior, verify it exists in the repository.

Never claim:

- an API exists when it does not
- a feature is implemented when it is not
- a guarantee exists without evidence