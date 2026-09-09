# Logical Domain Model

## Purpose
The purpose of this document is to define the logical domain model for Orchestra, providing a conceptual framework for durable AI execution. It serves as the foundation for subsequent architectural decisions and implementation tasks, ensuring alignment with `SPEC.md`.

## Modeling Principles
- **Durable Execution:** The model focuses on entities that must persist to support recovery from worker or process failure.
- **Auditable History:** Entities contribute to an immutable or append-only event-based history for execution reconstruction.
- **Strict Adherence:** Properties are defined based on explicit requirements in `SPEC.md`. No properties have been added for speculation.
- **Separation of Concerns:** The model distinguishes between definitions (immutable, versioned) and executions (mutable, stateful).

## Entities

### Workflow
- **Definition:** A durable definition describing a sequence or graph of executable steps.
- **Required properties:** ID, Version, Definition (the graph/sequence logic).
- **Relationships:** One Workflow has many WorkflowVersions, WorkflowExecutions.
- **Lifecycle relevance:** Active or Inactive.
- **Source references:** SPEC.md Section 6.1, 38.
- **Open questions:** How is the workflow definition graph represented? (DSL/JSON schema).

### WorkflowExecution
- **Definition:** A concrete instance of a workflow.
- **Required properties:** ID, WorkflowID, WorkflowVersionID, Status (CREATED, RUNNING, PAUSED, WAITING, COMPLETED, FAILED, CANCELLED), StartTime, EndTime.
- **Relationships:** Belongs to a Workflow; consists of many StepExecutions.
- **Lifecycle relevance:** Main unit of durable execution.
- **Source references:** SPEC.md Section 6.2, 8.
- **Open questions:** None.

### Step
- **Definition:** An individual unit of work within a workflow.
- **Required properties:** ID, WorkflowID, Type, Definition (e.g., tool configuration, LLM parameters).
- **Relationships:** Part of a Workflow.
- **Lifecycle relevance:** Defines potential execution unit.
- **Source references:** SPEC.md Section 6.3.
- **Open questions:** None.

### StepExecution
- **Definition:** A concrete instance of a step within a workflow execution.
- **Required properties:** ID, WorkflowExecutionID, StepID, Status (PENDING, SCHEDULED, RUNNING, SUCCEEDED, FAILED, RETRY_WAITING, CANCELLED, TIMED_OUT).
- **Relationships:** Belongs to a WorkflowExecution; consists of many StepAttempts.
- **Lifecycle relevance:** Tracks the status of a specific step in an execution.
- **Source references:** SPEC.md Section 6.3, 9.
- **Open questions:** None.

### StepAttempt
- **Definition:** An execution attempt of a step.
- **Required properties:** ID, StepExecutionID, WorkerID, StartTime, EndTime, Status, Output/Error metadata.
- **Relationships:** Belongs to a StepExecution.
- **Lifecycle relevance:** Tracks retry attempts.
- **Source references:** SPEC.md Section 6.8.
- **Open questions:** None.

### Event
- **Definition:** A state transition or meaningful occurrence in the system.
- **Required properties:** ID, WorkflowExecutionID, SequenceNumber (monotonically increasing), EventType, Payload, Timestamp.
- **Relationships:** Associated with a WorkflowExecution.
- **Lifecycle relevance:** Essential for reconstructing execution history.
- **Source references:** SPEC.md Section 6.6, 10, 11.
- **Open questions:** How is causality (parent-child events) represented for parallel execution?

### Run
- **Definition:** An execution attempt of a workflow step (synonymous with StepAttempt in this model).
- **Required properties:** (Merged with StepAttempt).
- **Relationships:** (Merged with StepAttempt).
- **Lifecycle relevance:** Tracking success/failure of an attempt.
- **Source references:** SPEC.md Section 6.8.
- **Open questions:** Should Run and StepAttempt be separate or one entity?

### Agent
- **Definition:** An execution entity capable of reasoning and deciding which tools or actions to invoke.
- **Required properties:** ID, Name, SystemInstructions, Version, ModelPolicy.
- **Relationships:** Linked to Workflow/Step via execution context.
- **Lifecycle relevance:** Persistent entity for AI capabilities.
- **Source references:** SPEC.md Section 6.4, 39.
- **Open questions:** Agent memory structure (transient vs. durable).

### Tool
- **Definition:** An externally callable capability available to an agent.
- **Required properties:** ID, Name, Schema (Input/Output), RiskLevel.
- **Relationships:** Available to Agents.
- **Lifecycle relevance:** Tool discovery and validation.
- **Source references:** SPEC.md Section 6.5, 24, 26.
- **Open questions:** How is "Schema" represented?

### Worker
- **Definition:** An entity that executes units of work.
- **Required properties:** ID, LivenessStatus (heartbeat/lease).
- **Relationships:** Performs StepAttempts.
- **Lifecycle relevance:** Transient, stateless relative to workflow state.
- **Source references:** SPEC.md Section 6.7, 15.
- **Open questions:** How is liveness tracking implemented without centralizing state in the worker?

## Relationships
```text
Workflow (1) -- (*) WorkflowExecution
Workflow (1) -- (*) Step
WorkflowExecution (1) -- (*) StepExecution
WorkflowExecution (1) -- (*) Event
StepExecution (1) -- (*) StepAttempt
StepAttempt (1) -- (1) Worker
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
- Liveness/Lease mechanism details.
