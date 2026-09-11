# ADR-0002 — Concurrency Control Strategy

- Status: Proposed
- Date: 2025-01-21

## Context

Orchestra is a durable execution platform for AI agents and long-running workflows. Multiple stateless workers may concurrently operate on the same `WorkflowExecution`, `StepExecution`, or `StepAttempt`. PostgreSQL is the durable source of truth.

The system must guarantee:

- Strongly consistent state transitions.
- Only one worker may successfully claim a `StepExecution` into `RUNNING`.
- Duplicate or stale completion attempts do not corrupt execution state.
- Recovery from worker/process failure without relying on in-memory state.

This decision addresses how concurrent state transitions are coordinated at the database layer. It does not address worker liveness, external side-effect idempotency, event allocation, or messaging infrastructure, which remain open architectural questions.

## Decision

State-transition concurrency shall use **optimistic concurrency / compare-and-swap (CAS)** as the default mechanism.

### What CAS Guarantees

CAS guarantees an atomic read-modify-write operation expressed as:

> Update the row **only if** the durable state still matches the state observed by the updater.

In PostgreSQL, this is expressed conceptually as:

```sql
UPDATE step_executions
SET status = 'RUNNING'
WHERE id = :id
  AND status = 'PENDING';
```

The database reports the number of rows affected. Affected row count of **1** means the transition succeeded. Affected row count of **0** means the expected state no longer holds; the transition did not occur.

### CAS Failure Semantics

When a CAS update affects **0 rows**:

- The operation MUST be treated as a conflict, not as a silent success.
- The caller MUST NOT assume the row is in the intended target state.
- The caller SHOULD re-read the current state and decide the next action based on the current state and business rules.
- The system MAY raise a conflict error, return a sentinel value, or route the stale worker back into a recovery/retry path, depending on the operation.
- A 0-row update MUST NOT be retried blindly without re-reading state, because the state may have moved to a terminal or otherwise invalid source state.

CAS provides **atomic conditional update**. It does not guarantee that the winning transition is the most recently issued command, only that it is valid with respect to the state observed at commit time.

## Scope

This decision applies to state transitions where strong consistency is required and contention is expected to be low-to-moderate:

- `WorkflowExecution` state transitions (e.g., `RUNNING` → `PAUSED`, `RUNNING` → `COMPLETED`).
- `StepExecution` state transitions (e.g., `PENDING` → `RUNNING`, `RUNNING` → `SUCCEEDED`, `RUNNING` → `FAILED`).
- `StepAttempt` state transitions where applicable (e.g., start/completion of a single attempt record).
- Retry creation where concurrency matters (e.g., transitioning a `StepExecution` from `RETRY_WAITING` / `FAILED` back to a claimable state).

CAS operates on the durable row state. The compare token may be the current state/status value, a dedicated version column, or any column that changes atomically with the transition. The exact column used for the compare token is an implementation detail left to subsequent design.

## Duplicate Claim Invariant

The system MUST enforce the following invariant:

> Only one worker may successfully transition a claimable `StepExecution` into `RUNNING`.

The conceptual CAS operation for claiming is:

> Transition `StepExecution` to `RUNNING` only if the current state is still the expected claimable state (e.g., `PENDING` or `SCHEDULED`).

If two workers concurrently issue:

```sql
UPDATE step_executions
SET status = 'RUNNING'
WHERE id = :step_execution_id
  AND status = 'PENDING';
```

PostgreSQL serializes the updates. Exactly one statement will affect one row; the other will affect zero rows. The losing worker detects the 0-row result and MUST NOT proceed as if it owns the step.

This satisfies the **Single Ownership** invariant defined in `docs/architecture/concurrency-control-analysis.md`.

## Duplicate Completion

Concurrent completion attempts operate under the same CAS rule.

Example: two processes attempt `RUNNING` → `SUCCEEDED` for the same `StepExecution`.

```sql
UPDATE step_executions
SET status = 'SUCCEEDED'
WHERE id = :step_execution_id
  AND status = 'RUNNING';
```

Only one update succeeds. The second receives a 0-row result. The second caller MUST re-read the row, observe that the state is already `SUCCEEDED`, and treat the operation as idempotent from a state perspective.

The system MUST NOT emit duplicate completion events solely because two callers attempted the same transition. Event emission rules are governed by the separate transaction-boundary and event-allocation decisions (TASK-01-02 and TASK-01-04).

CAS prevents duplicate state corruption; it does not by itself prevent duplicate event emission. That requires the transaction boundary and idempotency-key designs yet to be decided.

## Retry Races and Stale Workers

A specific failure scenario must be considered:

1. Worker A claims a `StepExecution` and transitions it to `RUNNING`.
2. Worker A becomes unreachable (crash, network partition, long GC pause).
3. After a timeout/lease expiration, Worker B retries/reclaims the same step, creating a new `StepAttempt`.
4. Worker A resumes and attempts to write a completion or heartbeat against the original attempt.

CAS will protect the `StepExecution` row: if Worker A tries to write `RUNNING` → `SUCCEEDED` after Worker B has already moved the step to a new state, the CAS will affect 0 rows. Worker A will detect the conflict and must abandon or reconcile.

However, **CAS alone does NOT solve the stale-worker problem**. Specifically:

- CAS does not detect that Worker A has died; it only detects state mismatch.
- CAS does not prevent Worker A from performing external side effects between the claim and the eventual conflict detection.
- CAS does not provide fencing of external resources or idempotency of external calls.
- CAS does not guarantee that Worker A will stop executing in-memory logic after it has been superseded.

Worker liveness, lease expiration, and fencing tokens require a separate architectural decision. This ADR explicitly excludes those concerns. Implementers MUST NOT assume that CAS eliminates the need for worker leases or external side-effect idempotency.

## SELECT FOR UPDATE Comparison

`SELECT ... FOR UPDATE` is a pessimistic locking mechanism that serializes access to a row within a transaction. It can guarantee that no other transaction modifies the row until the lock is released.

### Why it can serialize access

Holding a row lock for the duration of a transaction prevents concurrent transactions from acquiring the same lock or writing to the same row. For short, purely database-bound operations, this can simplify application logic by removing the need for application-layer retry loops.

### Why it is dangerous for long-running work

`SELECT FOR UPDATE` becomes dangerous when the lock is held across:

- External I/O (HTTP calls to LLMs, tools, or third-party APIs).
- Long-running computations.
- Worker heartbeats or lease checks.
- Suspensions waiting for human approval.

Long-held locks consume database connections and can lead to:

- Connection pool exhaustion.
- Cascading latency under contention.
- Deadlocks when multiple rows are locked in inconsistent orders.
- Reduced throughput under normal load.

Orchestra's steps routinely involve external AI/tool calls, so holding a database lock for the duration of a step is incompatible with the system's failure model.

### Where short-lived pessimistic locking could still be appropriate

Short-lived pessimistic locking may still be appropriate for operations that are:

- Confined to a single database transaction.
- Very short duration (milliseconds).
- High-contention.
- Not blocking on external I/O.

Candidate future uses include:

- Allocating the next `sequence_number` for an `Event` when gapless ordering is required.
- Updating a small coordinator row during lease renewal, if the lease design chooses to use row locking.

These uses are possible but are **not** the default state-transition mechanism. Any introduction of `SELECT FOR UPDATE` must be justified in a future ADR or design document.

### Why the architecture does not choose it as the default

The default state-transition mechanism must survive:

- Worker crashes.
- Long-running external calls.
- Variable step duration.
- Multiple workers competing for the same execution.

`SELECT FOR UPDATE` held across step execution violates these requirements. CAS allows workers to release the database connection immediately after the conditional update and perform long-running work without holding a lock.

## Explicit Limitations

This ADR does **NOT** decide the following items. They remain OPEN QUESTIONS for future ADRs:

- Worker lease/liveness mechanism.
- Fencing tokens for stale workers.
- External side-effect idempotency.
- Event `sequence_number` allocation mechanism.
- Event ordering and causality semantics.
- State-machine transition rules (which transitions are legal).
- Transaction boundaries for combined state + event writes.
- Messaging/queue infrastructure.
- Redis coordination patterns.
- Implementation framework, ORM, or repository pattern.

Implementers MUST NOT infer decisions about these topics from this ADR.

## Retry / Idempotency

Duplicate command delivery is expected in the system. Messages may be delivered more than once (SPEC.md §5, §67). CAS provides a first line of defense: a duplicate command that targets a stale state will fail the conditional update and can be ignored or reconciled.

However, CAS is not a complete idempotency mechanism. Two important distinctions:

1. **Concurrency decision (this ADR):** Defines how concurrent state transitions are coordinated so that only valid, non-conflicting transitions succeed.
2. **Future idempotency-key design:** Will define how logically duplicate commands are recognized across requests, how deduplication windows are managed, and how idempotency keys are stored and garbage-collected.

This ADR does **not** design the idempotency-key schema, storage, or lookup mechanism. It only notes that CAS-based state transitions reduce the risk of duplicate state corruption, and that a separate idempotency design will be required for command deduplication and external side-effect safety.

## Consequences

### Benefits

- **Durability under worker failure:** Workers do not hold database locks while executing steps, so a crashed worker cannot leave a long-held lock behind.
- **Horizontal scalability:** Workers can be added without increasing database lock contention.
- **Operational simplicity:** No lock timeout tuning or deadlock detection is required for normal state transitions.
- **Strong consistency for transitions:** PostgreSQL serializes concurrent CAS updates; exactly one transition wins.
- **Compatibility with long-running steps:** Steps may wait for LLMs, tools, human approvals, or timers without consuming a database connection.

### Costs

- **Application-layer retry logic:** Losers of CAS conflicts must re-read state and retry or reconcile, increasing application complexity.
- **Contention under high concurrency:** Bursty contention on a single `WorkflowExecution` or `StepExecution` increases conflict rate and retry volume.
- **No ordering guarantee between commands:** The command that reaches the database first (in serial order) wins; later commands see a 0-row result. This is correct for state transitions but may require careful handling for commands that logically depend on ordering.

### Operational Implications

- Metrics SHOULD track CAS conflict rate per entity type.
- Alerting SHOULD flag sustained high conflict rates, which may indicate a hot row or a missing lease mechanism.
- Workers MUST handle 0-row CAS results deterministically; unhandled conflicts are a likely source of bugs.
- Database connection pools do not need to be sized for long-held locks, but they MUST be sized for normal query volume plus retry bursts.

### Application Retry Requirements

- Application code MUST treat 0-row CAS updates as a first-class outcome.
- Retries MUST be preceded by a fresh read of the current state.
- Blind unconditional retries of the same UPDATE statement are prohibited.
- Retry policies for CAS conflicts should be bounded and may use backoff to avoid thundering-herd behavior.

## Rejected Alternatives

### SELECT FOR UPDATE for normal state transitions

Rejected as the default because holding row locks across long-running, I/O-bound step execution conflicts with the system's failure model and would risk connection exhaustion and cascading latency. It may be reconsidered for short-lived, database-only operations in future designs.

### SERIALIZABLE transactions as the general solution

Rejected because:

- `SERIALIZABLE` isolation does not remove the need for application handling of serialization failures.
- It adds complexity and performance cost without providing clearer semantics than CAS for the specific problem of conditional single-row updates.
- It does not address the fundamental hazard of long-held locks or external I/O.
- For single-row conditional updates, CAS is simpler and more explicit.

## Decision Boundaries

### This ADR decides

- The default concurrency mechanism for durable state transitions: optimistic concurrency / CAS.
- The semantics of a 0-row CAS update.
- The scope of entities to which CAS applies by default.
- The invariant that only one worker may successfully claim a `StepExecution` into `RUNNING`.
- The comparison with `SELECT FOR UPDATE` and why it is not the default.
- The conceptual relationship between CAS and duplicate command delivery.

### This ADR does not decide

- Legal state-machine transitions.
- Event `sequence_number` allocation.
- Worker lease/liveness design.
- Fencing tokens.
- External side-effect idempotency.
- Transaction boundaries for combined state + event writes.
- Messaging infrastructure.
- Redis coordination.
- Implementation framework or ORM.
- Database column choice for the CAS compare token.

## Failure Considerations

- **Worker crash during a step:** The worker does not hold a lock; the step remains in `RUNNING` until a lease/timeout mechanism reclaims it. CAS ensures that only the legitimate owner (per current state) can complete or reclaim.
- **Stale worker resume:** CAS detects state mismatch and rejects the stale write. The stale worker must abandon the operation. External side effects performed by the stale worker are not fenced by this ADR.
- **Duplicate messages:** CAS prevents duplicate transitions from corrupting state, but duplicate event emission and duplicate external side effects require separate mechanisms.
- **High contention:** CAS conflict rate rises with contention. Application retry logic must be bounded and instrumented.
- **Lost update without CAS:** Any unconditional `UPDATE` of state would violate the single-ownership invariant. CAS is required.

## Operational Impact

- No long-lived database locks are held by workers during step execution.
- Connection pool sizing is driven by query throughput, not lock duration.
- CAS conflict metrics should be part of standard observability.
- Incident response for stuck executions must consider worker lease/liveness, which is outside the scope of this ADR.

## Related

- SPEC.md: §5 (messages may be delivered more than once), §7 (durable execution), §8 (workflow lifecycle), §9 (step lifecycle), §12 (idempotency), §14 (timeout semantics), §15 (worker failure), §66 (consistency model), §67 (delivery semantics), §81 (design constraints).
- Architecture: `ARCHITECTURE.md` (principles 2, 4, 5, 7, 8, 10).
- Invariants: `docs/architecture/invariants.md`.
- Domain model: `docs/architecture/logical_domain_model.md`.
- Data model: `docs/architecture/physical_data_model.md`.
- Concurrency analysis: `docs/architecture/concurrency-control-analysis.md`.
- Backlog: `docs/backlog.md`.
- Task: TASK-01-01.