# System Invariants

This document defines the confirmed invariants for the Orchestra system, distinguishing between structural requirements, domain rules, and operational constraints.

## 1. Structural / Referential Integrity
These invariants ensure the consistency of the graph of data within the system.

- **Step-Execution Consistency**
    - **Invariant:** A `StepExecution` must reference a `Step` that belongs to the same `WorkflowVersion` as its associated `WorkflowExecution`.
    - **Why it matters:** Prevents orphaned executions or cross-version execution corruption.
    - **Entities:** `WorkflowExecution`, `WorkflowVersion`, `StepExecution`, `Step`.
    - **PG Enforcement:** Yes (via composite foreign keys).
    - **App Responsibility:** Enforce during task scheduling/dispatch.
    - **Enforcement Open?** No.

- **WorkflowExecution Pinning**
    - **Invariant:** An execution is permanently bound to a single `WorkflowVersion`.
    - **Why it matters:** Ensures consistency throughout the execution lifecycle; `workflow_version_id` must be treated as immutable.
    - **Entities:** `WorkflowExecution`, `WorkflowVersion`.
    - **PG Enforcement:** Referential integrity ensures the ID remains valid, but does not prevent application-layer updates.
    - **App Responsibility:** Must enforce immutability of `workflow_version_id` after creation.
    - **Enforcement Open?** No.

## 2. Domain Invariants
Rules defining valid system state transitions and properties.

- **WorkflowVersion Uniqueness**
    - **Invariant:** A `version` number must be unique within a single `Workflow`.
    - **Why it matters:** Provides distinct identity for workflow definitions.
    - **Entities:** `Workflow`, `WorkflowVersion`.
    - **PG Enforcement:** Yes (via `UNIQUE(workflow_id, version)`).
    - **App Responsibility:** N/A.
    - **Enforcement Open?** No.

- **WorkflowVersion Immutability**
    - **Invariant:** A published `WorkflowVersion` is immutable.
    - **Why it matters:** Ensures reproducible executions.
    - **Entities:** `WorkflowVersion`.
    - **PG Enforcement:** Limited (possible via restrictive permissions).
    - **App Responsibility:** Enforce read-only access/logic after publication.
    - **Enforcement Open?** Yes (requires architectural decision on state enforcement).

## 3. Concurrency Invariants
Constraints governing state transitions under parallel operations.

- **WorkflowVersion Monotonicity**
    - **Invariant:** Versions must be strictly increasing.
    - **Why it matters:** Ensures version ordering and prevents conflicting updates.
    - **Entities:** `WorkflowVersion`.
    - **PG Enforcement:** No (PG can enforce uniqueness, but not that the next number is strictly greater than the previous).
    - **App Responsibility:** Must manage strictly increasing sequence generation.
    - **Enforcement Open?** Yes (requires decision on sequence/locking mechanism).

- **Note on Terms:**
    - **Uniqueness vs. Monotonicity:** Uniqueness ensures no duplicates; monotonicity ensures valid progression. Uniqueness is database-enforceable; monotonicity requires concurrency control.

## 4. Durability Invariants
Constraints ensuring data persistence and integrity.

- **Event Immutability**
    - **Invariant:** Events are append-only and immutable.
    - **Why it matters:** Serves as the source of truth for execution history.
    - **Entities:** `Event`.
    - **PG Enforcement:** Limited (possible via permissions).
    - **App Responsibility:** Enforce write-once semantics.
    - **Enforcement Open?** No.

## 5. Audit / Event-history Invariants
Constraints on the order and integrity of events.

- **Event Sequence Uniqueness**
    - **Invariant:** `sequence_number` must be unique per `WorkflowExecution`.
    - **Why it matters:** Prevents collision and ensures distinct ordering.
    - **Entities:** `Event`, `WorkflowExecution`.
    - **PG Enforcement:** Yes (via unique index).
    - **App Responsibility:** Must manage allocation of numbers.
    - **Enforcement Open?** Yes (requires decision on allocation mechanism).

- **Event Sequence Ordering**
    - **Invariant:** `sequence_number` defines increasing order of events within an execution.
    - **Why it matters:** Allows deterministic event stream replay.
    - **Entities:** `Event`.
    - **PG Enforcement:** No (enforces existence, not temporal ordering).
    - **App Responsibility:** Must ensure allocation respects ordering.
    - **Enforcement Open?** Yes.

- **Note on Terms:**
    - **Monotonicity vs. Contiguity:** Monotonicity ensures numbers increase; contiguity ensures no gaps. We require monotonicity, but *do not* require contiguity.
    - **Ordering vs. Causality:** Ordering defines when things happened in the stream; causality relates events to their triggers. Causality is currently out of scope.

## Open Architectural Questions
- Concurrency control strategy (e.g., Optimistic Locking vs. SELECT FOR UPDATE).
- Event sequence allocation mechanism (how to handle high-concurrency generation).
- State-machine enforcement mechanism.
- Worker lease management.
- Messaging/Coordination infrastructure.
- Definition of event causality/parent-child relationships.
