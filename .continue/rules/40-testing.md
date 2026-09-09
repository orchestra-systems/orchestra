# Testing Rules

Tests are evidence of behavior, not decoration.

## Test Planning

For every implementation task, identify:

- happy path
- invalid input/state
- failure path
- retry behavior where applicable
- concurrency concerns where applicable
- recovery behavior where applicable
- idempotency behavior where applicable

## Test Strategy

Prefer tests that verify observable behavior and system invariants.

Do not write tests merely to reproduce implementation details.

## Durable Execution

When relevant, explicitly test:

- duplicate delivery
- retry
- worker failure
- recovery
- invalid state transitions
- event ordering
- terminal state protection
- idempotency

## Verification

Never claim tests pass without actually running them.

Report:

- what was tested
- how it was tested
- result
- remaining limitations