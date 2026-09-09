# Orchestra Architecture

> This document is intentionally a living architecture document. It must describe the architecture that has actually been decided, not speculative implementation details.

## Status

- Status: Initial scaffold
- Specification: `SPEC.md`
- Organization: `orchestra-systems`

## Architecture Principles

1. Durability over raw performance.
2. PostgreSQL is the durable source of truth unless explicitly justified otherwise.
3. Redis is not the source of truth for workflow execution.
4. Workers may disappear at any time.
5. Messages may be delivered more than once.
6. External dependencies are unreliable.
7. State transitions should be explicit.
8. Every distributed component must have a defined failure model.
9. Control-plane services remain independent from model execution.
10. AI provider integrations remain replaceable.
11. Prefer operational simplicity until scale requires additional complexity.

## Current Architecture

_To be filled through explicit architectural decisions and ADRs._

## Open Architectural Questions

- Service boundaries
- Workflow representation / DSL
- Execution state model
- Event model
- Worker coordination and lease model
- Queue/messaging strategy
- Concurrency and consistency model
- Recovery model
- API design
- Observability architecture
- Model routing abstraction

## Related

- `SPEC.md`
- `adr/`
- `docs/architecture/`
