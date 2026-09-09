# Orchestra — Durable Execution Infrastructure for AI Agents

**Organization:** `orchestra-systems`  
**Product:** `Orchestra`  
**Core concept:** Durable AI Execution

---

## AI Agent Runtime & Durable Workflow Engine

## 1. Document Status

- **Document type:** Technical Product & System Specification
- **Status:** Draft v1.1
- **Primary language:** English
- **Implementation status:** Not started
- **Target cloud:** Amazon Web Services (AWS)
- **Purpose:** Define the system's behavior, capabilities, constraints, guarantees, technology boundaries, cloud environment, and AI model strategy.
- **Audience:** Software engineers, system architects, AI coding agents, and AI agents responsible for architecture and implementation planning.

This document intentionally does **not** define implementation tasks, sprint plans, ticket breakdowns, code structure, or detailed implementation steps.

---

## 2. Vision

Build a production-grade platform for executing AI agents and long-running workflows reliably.

The platform should allow an AI agent to perform multi-step work involving:

- LLM reasoning
- tool invocation
- external APIs
- asynchronous jobs
- human approval
- persistent state
- retries
- timeouts
- scheduling
- parallel execution
- failure recovery
- multi-model execution

The central design principle is:

> An AI workflow must be treated as a durable distributed computation, not as a single HTTP request.

A workflow should be able to continue correctly even when individual workers, services, network connections, external dependencies, or human participants fail or become unavailable.

---

## 3. Problem Statement

Typical AI applications are implemented as:

```text
User
  |
  v
API
  |
  v
LLM
  |
  v
Tool
  |
  v
Response
```

This architecture is insufficient for complex agents.

Real-world agents may require:

```text
User
  |
  v
Workflow
  |
  +--> LLM reasoning
  |
  +--> Search
  |
  +--> External API
  |
  +--> Database
  |
  +--> Long-running computation
  |
  +--> Human approval
  |
  +--> Additional agent execution
  |
  v
Final result
```

Failures can occur at any point.

The platform therefore needs:

- durable execution
- persistent state
- event history
- idempotency
- retries
- timeouts
- authorization
- observability
- cost tracking
- model abstraction
- recovery mechanisms

## 4. Goals

### 4.1 Primary Goals

The system MUST:

- Execute multi-step workflows reliably.
- Persist workflow execution state.
- Survive worker crashes.
- Resume interrupted workflows.
- Support asynchronous and long-running execution.
- Support retries and backoff policies.
- Support execution timeouts.
- Support human-in-the-loop workflows.
- Support AI agents capable of tool calling.
- Maintain an auditable execution history.
- Provide structured observability.
- Support horizontal worker scaling.
- Provide strong isolation between workflows.
- Provide authentication and authorization.
- Track execution and LLM costs.
- Expose a clear API for creating and monitoring executions.
- Run production workloads on AWS.
- Support multiple AI model providers.
- Keep the production AI layer replaceable and model-agnostic.
- Support model routing based on task requirements.

## 5. Non-Goals

The initial system is NOT intended to be:

- A general-purpose Kubernetes replacement.
- A general-purpose cloud provider.
- A complete CRM.
- A general-purpose vector database.
- A foundation model.
- A browser automation platform.
- A replacement for existing message brokers.
- A fully autonomous unrestricted computer-use agent.
- A general-purpose ETL platform.
- A replacement for Amazon Bedrock.
- A replacement for an existing general-purpose workflow platform unless the project's own durable execution requirements justify custom infrastructure.

## 6. Core Concepts

### 6.1 Workflow

A workflow is a durable definition describing a sequence or graph of executable steps.

Example:

```text
START
  |
  v
Research
  |
  +------+
  |      |
  v      v
Search  Database
  |      |
  +------+
     |
     v
Generate Report
     |
     v
Human Approval
     |
     v
Publish
     |
     v
END
```

A workflow definition is separate from a workflow execution.

### 6.2 Workflow Execution

A workflow execution is one concrete instance of a workflow.

Example:

- Workflow:
- research_report

Execution:
exec_01JXYZ...

Multiple executions of the same workflow may run concurrently.

### 6.3 Step

A step is an individual unit of work.

Examples:

- LLM call
- HTTP request
- database operation
- code execution
- search
- human approval
- sub-workflow
- transformation
- conditional branch
- Each step has its own lifecycle and execution metadata.

### 6.4 Agent

An agent is an execution entity capable of reasoning and deciding which tools or actions to invoke.

An agent may:

- Receive context.
- Invoke an LLM.
- Receive tool-call instructions.
- Execute tools.
- Feed tool results back to the LLM.
- Continue until completion or termination.

### 6.5 Tool

A tool is an externally callable capability available to an agent.

Examples:

- web_search
- http_request
- database_query
- send_email
- create_file
- execute_code
- query_customer

Tools MUST have explicit schemas for:

- input
- output
- permissions
- timeout
- retry behavior
- risk level

### 6.6 Event

An event represents a state transition or meaningful occurrence in the system.

Examples:

- WorkflowCreated
- WorkflowStarted
- StepScheduled
- StepStarted
- StepCompleted
- StepFailed
- RetryScheduled
- ApprovalRequested
- ApprovalGranted
- ApprovalRejected
- WorkflowPaused
- WorkflowResumed
- WorkflowCompleted
- WorkflowFailed
- WorkflowCancelled

Events form the execution history of a workflow.

### 6.7 Worker

A worker executes actual units of work.

Workers MUST be stateless with respect to durable workflow state.

Durable state belongs to the platform's persistent storage.

### 6.8 Run

A run represents an execution attempt of a workflow step.

One logical step may have multiple runs because of retries.

Example:

- Step:
- call_payment_api

Run #1:
failed

Run #2:
failed

Run #3:
succeeded

## 7. Execution Model

### 7.1 Durable Execution

Workflow execution MUST be durable.

If a worker crashes:

```text
Worker
   |
   v
Step A
   |
   X CRASH
```

the workflow MUST remain recoverable.

After recovery:

```text
Persistent State
       |
       v
Scheduler
       |
       v
New Worker
       |
       v
Resume Execution
```

The system MUST NOT rely on in-memory worker state for correctness.

## 8. Workflow Lifecycle

A workflow execution has the following conceptual states:

```text
CREATED
   |
   v
RUNNING
   |
   +-------> PAUSED
   |           |
   |           v
   |         RUNNING
   |
   +-------> WAITING
   |           |
   |           v
   |         RUNNING
   |
   +-------> COMPLETED
   |
   +-------> FAILED
   |
   +-------> CANCELLED
```

Terminal states:

- COMPLETED
- FAILED
- CANCELLED
- Terminal executions MUST NOT transition to another state.

## 9. Step Lifecycle

A step may transition through:

```text
PENDING
  |
  v
SCHEDULED
  |
  v
RUNNING
  |
  +------> SUCCEEDED
  |
  +------> FAILED
  |
  +------> RETRY_WAITING
  |
  +------> CANCELLED
  |
  +------> TIMED_OUT
```

The state model MUST prevent invalid transitions.

## 10. Event History

Each workflow execution MUST maintain an ordered execution history.

Example:

- WorkflowStarted
- StepScheduled(search)
- StepStarted(search)
- StepCompleted(search)

StepScheduled(analyze)
StepStarted(analyze)
StepFailed(analyze)

RetryScheduled(analyze)
StepStarted(analyze)
StepCompleted(analyze)

WorkflowCompleted

The history MUST be sufficient to understand:

- what happened
- when it happened
- which worker performed it
- which attempt produced the result
- why a step failed
- whether the step was retried
- what the final workflow state was

## 11. Event Ordering

Events belonging to the same workflow execution MUST have a deterministic ordering.

Each event SHOULD contain a monotonically increasing sequence number within its workflow execution.

Example:

- 1 WorkflowStarted
- 2 StepScheduled
- 3 StepStarted
- 4 StepCompleted
- 5 StepScheduled
- 6 StepStarted
- 7 StepFailed
- 8 RetryScheduled
- 9 StepStarted
- 10 StepCompleted

Concurrent branches may execute in parallel, but the system MUST preserve enough causal information to reconstruct execution.

## 12. Idempotency

Idempotency is a core system requirement.

A worker may receive the same logical job more than once.

The system MUST distinguish between:

- logical execution identity
- physical execution attempts
- external side effects
- External side effects SHOULD use idempotency keys whenever supported.

Example conceptual identity:

- workflow_id
- +
- execution_id
- +
- step_id
- +
- logical_operation_id

The system MUST prevent duplicate state transitions from corrupting execution state.

## 13. Retry Semantics

Steps MAY define retry policies.

A retry policy may contain:

- maximum attempts
- initial delay
- maximum delay
- exponential backoff
- jitter
- retryable error classes
- non-retryable error classes
- Example:

- initial_delay: 1s
- max_delay: 60s
- max_attempts: 5
- backoff: exponential
- jitter: enabled

The platform MUST NOT blindly retry operations that may create unsafe duplicate side effects.

## 14. Timeout Semantics

Timeouts MAY exist at multiple levels:

- Workflow timeout
- Step timeout
- Tool timeout
- LLM timeout
- HTTP timeout
- Worker lease timeout

The system MUST distinguish between:

- operation timeout
- worker failure
- workflow cancellation
- external dependency timeout
- A timeout MUST produce an observable execution event.

## 15. Worker Failure

If a worker disappears while executing a step, the system MUST eventually detect the lost execution.

Worker liveness MAY be represented using:

- leases
- heartbeats
- visibility timeouts
- execution deadlines
- The system MUST avoid permanently orphaning workflow executions.

## 16. Cancellation

Users MUST be able to request workflow cancellation.

Cancellation semantics MUST distinguish:

- Cancellation requested

from:

- Cancellation completed

A currently executing external operation may not be immediately cancellable.

The platform MUST expose the actual cancellation state.

## 17. Pause and Resume

A workflow MAY be paused.

When paused:

- new steps MUST NOT start.
- currently executing operations MAY finish depending on policy.
- workflow state MUST remain durable.
- A paused workflow MUST be resumable without losing execution history.

## 18. Human-in-the-Loop

The platform MUST support workflows that wait for human decisions.

Example:

```text
Agent
  |
  v
Generate Action
  |
  v
Risk Evaluation
  |
  v
Approval Required
  |
  v
WAITING_FOR_APPROVAL
  |
  v
Human Decision
  |
  +---- APPROVE ----> Continue
  |
  +---- REJECT -----> Fail / Alternative Path
```

Approval requests MUST contain:

- workflow ID
- step ID
- requested action
- human-readable explanation
- structured action data
- risk level
- creation time
- expiration time
- current status
- Approvals MUST be auditable.

## 19. Approval Expiration

Approval requests MAY expire.

Possible outcomes:

- EXPIRED

The workflow may then:

- fail
- retry
- execute a fallback
- request another approval
- This behavior MUST be explicitly defined by the workflow.

## 20. Agent Runtime

The agent runtime is responsible for executing an agent loop.

Conceptually:

```text
Context
   |
   v
LLM
   |
   +---- final response ----> DONE
   |
   +---- tool call ----------> Tool
                                |
                                v
                             Result
                                |
                                v
                               LLM
                                |
                               ...
```

The runtime MUST support:

- system instructions
- user input
- conversation/context state
- tool definitions
- tool calls
- tool results
- termination conditions
- token usage tracking
- model metadata
- execution limits
- model selection

## 21. Agent Termination

An agent MUST NOT be allowed to run indefinitely.

Termination may occur because of:

- successful completion
- maximum iterations
- token budget
- time limit
- cost limit
- explicit cancellation
- unrecoverable error
- policy violation

## 22. Agent Context

Agent context may contain:

- System instructions
- User input
- Conversation history
- Workflow state
- Previous tool results
- Retrieved knowledge
- Relevant memory
- Execution metadata

The platform MUST distinguish between:

- durable workflow state
- transient runtime context
- persisted agent memory
- These are separate concepts.

## 23. Memory

The platform SHOULD support multiple memory types.

Short-Term Memory
Context belonging to a single execution.

Long-Term Memory
Information that may persist across executions.

Semantic Memory
Information retrievable through semantic/vector search.

Memory MUST have explicit ownership and access boundaries.

An agent MUST NOT automatically have unrestricted access to all stored information.

## 24. Tool Execution

Every tool invocation MUST have:

- unique invocation ID
- tool name
- validated input
- execution status
- start time
- completion time
- output or error
- execution metadata
- Tools MUST use explicit schemas.

Invalid tool arguments MUST be rejected before execution.

## 25. Tool Permissions

Tools MUST support permission controls.

Example:

- read:web
- read:database
- write:database
- send:email
- execute:code
- admin:system

An agent's available tools MUST be restricted by policy.

The runtime MUST NOT assume that an LLM-generated tool call is trusted.

## 26. Risk Levels

Tools MAY have risk classifications:

- LOW
- MEDIUM
- HIGH
- CRITICAL

Example:

- web_search       LOW
- database_read    LOW
- database_write   MEDIUM
- send_email       MEDIUM
- delete_resource  HIGH
- execute_code     CRITICAL

High-risk operations SHOULD support human approval.

## 27. Workflow Composition

A workflow SHOULD support:

- sequential execution
- conditional branching
- parallel branches
- joins
- loops
- retries
- delays
- human approval
- sub-workflows
- Example:

```text
             +--> Search Web ----+
             |                   |
Start --> Plan                    +--> Analyze --> End
             |                   |
             +--> Query DB ------+
```

Parallel branches MUST have explicit join semantics.

## 28. Sub-Workflows

A workflow MAY invoke another workflow.

Example:

```text
Main Workflow
    |
    +--> Research Workflow
    |
    +--> Validation Workflow
    |
    +--> Publishing Workflow
```

Sub-workflow executions MUST have their own execution identity and lifecycle.

The parent workflow MUST be able to observe the child workflow's status.

## 29. Scheduling

The platform SHOULD support delayed execution.

Examples:

- Run immediately
- Run after 10 minutes
- Run at timestamp T
- Run periodically

Scheduling MUST be durable.

A service restart MUST NOT cause scheduled workflows to disappear.

## 30. API Requirements

The platform MUST expose APIs for at least:

- Workflow Management
- Create workflow
- Get workflow
- List workflows
- Update workflow
- Delete/deactivate workflow

Execution Management
Start execution
Get execution
List executions
Cancel execution
Pause execution
Resume execution

Step / Event Inspection
Get execution steps
Get execution events
Get step attempts

Human Approval
List pending approvals
Get approval
Approve
Reject

Agent Management
Create agent
Get agent
Update agent
Execute agent

The exact API protocol and endpoint structure are implementation decisions.

## 31. API Characteristics

APIs MUST support:

- authentication
- authorization
- request validation
- idempotency where appropriate
- pagination
- structured errors
- request IDs
- execution IDs
- consistent status representation
- Long-running operations MUST NOT depend on keeping an HTTP request open.

## 32. Asynchronous Execution

Starting a workflow SHOULD return quickly.

Example conceptual response:

- {
- "execution_id": "exec_123",
- "status": "RUNNING"
- }

The client can then observe execution through:

- polling
- streaming
- webhook
- server-sent events
- WebSocket
- The initial system SHOULD support at least one real-time execution observation mechanism.

## 33. Multi-Tenancy

The architecture MUST allow multiple isolated users or organizations.

Conceptually:

```text
Organization
   |
   +-- Users
   |
   +-- Agents
   |
   +-- Workflows
   |
   +-- Executions
   |
   +-- Tools
   |
   +-- Credentials
```

Tenant data MUST NOT leak across organizations.

Every durable resource SHOULD have an explicit ownership boundary.

## 34. Authentication

The platform MUST authenticate API clients.

Authentication mechanisms MAY include:

- API keys
- JWT
- OAuth/OIDC
- The implementation should support a clear path toward enterprise identity providers.

## 35. Authorization

Authorization MUST be enforced server-side.

The system SHOULD support at least:

- Organization
- Project
- Resource
- Role
- Permission

Example roles:

- Owner
- Admin
- Developer
- Operator
- Viewer

## 36. Secrets

Secrets MUST NOT be stored as plain workflow configuration.

Examples:

- API keys
- database passwords
- OAuth tokens
- cloud credentials
- Secrets SHOULD be referenced indirectly.

Example:

- tool:
- send_email

credential:
credential_123

rather than embedding the secret directly in workflow configuration.

On AWS, the preferred production secret-management boundary is AWS Secrets Manager or an equivalent managed mechanism.

## 37. Data Model Requirements

The system is expected to persist entities conceptually similar to:

- Organization
- User
- Project
- Agent
- AgentVersion
- Workflow
- WorkflowVersion
- WorkflowExecution
- StepExecution
- StepAttempt
- Event
- Tool
- ToolInvocation
- Approval
- CredentialReference
- Memory
- Schedule
- Model
- ModelPolicy

Exact schemas are implementation decisions.

The data model MUST support:

- historical execution inspection
- concurrent executions
- retries
- event ordering
- ownership
- versioning
- model selection
- cost tracking

## 38. Workflow Versioning

Workflow definitions MUST be versioned.

An execution MUST reference the workflow version from which it was created.

Example:

- Workflow:
- research_agent

Version 1
Version 2
Version 3

An execution started using Version 2 MUST NOT silently change behavior because Version 3 was deployed.

## 39. Agent Versioning

Agents SHOULD also be versioned.

A workflow execution SHOULD record:

- agent version
- model
- model configuration
- tool configuration
- relevant system prompt version
- This is required for debugging and reproducibility.

## 40. LLM Provider Abstraction

The system MUST NOT be tightly coupled to a single LLM provider.

The platform SHOULD support:

- Amazon Bedrock
- OpenAI-compatible providers
- Anthropic-compatible providers
- future providers
- Amazon Bedrock SHOULD be the primary production integration because AWS is the target cloud.

The application SHOULD use a provider abstraction layer so that changing models does not require changing workflow semantics.

## 41. Model Routing

Model selection SHOULD be treated as a runtime policy rather than hard-coded into individual workflows.

Conceptually:

```text
Task
 |
 v
Model Policy
 |
 +--> Reasoning task ----> High-reasoning model
 |
 +--> Simple extraction -> Fast/cheap model
 |
 +--> Classification ----> Small model
 |
 +--> Embedding ---------> Embedding model
 |
 +--> Vision ------------> Multimodal model
```

A model policy MAY consider:

- task type
- latency requirement
- quality requirement
- token budget
- cost budget
- context size
- tool-use capability
- modality
- region
- availability

## 42. Recommended Production Model Strategy

The initial platform SHOULD NOT depend on a single model.

A baseline strategy SHOULD include:

- Reasoning / Complex Agentic Tasks
- Use a high-capability reasoning model available through Amazon Bedrock.

Candidate models may include:

- OpenAI GPT-5.6 family
- Anthropic Claude Opus/Sonnet family
- Amazon Nova Premier
- The final default should be determined through evaluation rather than assumption.

Fast / Low-Cost Tasks
Candidate models include:

- Amazon Nova Micro
- Amazon Nova Lite
- Anthropic Claude Haiku
- other cost-efficient Bedrock models
- Embeddings
- Candidate models include:

- Amazon Titan Text Embeddings V2
- Cohere Embed models
- Multimodal
- Candidate models include:

- Amazon Nova
- Anthropic Claude vision-capable models
- other Bedrock multimodal models
- The platform MUST record the exact model used for every inference.

## 43. Model Evaluation

Model selection MUST eventually be evaluation-driven.

The system SHOULD support evaluation datasets containing representative tasks.

Evaluation dimensions SHOULD include:

- Task correctness
- Tool-call correctness
- Reasoning quality
- Latency
- Token usage
- Cost
- Failure rate
- Safety
- Structured-output validity

The platform SHOULD make it possible to compare multiple models on the same workload.

## 44. Model Fallback

The runtime SHOULD support model fallback.

Example:

```text
Primary Model
     |
     X unavailable
     |
     v
Fallback Model
     |
     v
Continue
```

Fallback policy MUST consider:

- semantic compatibility
- tool-use support
- structured output compatibility
- context limits
- cost
- safety policy
- A fallback model MUST NOT silently change business-critical behavior without being observable.

## 45. Cost Tracking

Every LLM execution SHOULD record estimated cost.

The platform SHOULD support:

- Execution cost
- Step cost
- Agent cost
- Workflow cost
- Organization cost
- Model cost
- Provider cost

Cost tracking MUST distinguish between:

- input tokens
- output tokens
- cached tokens where applicable
- tool execution cost
- external service cost where measurable

## 46. Observability

Observability is a first-class requirement.

Logs
Structured logs containing:

- timestamp
- severity
- service
- workflow ID
- execution ID
- step ID
- request ID
- worker ID
- Metrics
- At minimum:

- workflow executions
- workflow failures
- workflow duration
- step duration
- step failures
- retry count
- queue depth
- worker utilization
- LLM latency
- LLM token usage
- LLM cost

Traces
Distributed traces SHOULD connect:

```text
API Request
   |
Workflow Execution
   |
Step
   |
Worker
   |
LLM
   |
Tool
   |
External API
```

## 47. Auditability

Important actions MUST be auditable.

Examples:

- workflow creation
- workflow modification
- execution start
- cancellation
- approval
- rejection
- credential changes
- permission changes
- high-risk tool execution
- model-policy changes
- Audit records SHOULD be immutable.

## 48. Security Requirements

The system MUST follow secure-by-default principles.

Requirements include:

- encrypted transport
- encrypted sensitive data at rest
- authentication
- authorization
- secret isolation
- tenant isolation
- input validation
- output validation where appropriate
- tool permission enforcement
- audit logging
- model/provider access control
- The LLM MUST NOT be considered a trusted security boundary.

## 49. Prompt Injection

The system MUST assume that external content may contain malicious instructions.

Examples:

- Web page
- Email
- PDF
- Database content
- User-uploaded document
- Tool response

The agent runtime MUST distinguish between:

- Trusted instructions

and:

- Untrusted external content

Tool results MUST NOT automatically override system-level policies.

## 50. Code Execution

If code execution is supported, it MUST be isolated from the core control plane.

The execution environment SHOULD provide:

- resource limits
- CPU limits
- memory limits
- execution timeout
- filesystem isolation
- network restrictions
- process isolation
- Arbitrary code MUST NOT execute directly inside the API server or workflow control plane.

## 51. AWS Cloud Environment

AWS is the target production cloud.

The implementation SHOULD favor managed AWS services when they materially improve:

- reliability
- security
- operational simplicity
- scalability
- observability
- The system should not introduce self-managed infrastructure merely for the sake of technical complexity.

## 52. AWS Service Baseline

The following services are the preferred starting points.

Compute
Preferred options:

- Amazon EKS for Kubernetes-based workloads
- ECS/Fargate where Kubernetes is unnecessary
- AWS Lambda for appropriate short-lived event-driven functions
- The architecture should allow the final implementation plan to evaluate EKS versus ECS/Fargate rather than assuming Kubernetes is mandatory for every component.

Networking
Preferred AWS components:

- VPC
- private subnets
- public subnets only where necessary
- security groups
- Application Load Balancer where appropriate
- NAT Gateway where required
- VPC endpoints where useful
- Database
- Preferred:

- Amazon RDS for PostgreSQL
- Amazon Aurora PostgreSQL where scale or availability requirements justify it
- PostgreSQL remains the logical system of record.

Cache
Preferred:

- Amazon ElastiCache for Redis
- Redis MUST remain an optimization/coordination layer rather than the durable workflow source of truth.

Object Storage
Preferred:

- Amazon S3
- Use cases:

- workflow artifacts
- generated files
- uploaded documents
- execution outputs
- large payloads
- Secrets
- Preferred:

- AWS Secrets Manager
- AWS KMS
- Sensitive credentials MUST NOT be committed to source control.

Identity
Preferred:

- AWS IAM for infrastructure identity
- Amazon Cognito or external OIDC provider for application users, depending on final product requirements
- AI
- Preferred:

- Amazon Bedrock
- Bedrock SHOULD be the primary production AI gateway.

Messaging
The architecture MAY use:

- Amazon MSK / Kafka
- Amazon SQS
- Amazon SNS
- EventBridge
- The final choice must be based on the semantics required by the workflow engine.

The specification does NOT mandate Kafka simply because the project is distributed.

Monitoring
Preferred:

- Amazon CloudWatch
- AWS X-Ray where useful
- OpenTelemetry
- Prometheus
- Grafana
- OpenTelemetry SHOULD remain the application-level observability abstraction.

Container Registry
Preferred:

- Amazon ECR
- Infrastructure as Code
- Preferred:

- Terraform
- AWS-native alternatives may be considered if they provide a clear advantage.

## 53. AWS Architecture Principles

The AWS deployment SHOULD follow:

- Least Privilege
- Every workload should receive only the permissions required for its responsibilities.

Private by Default
Internal services should remain private unless public exposure is necessary.

Managed Services First
Prefer managed AWS services when they reduce operational burden without compromising the project's learning objectives.

Failure Isolation
A failure in one workload should not cascade into the entire platform.

Multi-AZ Readiness
Production-critical stateful services SHOULD support Multi-AZ deployment where practical.

Infrastructure Reproducibility
Production infrastructure MUST be reproducible through infrastructure-as-code.

## 54. Infrastructure Environments

At minimum, the project SHOULD distinguish:

- local
- development
- staging
- production

Environment boundaries MUST prevent accidental production access from development workloads.

## 55. Deployment Strategy

The production deployment SHOULD support:

- automated builds
- automated tests
- immutable container artifacts
- infrastructure validation
- controlled rollout
- rollback
- database migration management
- health checks
- Deployment strategy may use:

- rolling deployments
- blue/green
- canary
- The final choice is an implementation decision.

## 56. AI-Assisted Engineering

AI coding agents are considered part of the development workflow for this project.

The project SHOULD use multiple specialized AI agents rather than expecting a single model to perform every engineering task.

The engineering workflow is conceptually:

```text
Specification
     |
     v
Architecture Agent
     |
     v
Implementation Plan
     |
     +------------------+
     |                  |
     v                  v
Coding Agent       Review Agent
     |                  |
     +--------+---------+
              |
              v
         Test / Verify
              |
              v
          Documentation
```

The specification remains the highest-level source of truth.

AI agents MUST NOT silently modify the specification to justify an implementation.

## 57. AI Agent Roles for Development

### 57.1 Architect Agent

Responsibilities:

- analyze SPEC.md
- identify architectural constraints
- propose system architecture
- identify trade-offs
- identify risks
- challenge assumptions
- The Architect Agent SHOULD NOT implement code.

### 57.2 Planning Agent

Responsibilities:

- convert the approved specification into an implementation plan
- define implementation phases
- identify dependencies
- identify interfaces
- identify testing requirements
- identify migration/deployment concerns
- The Planning Agent SHOULD NOT begin implementation.

### 57.3 Coding Agent

Responsibilities:

- implement approved plan items
- write tests
- run tests
- inspect existing code
- make small, reviewable changes
- report deviations
- The Coding Agent SHOULD NOT redefine system requirements.

### 57.4 Review Agent

Responsibilities:

- review implementation against specification
- inspect correctness
- identify architectural violations
- inspect security concerns
- inspect concurrency and failure handling
- inspect test coverage
- identify hidden coupling

### 57.5 Documentation Agent

Responsibilities:

- maintain technical documentation
- explain architecture
- maintain API documentation
- document operational procedures
- generate diagrams where useful
- maintain ADRs
- Documentation MUST reflect actual implementation rather than inventing behavior.

## 58. Recommended Coding Agent Stack

The preferred development stack is:

- Primary Coding Agent
- OpenAI Codex CLI.

Codex is an OpenAI coding agent designed to run locally in the terminal and work directly against the repository.

Secondary Coding / Review Agent
Claude Code.

Claude Code is an agentic coding tool designed to understand a repository, execute development tasks, and handle Git workflows.

Alternative / Fast Exploration Agent
Gemini CLI.

Gemini CLI provides a terminal-based coding agent with file operations, shell execution, web fetching, Google Search grounding, and MCP support.

The project SHOULD avoid making one agent the only source of engineering judgment.

## 59. Recommended Agent Skills / Methodology

The project SHOULD use:

- obra/superpowers

Superpowers provides a reusable agentic software-development methodology with skills covering:

- brainstorming
- specification refinement
- implementation planning
- TDD
- systematic debugging
- parallel agents
- code review
- Git worktrees
- subagent-driven development
- verification
- Superpowers should be treated as a development methodology layer, not as a runtime dependency of the product.

## 60. AI Development Governance

AI coding agents MUST follow these principles:

- Read the relevant specification before implementation.
- Do not change requirements without explicit approval.
- Do not invent APIs or behavior that contradicts the specification.
- Verify assumptions against repository state.
- Prefer tests over claims.
- Report uncertainty.
- Keep changes reviewable.
- Avoid unnecessary abstraction.
- Preserve existing invariants.
- Never claim successful implementation without verification.

## 61. AI Model Selection for Development

Different development activities SHOULD use different models when appropriate.

Deep Architecture / Reasoning
Preferred:

- GPT-5.6-class reasoning models
- Claude Opus-class models
- Use for:

- architecture
- distributed systems analysis
- failure-mode analysis
- security review
- difficult debugging
- Implementation
- Preferred:

- OpenAI Codex
- Claude Code
- Gemini CLI
- The model underneath the agent may vary.

The important requirement is that the coding agent has sufficient:

- repository context
- tool access
- terminal access
- test execution
- Git integration
- Documentation
- Preferred:

- Claude
- GPT-5.6-class models
- Documentation tasks SHOULD favor accuracy, consistency, and context preservation over raw coding capability.

## 62. AI Model Separation

Development-time AI models and production runtime models MUST be treated as independent concerns.

Example:

- Development

```text
GPT / Claude
     |
     v
Coding Agent
     |
     v
Repository
```

Production

```text
Application
     |
     v
Agent Runtime
     |
     v
Model Router
     |
     +--> Bedrock Model A
     +--> Bedrock Model B
     +--> Bedrock Model C
```

Changing the development coding agent MUST NOT require changing the production architecture.

Changing the production LLM MUST NOT require changing the development workflow.

## 63. AI Agent Repository Strategy

The project SHOULD maintain AI-agent-specific instructions in repository-level documentation.

Potential files include:

- SPEC.md
- ARCHITECTURE.md
- AGENTS.md
- CLAUDE.md
- CODEX.md
- GEMINI.md
- CONTRIBUTING.md

These files should contain agent-specific operational instructions.

SPEC.md remains the product/system specification.

Agent-specific instructions MUST NOT silently override SPEC.md.

## 64. Testing Requirements

The system SHOULD have multiple levels of testing.

Unit Tests
For:

- state transitions
- retry policies
- validation
- authorization
- workflow logic
- model routing policies
- Integration Tests
- For:

- database
- queue
- worker execution
- LLM providers
- tool execution
- AWS integrations
- Failure Tests
- Critical scenarios include:

- Worker crash
- Database connection failure
- Queue unavailable
- LLM timeout
- Tool timeout
- Duplicate delivery
- Network partition
- Deployment during execution
- Process restart
- Model provider unavailable
- Model rate limit
- AWS service dependency failure

End-to-End Tests
At minimum:

- Create workflow
- Start execution
- Execute multiple steps
- Retry failed step
- Pause workflow
- Request approval
- Approve
- Resume
- Complete workflow

## 65. Reliability Targets

The initial production target SHOULD be:

- API availability:              99.9%
- Durable execution recovery:    required
- No silent workflow loss:       required
- No cross-tenant data access:   required

The system should prioritize correctness over raw throughput.

## 66. Consistency Model

The system SHOULD favor strong consistency for:

- workflow state transitions
- execution ownership
- approval state
- cancellation state
- version references
- Eventual consistency is acceptable for:

- analytics
- dashboards
- non-critical derived views
- usage aggregation

## 67. Delivery Semantics

The platform may use at-least-once delivery internally.

Exactly-once execution SHOULD NOT be assumed at the infrastructure level.

Correctness should instead be achieved through:

- durable state
- idempotency
- deduplication
- transactional state transitions
- explicit execution identities

## 68. Backpressure

The system MUST handle load spikes.

Example:

- Normal:
- 100 jobs/minute

Spike:
10,000 jobs/minute

The platform SHOULD:

- queue work
- limit worker concurrency
- apply backpressure
- prevent database overload
- prevent uncontrolled LLM calls
- expose queue depth

## 69. Rate Limiting

Rate limits MAY exist at:

- Organization
- User
- API key
- Workflow
- Agent
- Tool
- LLM provider
- Model

The platform SHOULD prevent a single tenant or workflow from exhausting shared resources.

## 70. Resource Limits

Executions SHOULD support limits such as:

- max execution duration
- max steps
- max agent iterations
- max tool calls
- max token usage
- max estimated cost
- max concurrent branches

These limits protect both reliability and cost.

## 71. Failure Classification

Failures SHOULD be classified into categories:

- VALIDATION_ERROR
- AUTHORIZATION_ERROR
- TRANSIENT_ERROR
- DEPENDENCY_ERROR
- TIMEOUT
- RATE_LIMIT
- RESOURCE_EXHAUSTED
- POLICY_VIOLATION
- USER_ERROR
- INTERNAL_ERROR

Retry behavior MUST depend on failure classification.

## 72. Dead Letter Handling

Repeatedly failing executions or jobs SHOULD eventually be moved into a dead-letter state.

The system MUST preserve enough information to diagnose the failure.

Operators SHOULD be able to inspect:

- original input
- execution history
- failure reason
- attempts
- timestamps
- worker information

## 73. Data Retention

Execution history and logs SHOULD have configurable retention policies.

Retention MAY differ for:

- Workflow state
- Events
- Logs
- Traces
- LLM prompts
- LLM responses
- Audit records

Sensitive data SHOULD have stricter retention controls.

## 74. Privacy

The platform SHOULD support data minimization.

The system SHOULD allow configuration determining whether LLM prompts and responses are persisted.

Not every piece of runtime context needs to be permanently stored.

## 75. Local Development

The system SHOULD be runnable locally using containers.

A developer should eventually be able to start the complete environment with a single documented command.

The local environment should provide equivalents for:

- PostgreSQL
- Redis
- Message broker
- Object storage
- API
- Workers
- Agent runtime
- Frontend
- Observability

AWS services may be emulated locally where practical, but production behavior MUST be validated against real AWS services before production deployment.

## 76. Deployment Model

The system SHOULD support deployment as independent services where appropriate.

Conceptual architecture:

```text
                    Internet
                       |
                       v
                 API Gateway / ALB
                       |
              +--------+--------+
              |                 |
              v                 v
         Control Plane      Streaming API
              |
       +------+------+
       |             |
       v             v
   Scheduler      Workflow API
       |
       v
    Queue/Event Bus
       |
   +---+---+---+
   |   |   |   |
   v   v   v   v
 Worker Worker Worker Worker
       |
       +----------+
       |          |
       v          v
     LLM        Tools
```

The exact service boundaries are implementation decisions.

## 77. Technology Stack

Frontend
Next.js
React
TypeScript
Core API / Control Plane
Go
Agent Runtime
Python
Primary Database
PostgreSQL
AWS RDS PostgreSQL or Aurora PostgreSQL in production
Cache / Coordination
Redis
AWS ElastiCache for Redis in production
Messaging
Potential choices:

- Kafka / Redpanda
- Amazon MSK
- Amazon SQS
- Amazon EventBridge
- Final selection is an architecture decision.

Vector Search
pgvector
Object Storage
Amazon S3
Secrets
AWS Secrets Manager
AWS KMS
AI
Amazon Bedrock
Containers
Docker
Amazon ECR
Orchestration
Amazon EKS or ECS/Fargate
Infrastructure
Terraform
Observability
OpenTelemetry
Amazon CloudWatch
Prometheus
Grafana
CI/CD
GitHub Actions

## 78. Technology Principles

Technology choices should follow these principles:

- PostgreSQL is the durable source of truth unless there is a strong reason otherwise.
- Redis is not the source of truth for workflow execution.
- Workers should be horizontally scalable.
- Control-plane services should remain independent from AI model execution.
- AI provider integrations should be replaceable.
- External tools should be treated as unreliable dependencies.
- Infrastructure failures must not corrupt workflow state.
- The system should prefer explicit state transitions over hidden state.
- Operational simplicity is preferred until scale requires additional complexity.
- Every distributed component must have a clearly defined failure model.
- AWS managed services should be preferred when they reduce operational burden.
- Vendor lock-in is acceptable where AWS provides substantial reliability or operational advantages, but core workflow semantics should remain portable.
- Production AI model selection must remain configurable.

## 79. Operational Dashboard

The platform SHOULD provide a dashboard showing:

- Active executions
- Failed executions
- Completed executions
- Queued jobs
- Worker health
- Approval requests
- Average execution latency
- Failure rate
- Retry rate
- LLM usage
- LLM cost
- Model distribution
- Provider health

An operator should be able to drill down from:

- Organization
- -> Workflow
- -> Execution
- -> Step
- -> Attempt
- -> Logs / Trace / Events

## 80. Developer Experience

A developer should be able to:

- Define an agent.
- Define tools.
- Define a workflow.
- Run it locally.
- Observe execution.
- Inspect failures.
- Retry execution.
- Add human approval.
- Select or configure model policies.
- Deploy the workflow.
- Monitor production executions.
- Inspect AI cost and model usage.

## 81. Design Constraints

The system MUST respect the following constraints.

Constraint 1
Durability is more important than raw performance.

Constraint 2
The LLM is not trusted.

Constraint 3
Workers can disappear at any time.

Constraint 4
External APIs can fail or behave unpredictably.

Constraint 5
Messages may be delivered more than once.

Constraint 6
Network communication is unreliable.

Constraint 7
A workflow may run for seconds, minutes, hours, or potentially days.

Constraint 8
Users may interact with the workflow while it is executing.

Constraint 9
Workflow definitions may evolve while previous executions are still running.

Constraint 10
AI output is probabilistic and cannot be treated as deterministic business logic.

Constraint 11
Production model providers may become unavailable.

Constraint 12
AWS service dependencies may fail or become throttled.

## 82. Initial Success Criteria

The platform will be considered technically successful when it can demonstrate the following scenario:

## 1. User creates a workflow.

## 2. Workflow contains:

- LLM step
- tool step
- parallel steps
- human approval

## 3. User starts execution.

## 4. Multiple workers execute the workflow.

## 5. One worker crashes during execution.

## 6. The workflow recovers.

## 7. One step fails transiently.

## 8. The platform retries it.

## 9. The workflow reaches human approval.

## 10. The workflow remains durable while waiting.

## 11. Human approves the action.

## 12. Workflow resumes.

## 13. A final external side effect executes idempotently.

## 14. Execution completes.

## 15. Operator can inspect:

- complete event history
- all step attempts
- logs
- traces
- LLM usage
- model used
- estimated cost

## 16. The same workflow can be executed using a different compatible model without changing workflow semantics.

## 83. Future Capabilities

The architecture SHOULD leave room for:

- distributed agent teams
- agent-to-agent communication
- workflow marketplace
- workflow templates
- scheduled autonomous agents
- advanced memory
- browser automation
- sandboxed computer use
- model routing
- model fallback
- model evaluation
- prompt versioning
- AI safety policies
- enterprise SSO
- advanced billing
- multi-region execution
- cross-region disaster recovery
- workflow replay
- workflow simulation
- agent evaluation pipelines
- These are explicitly outside the initial scope.

## 84. Final Product Definition

The system can be summarized as:

- A durable execution platform for AI agents and long-running workflows that combines workflow orchestration, distributed workers, tool execution, human approval, persistent state, event history, multi-model AI integration, and AWS-native infrastructure.

The core abstraction is:

```text
                WORKFLOW
                    |
                    v
              EXECUTION
                    |
          +---------+---------+
          |         |         |
          v         v         v
        STEP      STEP      STEP
          |         |         |
          v         v         v
        WORKER    WORKER    WORKER
          |         |         |
          +---------+---------+
                    |
                    v
              EVENT HISTORY
                    |
                    v
             DURABLE STATE
                    |
                    v
              MODEL ROUTER
                    |
          +---------+---------+
          |         |         |
          v         v         v
       Model A   Model B   Model C
```

The defining property of the system is:

- A workflow should be able to continue correctly despite failures of individual processes, workers, network connections, external dependencies, AI model providers, and human delays.

## 85. Specification Boundary

This document intentionally ends at the specification level.

The next engineering phase should independently determine:

- system architecture
- service boundaries
- AWS architecture
- database schema
- event schema
- workflow DSL
- queue strategy
- concurrency model
- consistency strategy
- API design
- repository structure
- implementation sequence
- testing strategy
- deployment architecture
- task decomposition
- model evaluation methodology
- Those decisions should be derived from this specification rather than prematurely encoded here.

The implementation plan MUST NOT be treated as part of this specification.
