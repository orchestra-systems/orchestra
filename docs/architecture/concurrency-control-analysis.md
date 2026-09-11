# Concurrency Control Analysis

## 1. Problem Statement
Orchestra requires a durable, reliable concurrency-control strategy to manage the state of long-running, multi-step AI workflows. As a distributed system, we must ensure state consistency, prevent race conditions during state transitions (e.g., step claims, completions, retries), and handle concurrent message processing while maintaining the integrity of the execution history.

## 2. Required Invariants
As defined in `docs/architecture/invariants.md` and `SPEC.md`:
- **State Transition Integrity:** Invalid state transitions (e.g., transition from terminal state) must be prevented.
- **Single Ownership:** Only one worker can be in a `RUNNING` state for a given `StepExecution` at any time.
- **Deterministic Ordering:** Events within a workflow must be append-only and have a monotonically increasing `sequence_number`.
- **Idempotency:** Duplicate delivery of execution commands must not lead to inconsistent state or duplicate side effects.

## 3. Concurrency Scenarios

### Scenario 1: Duplicate Step Claim
*Worker A and B both attempt to move a `StepExecution` from `PENDING` to `RUNNING`.*
- **Acceptable Outcome:** Only one worker succeeds; the other receives a conflict error or is notified that the step is already claimed.
- **Invariant:** Atomic state transition (e.g., `WHERE status = 'PENDING'`).

### Scenario 2: Duplicate State Transition
*Two processes attempt `RUNNING` → `SUCCEEDED` for the same step.*
- **Acceptable Outcome:** One wins, the second fails (typically harmless, as the result is already achieved).
- **Invariant:** Atomic state check and update.

### Scenario 3: Retry Race
*A failure triggers a retry while another process is still handling the current step.*
- **Acceptable Outcome:** The system must avoid creating conflicting duplicate attempts or transitioning to an invalid state. The state machine must distinguish between "retryable" and "terminal" states.

### Scenario 4: Worker Crash
*A worker claims a `StepExecution` and disappears.*
- **Acceptable Outcome:** The step must eventually be reclaimable.
- **Dependence:** Relies on a Worker Lease mechanism (out of scope for this analysis, but impacts concurrency design).

### Scenario 5: Duplicate Message
*The same command is delivered multiple times.*
- **Acceptable Outcome:** Idempotent processing; the state machine must recognize that the command has already been processed or that the target state is already reached.

### Scenario 6: Concurrent Workflow Execution State Changes
*Two processes attempt conflicting transitions on a `WorkflowExecution` (e.g., Pause while Cancelling).*
- **Acceptable Outcome:** The workflow remains in a valid, consistent terminal or operational state. Requires serialization or robust state transitions.

## 4. Candidate Strategies

| Strategy | Correctness | Contention Handling | Throughput | Implementation Complexity |
| :--- | :--- | :--- | :--- | :--- |
| **Optimistic (CAS)** | High | Requires retry logic | High | Low |
| **Pessimistic (FOR UPDATE)** | High | Blocks concurrent access | Lower | Low/Medium |
| **Serializable TX** | Very High | High impact | Lowest | Medium |

## 5. Comparison Matrix

| Criteria | Optimistic (CAS) | Pessimistic (SELECT FOR UPDATE) |
| :--- | :--- | :--- |
| **Suitability** | Best for low-contention state transitions | Best for critical, high-contention segments |
| **Failure Behavior** | Application handles retries | DB blocks, timeouts possible |
| **Long-running workflows** | Excellent | Risky if held across IO/API calls |
| **Scalability** | Good | Limited by DB connection/locking |

## 6. Failure Analysis
- **Optimistic Locking:** Conflicts during high contention lead to increased application-layer retry frequency.
- **Pessimistic Locking:** Risks blocking DB connections if locks are held too long, potentially leading to cascading failure in the worker pool.

## 7. Trade-offs
- Optimistic locking favors availability and throughput but adds complexity to application-layer retry logic.
- Pessimistic locking simplifies application logic but is sensitive to transaction duration.

## 8. Recommended Direction (Preliminary)
A hybrid approach is recommended:
1. **Optimistic Locking (CAS)** for most state transitions (e.g., `StepExecution` status changes) to ensure maximum scalability.
2. **Pessimistic Locking (`FOR UPDATE`)** only for highly critical, short-lived operations, specifically when allocating the next `sequence_number` in a workflow execution to guarantee strictly increasing, gap-safe order (if required).

## 9. Decisions Requiring Human Approval
- Adoption of a hybrid locking strategy.
- Whether to strictly enforce gapless sequences (currently not required, but requested for ordering).
- Definition of the exact error-handling strategy for optimistic lock collisions.

## 10. Consequences
- **Hybrid approach:** Increased implementation complexity due to two different locking patterns.
- **Optimistic Locking:** Requires robust application-level retry policies for contention.
- **Pessimistic Locking:** Requires strict transaction hygiene to prevent connection exhaustion.

---
### Status
- **Determined by Arch:** Durable state is Postgres, strong consistency for state transitions.
- **Architectural Decision:** Concurrency control strategy (Optimistic vs Pessimistic).
- **Missing Information:** Required performance/contention characteristics for the event sequence allocation (does it need to be contention-free or is DB locking sufficient?).
