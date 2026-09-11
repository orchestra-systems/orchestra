# Orchestra Implementation Backlog

This document tracks implementation tasks for the Orchestra system, strictly separating design/decision-making from implementation.

## Implementation Gate
Before production code is implemented in `src/`, the following architectural decisions MUST be resolved:
1. Concurrency Control Strategy (TASK-01-01)
2. Event Sequence Allocation Design (TASK-01-02)
3. Transaction Boundary Strategy for State + Events (TASK-01-04)
---

## EPIC-01: Architecture Decisions
| Task ID | Title | Type | Status |
| :--- | :--- | :--- | :--- |
| TASK-01-01 | Concurrency Control Strategy Design | DESIGN | DESIGN |
| TASK-01-02 | Event Sequence Allocation Design | DESIGN | DESIGN |
| TASK-01-03 | State Machine & Publication Enforcement | DESIGN | DESIGN |
| TASK-01-04 | Transaction Boundary: State + Event | DESIGN | DESIGN |
| TASK-01-05 | Step/Version Integrity Design | DESIGN | DESIGN |

*(For brevity, task details follow the structure in the provided ruleset.)*

## EPIC-02: Durable Execution Persistence
| Task ID | Title | Type | Status |
| :--- | :--- | :--- | :--- |
| TASK-02-01 | Workflow basic persistence | IMPLEMENTATION | READY |
| TASK-02-02 | WorkflowVersion basic persistence | IMPLEMENTATION | BLOCKED |
| TASK-02-03 | WorkflowExecution basic creation/read | IMPLEMENTATION | BLOCKED |
| TASK-02-04 | Step persistence | IMPLEMENTATION | BLOCKED |

- **TASK-02-01**
    - **Objective:** CRUD for Workflow base entity.
    - **Dependencies:** None.
    - **Invariants Affected:** None.
    - **Blocked:** No.
- **TASK-02-02** (Blocked by TASK-01-03)
- **TASK-02-03** (Blocked by TASK-01-01, TASK-01-03)

## EPIC-06: Event History
| Task ID | Title | Type | Status |
| :--- | :--- | :--- | :--- |
| TASK-06-01 | Event basic persistence | IMPLEMENTATION | BLOCKED |

- **TASK-06-01**
    - **Status:** BLOCKED
    - **Dependencies:** TASK-01-02, TASK-01-04.

*(Other epics deferred until foundational architecture is stable.)*

---

## Dependency Graph

