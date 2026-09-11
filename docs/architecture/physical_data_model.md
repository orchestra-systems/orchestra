# Physical Data Model

## Purpose
This document maps the approved Logical Domain Model into a concrete relational schema for PostgreSQL. This model supports the core durable execution requirements defined in `SPEC.md`.

## Mapping from Logical Entities to Relational Tables

| Logical Entity | Relational Table |
| :--- | :--- |
| Workflow | `workflows` |
| WorkflowVersion | `workflow_versions` |
| WorkflowExecution | `workflow_executions` |
| Step | `steps` |
| StepExecution | `step_executions` |
| StepAttempt | `step_attempts` |
| Event | `events` |

Supporting entities (Agents, Tools, Workers) are referenced conceptually but their schema is out of scope for this initial core model.

## Table Definitions

### workflows
- `id`: UUID PRIMARY KEY
- `created_at`: TIMESTAMP

### workflow_versions
- `id`: UUID PRIMARY KEY
- `workflow_id`: UUID REFERENCES workflows(id)
- `version`: INTEGER
- `definition`: JSONB
- `created_at`: TIMESTAMP
- UNIQUE(`workflow_id`, `version`)

### workflow_executions
- `id`: UUID PRIMARY KEY
- `workflow_version_id`: UUID REFERENCES workflow_versions(id)
- `status`: TEXT
- `start_time`: TIMESTAMP
- `end_time`: TIMESTAMP (NULLABLE)

### steps
- `id`: UUID PRIMARY KEY
- `workflow_version_id`: UUID REFERENCES workflow_versions(id)
- `type`: TEXT
- `definition`: JSONB

### step_executions
- `id`: UUID PRIMARY KEY
- `workflow_execution_id`: UUID REFERENCES workflow_executions(id)
- `step_id`: UUID REFERENCES steps(id)
- `status`: TEXT

### step_attempts
- `id`: UUID PRIMARY KEY
- `step_execution_id`: UUID REFERENCES step_executions(id)
- `worker_id`: UUID
- `start_time`: TIMESTAMP
- `end_time`: TIMESTAMP (NULLABLE)
- `status`: TEXT

### events
- `id`: UUID PRIMARY KEY
- `workflow_execution_id`: UUID REFERENCES workflow_executions(id)
- `sequence_number`: BIGINT
- `event_type`: TEXT
- `payload`: JSONB
- `timestamp`: TIMESTAMP

## Relationships & Cardinality
- `workflows` (1) - (*) `workflow_versions`
- `workflow_versions` (1) - (*) `steps`
- `workflow_versions` (1) - (*) `workflow_executions`
- `workflow_executions` (1) - (*) `step_executions`
- `step_executions` (1) - (*) `step_attempts`
- `steps` (1) - (*) `step_executions`

## Constraints
- Foreign keys ensure referential integrity for core primary relationships.

## Indexing Considerations
- `workflow_executions(workflow_version_id)`
- `step_executions(workflow_execution_id)`
- `step_attempts(step_execution_id)`
- `events(workflow_execution_id, sequence_number)` (Index for ordering)

## Event Persistence Considerations
- Events are intended to be append-only.
- Event immutability is an invariant of the system state.

## Durability and Consistency Considerations
- The use of PostgreSQL transactions allows for atomic state updates (e.g., status changes and event logging).
- Strong consistency for core state transitions is required per `SPEC.md`.

## Open Questions
- Enforcement mechanism for Workflow Execution state transitions.
- Mechanism for generating/enforcing sequence_number.
- Concurrency control strategy for state transitions.
- Worker/Tool/Agent schema and coordination details.
- DSL/JSON Schema structure for workflow definitions.
- Causality/Parent-child representation in Events.

## Traceability
- `SPEC.md`: Sections 6.1-6.8 (Core concepts), 8-9 (Lifecycles), 10-11 (Event history/ordering), 38 (Versioning).
- `ARCHITECTURE.md`: Principles 2 (PostgreSQL as source of truth), 7 (Explicit state transitions).

