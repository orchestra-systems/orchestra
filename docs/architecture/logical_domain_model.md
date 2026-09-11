# Logical Domain Model

## Purpose
The purpose of this document is to define the logical domain model for Orchestra, providing a conceptual framework for durable AI execution. It serves as the foundation for subsequent architectural decisions and implementation tasks, ensuring alignment with `SPEC.md`.

## Modeling Principles
- **Durable Execution:** The model focuses on entities that must persist to support recovery from worker or process failure.
- **Auditable History:** Entities contribute to an immutable or append-only event-based history for execution reconstruction.
- **Core vs. Supporting:** Distinguishes between the core execution state machine and supporting entities referenced during execution.
- **Strict Adherence:** Properties are defined based on explicit requirements in `SPEC.md`.

## Core Durable Execution Entities

### Workflow
- **Definition:** A container for workflow versions.
- **Required properties:** ID.
- **Relationships:** One Workflow has many WorkflowVersions, WorkflowExecutions.
- **Source references:** SPEC.md Section 6.1.

### WorkflowVersion
- **Definition:** An immutable, versioned definition of a sequence or graph of executable steps.
- **Required properties:** ID, WorkflowID, Definition (the graph/sequence logic).
- **Source references:** SPEC.md Section 38.

### WorkflowExecution
- **Definition:** A concrete instance of a workflow, using a specific WorkflowVersion.
- **Required properties:** ID, WorkflowVersionID, Status (CREATED, RUNNING, PAUSED, WAITING, COMPLETED, FAILED, CANCELLED), StartTime, EndTime.
- **Relationships:** Consists of many StepExecutions.
- **Source references:** SPEC.md Section 6.2, 8, 38.

### StepExecution
- **Definition:** A concrete instance of a step within a workflow execution.
- **Required properties:** ID, WorkflowExecutionID, StepID, Status (PENDING, SCHEDULED, RUNNING, SUCCEEDED, FAILED, RETRY_WAITING, CANCELLED, TIMED_OUT).
- **Relationships:** Consists of many StepAttempts.
- **Source references:** SPEC.md Section 6.3, 9.

### StepAttempt
- **Definition:** An execution attempt of a step. (Referred to as "Run" in SPEC.md).
- **Required properties:** ID, StepExecutionID, StartTime, EndTime, Status.
- **Relationships:** Associated with a Worker.
- **Source references:** SPEC.md Section 6.8.

### Event
- **Definition:** A state transition or meaningful occurrence in the system.
- **Required properties:** ID, WorkflowExecutionID, SequenceNumber (monotonically increasing), EventType, Payload, Timestamp.
- **Source references:** SPEC.md Section 6.6, 10, 11.

## Supporting / Referenced Entities

### Agent
- **Definition:** An execution entity capable of reasoning and deciding which tools or actions to invoke.
- **Required properties:** ID, Name, SystemInstructions, Version, ModelPolicy.
- **Source references:** SPEC.md Section 6.4, 39.

### Tool
- **Definition:** An externally callable capability available to an agent.
- **Required properties:** ID, Name, Schema (Input/Output), RiskLevel.
- **Source references:** SPEC.md Section 6.5, 24, 26.

### Worker
- **Definition:** An entity that executes units of work.
- **Required properties:** ID.
- **Source references:** SPEC.md Section 6.7, 15.

## Relationships
```text
Workflow (1) -- (*) WorkflowVersion
WorkflowVersion (1) -- (*) WorkflowExecution
WorkflowExecution (1) -- (*) StepExecution
WorkflowExecution (1) -- (*) Event
StepExecution (1) -- (*) StepAttempt
StepAttempt (*) -- (1) Worker
Agent (1) -- (*) Tool
```

## Durability Requirements
The following state must survive worker/process failure:
- WorkflowExecution status and history.
- StepExecution status.
- StepAttempt status and history.
- Event log (to allow reconstruction).
- Pending approvals.

## Auditability Requirements
- The Event log provides the ordered history.
- Every workflow execution must have a complete, auditable history of events.
- Audit records must be immutable.

## Open Questions
- DSL/JSON Schema for Workflow Definitions.
- Causality/Parent-child representation in Events.
- Detailed Agent memory structure (Short-term vs. Long-term vs. Semantic).
- Tool schema definition format.
- Liveness/Lease mechanism for Workers.
