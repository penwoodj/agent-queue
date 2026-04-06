# Agent Queue vs yaml-to-rust-agentsdk: Comprehensive Analysis

## Executive Summary

**Purpose**: Understand capabilities overlap and differentiation between `agent-queue` (queue orchestration system) and `yaml-to-rust-agentsdk` (workflow execution engine) to identify what agent-queue uniquely provides vs. what should be delegated to transpiler.

**Key Finding**: These are **complementary systems** with minimal but important overlap. Agent-queue provides **orchestration and coordination** around workflows, while transpiler provides **execution and code generation** for those workflows.

**Recommendation**: Agent-queue should focus on queue-specific features and delegate all execution-related concerns to transpiler.

---

## System Comparison Matrix

| Aspect | Agent Queue | yaml-to-rust-agentsdk | Overlap | Owner |
|---------|-------------|----------------------|---------|--------|
| **Primary Domain** | Queue orchestration | Workflow execution | Low | Distinct |
| **Input Format** | YAML workflows (enqueued to queues) | YAML workflows (executed directly) | Complete | Shared |
| **Core Function** | Schedule, dispatch, track workflows | Execute workflows, generate code | Low | Complementary |
| **Execution Model** | Queue → Agent → Workflow Engine | Direct execution OR compiled code generation | None | Different |
| **State Persistence** | SQLite (runs, tasks, steps, artifacts) | Checkpoints, versioned state | Medium | Both have state |
| **Scheduling** | 25 queue levels, meta-scheduler, priority aging, deadlines | Parallel/serial/hybrid modes | None | Distinct |
| **Retry Logic** | Queue-level exponential backoff, DLQ | Step-level retry, validation loops | Medium | Different semantics |
| **Observability** | Structured JSON logs with 9 levels | Hierarchical logging (9 levels) | Medium | Different approaches |
| **LLM Integration** | Single provider trait (llama-cpp) | Multi-model (LM Studio, Ollama, llama.cpp, Jina AI) | High | Transpiler richer |
| **Tool System** | Sandbox allowlist (shell, file_read, file_write) | Built-in tools (file, web, shell) + custom | Medium | Different scopes |
| **Multi-tenancy** | Full support (tenant isolation, quotas) | Single-process | None | Agent-queue unique |
| **Hooks & Events** | Hook-triggered queue category (QL-026) | Webhooks, git hooks, file watchers, timers, email, DB triggers | None | Agent-queue unique |
| **Advanced Scheduling** | EDF, WRR, DRR, fair share, work stealing | Priority + aging only | None | Agent-queue unique |
| **Output Format** | Artifacts + logs + metrics | Console + logs + chat + files + metrics + state + compiled executables | None | Transpiler richer |

---

## Detailed Feature Comparison

### 1. Workflow Definition (Shared Foundation)

#### Agent Queue (MVP: YA-01)
```yaml
schema_version: "0.1.0"
name: my-workflow
queue:
  category: asap | whenever | scheduled | cron
  priority: critical | high | normal | low
  schedule:
    cron: "0 2 * * *"  # For cron
    due_ts: "2025-04-03T02:00:00-00"  # For scheduled
    timezone: "America/Chicago"
retry:
  max_attempts: 3
model:
  provider: llama-cpp
  model_path: ./models/llama-3.2-3b-q4_k_m.gguf
  temperature: 0.7
  max_tokens: 4096
tools:
  allowed: [shell, file_read, file_write]
  blocked: [network]  # No network in MVP
steps:
  - id: step-1
    description: "Process input"
    prompt:
      user: "Analyze this: {{input_file}}"
    timeout_s: 120
    on_error: abort  # abort | continue
```

#### Transpiler (Full YAML Schema)
```yaml
workflow_id: ai_code_refactor
name: "AI-Powered Code Refactoring"

models:
  primary:
    provider: lmstudio
    model: "llama-3.2-3b-instruct"
    backend: vulkan

  secondary:
    provider: ollama
    model: "llama3.2"

execution:
  mode: parallel | serial | hybrid
  memory:
    max_allocated_memory_mb: 16384
    model_memory_mb: 4096
    unload_unused: true

logging:
  global:
    level: debug | info | error
    detail: high | medium | low
    output_type: chat | log | stateless_direct_io
    format: json

agentic_workflow:
  - step: analyze_code
    id: step_1
    model: "${models.primary}"
    input:
      prompt: "Analyze the codebase"
      code_path: ./src/
    output:
      save_to: analysis
      format: json
      fields: [issues, suggestions]
    retry:
      max_attempts: 3
      backoff_strategy: exponential

  - step: validate_code
    model: "${models.primary}"
    validation_loop:
      type: validation
      exact_criteria: true
      tolerance: 0.0
      max_iterations: 5
      stop_conditions:
        - validation.compiles == true
        - validation.tests_pass == true
```

**Analysis**:
- Agent Queue: **Simpler schema** focused on queue integration
- Transpiler: **Richer schema** supporting multi-model, memory management, complex validation loops
- **Overlap**: Both use YAML, but transpiler schema is more comprehensive
- **Recommendation**: Agent Queue should adopt transpiler's schema richness for workflow definition, OR transpiler should provide a **lite schema** for queue integration

---

### 2. Execution Orchestration (Complementary)

#### Agent Queue (IN-01 through IN-07)
```
┌─────────────────────────────────────────────────────────────────────────┐
│                    agent-queue (single process)          │
│                                                         │
│  ┌──────────┐    ┌──────────────┐    ┌─────────────────┐  │
│  │   CLI     │───▶│ Queue Engine  │───▶│ Stubbed Agent  │───▶│ llama.cpp │  │
│  │  (Clap)   │    │ (Scheduler)  │    │   Executor     │    │   (llama-   │  │
│  └──────────┘    └──────┬───────┘    └──────┬─────────┘    │
│                         │               │              │             │              │
│                    └───────┬───────┘              │             │              │
│                           │               │              │             │              │
│                    ┌────▼────┐    ┌─────────────┐    │             │              │
│                    │ SQLite    │    │ Artifacts   │    │             │              │
│                    │ (WAL)    │    │ (files)     │    │             │              │
│                    └───────────┘    └─────────────┘    │             │              │
│                                                         │
│                    ┌─────────────────────────────────────────────┐    │
│                    │         Logs (JSON) + Audit Trail     │    │
│                    └─────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
```

**Key Points**:
- **Queue Engine**: Scheduler coordinates task execution
- **Stubbed Executor**: Simulates workflow execution with sleep-based mocks (MVP only)
- **Integration Point**: IN-01 (queue dispatches to agent)
- **Single Process**: All components in one Tokio runtime

#### Transpiler (Execution Modes)

**Development Mode (Direct Execution)**:
```
1. Load YAML workflow
2. Validate schema
3. Initialize execution context
4. Execute agentic_workflow steps:
   a. Load model
   b. Execute prompt
   c. Capture output
   d. Save to variables
   e. Unload model if needed
5. Handle branching and loops
6. Execute validation loops
7. Generate logs and metrics
8. Return final output
```

**Production Mode (Code Generation)**:
```
1. Load YAML workflow
2. Validate schema
3. Generate WorkflowIR (internal representation)
4. Generate Rust code using templates:
   a. Agent definitions with model management
   b. Tool implementations (file, web, shell)
   c. Scheduler and queue logic
   d. State management and checkpointing
   e. Logging and metrics collection
5. Compile Rust code:
   a. cargo build --release
   b. Optimize binary
   c. Generate executable
6. Validate generated code
7. Package for deployment
```

**Analysis**:
- Agent Queue: **Queue-centric** - workflows are managed by the queue
- Transpiler: **Workflow-centric** - execution is the primary concern
- **Overlap**: Both execute workflows, but semantics are different
- **Recommendation**: Agent Queue's integration point (IN-01) should call transpiler's execution engine directly when ready to replace stubbed executor

---

### 3. State Management (Different Semantics)

#### Agent Queue (QE-02: State Machine)
**States**: NEW → QUEUED → SCHEDULED → LEASED → RUNNING → DONE/FAILED/DLQ/CANCELED/EXPIRED

**Transitions**:
```rust
QUEUED → LEASED     // Scheduler claims task for agent
LEASED → RUNNING    // Agent starts execution
RUNNING → DONE       // Success
RUNNING → FAILED     // Failure (triggers retry)
FAILED → DLQ        // Max retries exhausted
```

**Persistence**: SQLite (ACID, WAL mode, single-writer)

#### Transpiler (State Management)
**State Types**:
- Execution context (model loaded, variables, step results)
- Checkpoints (snapshot at intervals)
- Versioned state (delta updates for efficiency)
- Resume capability (restore from any checkpoint)

**Storage**:
- Checkpoints: `/workspace/checkpoints/`
- State: `/workspace/state/execution_state.json`
- Logs: `/workspace/logs/workflow.log`
- Metrics: `/workspace/metrics/run_{timestamp}.json`

**Analysis**:
- Agent Queue: **Queue state machine** - tracks where a workflow is in the queue
- Transpiler: **Execution state management** - tracks what's happening during workflow execution
- **Overlap**: Both persist state, but different purposes
- **Recommendation**: These are complementary - queue tracks orchestration state, transpiler tracks execution state

---

### 4. Retry and Error Handling (Different Approaches)

#### Agent Queue (QE-09, QE-10: Queue-Level Retry)
**Configuration**:
```rust
RetryConfig {
    max_attempts: 3,
    base_delay_ms: 1000,
    max_delay_ms: 30000,
    backoff_multiplier: 2.0,
    jitter_factor: 0.2,
}
```

**Retry Behavior**:
- Queue-level retry (entire workflow)
- Exponential backoff: 1s, 2s, 4s, 8s
- 10% random jitter prevents thundering herd
- Failed workflows → DLQ after max attempts
- Manual retry from DLQ

#### Transpiler (Step-Level Retry + Validation Loops)
**Step Retry**:
```yaml
retry:
  default:
    max_attempts: 3
    backoff_strategy: exponential | linear | fixed
    on_failure: escalate_model
```

**Validation Loop**:
```yaml
validation_loop:
  type: validation | exact | tolerance
  max_iterations: 5
  stop_conditions:
    - validation.score >= 0.9
    - validation.errors == []
```

**Analysis**:
- Agent Queue: **Coarse-grained retry** - re-entire workflow
- Transpiler: **Fine-grained retry** - retry individual steps or validation loops
- **Overlap**: Both have retry semantics, but different granularity
- **Recommendation**: Agent Queue's retry logic is simpler for queue orchestration. Transpiler's validation loops are more sophisticated - defer to transpiler for complex validation scenarios.

---

### 5. Tool Systems (Different Scopes)

#### Agent Queue (YA-06: Tool Framework - MVP)
**MVP Tools**:
```rust
ShellTool {
    enabled: true,
    timeout: 60s,
    allowed_commands: [cargo, rustc, git],  // Constrained in MVP
}

FileReadTool {
    enabled: true,
    path_restricted: workspace,  // No absolute paths
}

FileWriteTool {
    enabled: true,
    path_restricted: workspace,
}
```

**Permission Model**:
- Allowlist per workflow (explicit opt-in)
- Default deny (no tools unless allowed)
- Network blocked in MVP

#### Transpiler (Built-in Tools + Custom)
**Built-in Tools**:
```yaml
tools:
  file:
    read:
      enabled: true
      require_confirmation: false
      allowed_paths: [./src, ./config]
    write:
      enabled: true
      require_confirmation: true
      backup_existing: true
    delete:
      enabled: true
      require_confirmation: true
      allowed_paths: [./temp, ./cache]

  web:
    fetch:
      enabled: true
      timeout_seconds: 30
      respect_robots_txt: true
    scrape:
      enabled: true
      parse_html: true
      extract_structure: true

  shell:
    exec:
      enabled: true
      require_confirmation: true
      timeout_seconds: 60
      allowed_commands: [cargo, rustc, git]
```

**Custom Tool Hooks**:
```rust
pub trait Tool: Send + Sync {
    async fn execute(&self, ctx: ToolContext) -> Result<ToolOutput, ToolError>;
}
```

**Analysis**:
- Agent Queue: **Constrained sandbox** - limited tool set, default deny
- Transpiler: **Richer toolset** - file, web, shell + custom hooks
- **Overlap**: Both have shell, file_read, file_write
- **Recommendation**: Agent Queue should use transpiler's tool framework when available. Defer tool implementation to transpiler (e.g., network tools, browser automation, search).

---

### 6. Scheduling (Agent Queue's Core Strength)

#### Agent Queue (25 Queue Levels + Meta-Scheduler)

**Core Queues (MVP: QE-05)**:
1. **ASAP (QL-001, QL-002)**: Highest urgency
2. **Whenever (QL-003, QL-004)**: Background processing
3. **Scheduled-Once (QL-005)**: Time-gated execution
4. **Repeat-Cron (QL-007)**: Recurring automation

**Advanced Queues (Unique Requirements)**:
5. **Scheduled-Window (QL-006)**: Maintenance windows
6. **Repeat-FixedDelay (QL-008)**: Fixed cadence
7. **Repeat-FixedRate (QL-009)**: Fixed frequency
8. **ASAP-Blocking (QL-003)**: Critical path blocking
9. **Rate-Limited (QL-011)**: Throughput caps
10. **Quota-Governed (QL-012)**: Multi-tenant quotas
11. **Human-Gated (QL-013)**: Approval required
12. **Manual-Override (QL-014)**: Operator control
13. **DLQ (QL-015)**: Triage-only, no auto-execution
14. **Sandbox (QL-016)**: Security isolation
15. **Capability-Pinned (QL-017)**: Skill matching
16. **Cost-Aware (QL-018)**: Optimization
17. **Experiment (QL-020)**: Best-effort, can drop
18. **Meta-Scheduler (QL-021)**: Coordinates across queues
19. **Deadline-Driven (QL-035)**: EDF scheduling
20. **Fair-Share (QL-022)**: Balanced allocation
21. **Priority+Aging (QL-012)**: Anti-starvation
22. **WRR (QL-034)**: Weighted round-robin
23. **DRR (QL-034)**: Deficit round-robin
24. **EDF (QL-035)**: Earliest deadline first
25. **Token-Bucket (QL-043)**: Rate limiting

**Scheduling Strategies**:
- **Priority + Aging**: Low-priority tasks get boost over time
- **Deadline Escalation**: Tasks approach deadline get automatic priority boost
- **Multi-tenant Fair Share**: Balanced resource allocation per tenant
- **Work Stealing**: Idle workers steal tasks from busy queues
- **Spillover Queues**: Overflow handling to alternate queues

#### Transpiler (Execution Modes, No Scheduling)

**Execution Modes**:
```yaml
execution:
  mode: parallel | serial | hybrid
```

**Parallel Mode**:
- Execute steps/agents simultaneously
- Higher throughput
- Higher memory usage

**Serial Mode**:
- Execute steps/agents sequentially
- Lower throughput
- Lower memory usage

**Hybrid Mode**:
- Mixed strategy for complex workflows

**Analysis**:
- Agent Queue: **Sophisticated scheduling** - 25 queue levels, advanced algorithms
- Transpiler: **Simple execution modes** - parallel/serial/hybrid
- **Overlap**: None - these are fundamentally different concerns
- **Recommendation**: Agent Queue owns all scheduling logic. Transpiler's "parallel/serial/hybrid" is about execution concurrency, not queue orchestration.

---

### 7. Multi-Tenancy (Agent Queue Only)

#### Agent Queue (Q-018: Multi-Tenancy Full Support)

**Features**:
```rust
Tenant {
    id: String,
    quota_profile: QuotaProfile,
    isolation_level: Isolation,
}

QuotaProfile {
    compute_quota: Option<u32>,
    storage_quota: Option<u64>,
    concurrency_limit: Option<u32>,
}

Isolation {
    Namespace: Shared | Dedicated,
    Database: Shared | Tenant-Specific,
    Runtime: Shared | Dedicated,
}
```

**Capabilities**:
- Tenant isolation
- Per-tenant quotas
- Fair share allocation
- Tenant-specific queues
- Audit logs per tenant
- RBAC for access control

#### Transpiler (Single-Process, No Multi-Tenancy)

**Architecture**:
- Single-process execution
- Single-user focus
- No tenant isolation

**Analysis**:
- Agent Queue: **Enterprise-grade multi-tenancy**
- Transpiler: **Single-process, single-user**
- **Overlap**: None
- **Recommendation**: This is a clear agent-queue differentiator. Keep multi-tenancy in agent-queue.

---

### 8. Observability (Complementary Approaches)

#### Agent Queue (QE-14: Structured Logging + QE-25: Audit Log)

**Logging Structure**:
```json
{
  "trace_id": "uuid-here",
  "run_id": "run-xxx",
  "task_id": "task-xxx",
  "action": "state_transition",
  "from": "queued",
  "to": "leased",
  "timestamp": "2025-04-06T15:30:00Z",
  "duration_ms": 45,
  "actor": "scheduler|agent|cli"
}
```

**Audit Log**:
```sql
CREATE TABLE audit_log (
    id INTEGER PRIMARY KEY,
    ts TEXT NOT NULL,
    actor TEXT,
    action TEXT,
    entity_type TEXT,
    entity_id TEXT,
    prev_state TEXT,
    new_state TEXT,
    diff_json TEXT,
    reason TEXT
);
```

#### Transpiler (9-Level Hierarchical Logging)

**Logging Levels**:
```yaml
logging:
  global:
    level: debug | info | error
    detail: high | medium | low

  workflow_execution:
    level: info
    detail: medium

  step_execution:
    level: debug
    detail: high

  agent_execution:
    level: info
    detail: medium

  tool_execution:
    level: debug
    detail: very_high

  file_operations:
    level: debug
    detail: very_high

  web_operations:
    level: info
    detail: high

  state_management:
    level: debug
    detail: low

  performance_metrics:
    level: info
    detail: medium
```

**Output Types**:
- `chat`: Human-readable conversation
- `log`: Structured machine-readable
- `stateless_direct_io`: Direct I/O without context

**Metrics Collected**:
```json
{
  "total_execution_duration_seconds": 123,
  "agent_execution_count": 5,
  "validation_loop_iterations": 3,
  "retry_count": 2,
  "final_validation_score": 0.92,
  "code_quality_score": 0.87,
  "memory_usage_mb": 1024,
  "cpu_usage_percent": 67,
}
```

**Analysis**:
- Agent Queue: **Orchestration-focused** - tracks queue state, transitions, audit trail
- Transpiler: **Execution-focused** - tracks step execution, tool calls, metrics, validation iterations
- **Overlap**: Both have structured logging, but different focus
- **Recommendation**: Agent Queue should integrate with transpiler's metrics for richer observability. Defer detailed execution metrics to transpiler.

---

### 9. LLM Integration (Different Levels of Support)

#### Agent Queue (YA-03, YA-04: Single Provider Trait)

**MVP LLM Provider**:
```rust
pub trait LlmProvider: Send + Sync {
    async fn generate(&self, request: LlmRequest) -> Result<LlmResponse, LlmError>;
    async fn stream(&self, request: LlmRequest) -> Result<Pin<Box<dyn Stream<Item = Result<String, LlmError>>>, LlmError>;
    fn count_tokens(&self, messages: &[LlmMessage]) -> usize;
}

pub struct LlamaCppProvider {
    // llama.cpp-based implementation
}
```

**Usage**:
- Queue engine holds provider reference
- Agent executor calls provider for each step

#### Transpiler (Multi-Model Support with Auto-Routing)

**Models**:
```yaml
models:
  primary:
    provider: lmstudio | ollama | llama.cpp
    model: "llama-3.2-3b-instruct"

  analyzer:
    provider: lmstudio
    model: "llama-3.2-3b-instruct"

  validator:
    provider: ollama
    model: "llama3.2"

  embedder:
    provider: jinaai
    model: "ReaderLM-v2"
```

**Auto-Routing**:
- Model selection based on task type
- Fallback on failure (escalate to alternative model)
- Backend support: Vulkan, CUDA, CPU, Metal

**Analysis**:
- Agent Queue: **Single provider abstraction** - one trait, one implementation (MVP)
- Transpiler: **Multi-model auto-routing** - 4+ providers, intelligent selection
- **Overlap**: Agent Queue's trait allows transpiler integration in future, but current stubbed executor doesn't use it
- **Recommendation**: When replacing stubbed executor, use transpiler's multi-model support directly. Defer LLM integration to transpiler.

---

### 10. Hooks and Event-Driven Workflows (Agent Queue Only)

#### Agent Queue (QL-026: Hook-Triggered Queue Category)

**Hook Types**:
- Webhooks (HTTP POST/GET)
- File watchers (inotify)
- Git hooks (commit/push/merge)
- Message queues (NATS/Redis/Kafka)
- Timers (cron-style internal)
- Email triggers (IMAP/SMTP)
- Database triggers (CDC)
- Manual triggers (CLI/API)
- API callbacks (REST endpoints)

**Hook Configuration**:
```yaml
hooks:
  - id: hook-github-pr
    event_type: webhook
    match:
      path: /hooks/github
      method: POST
      headers:
        X-GitHub-Event: pull_request
    workflow_id: pr-review-workflow
    response_queue: asap
    config:
      model_override: llama-3.2-3b
      agent_pool: code-review
      timeout: 300s
      dedupe_window: 300s
    retry_policy:
      max: 3
      backoff: exponential
```

**Hook Lifecycle**:
1. Event arrives (webhook, file watcher, git hook, message queue, timer)
2. Dedupe within window (300s default)
3. Match event type to registered hooks
4. Execute hook handler
5. Trigger workflow enqueue
6. Route to appropriate queue based on hook config
7. Send response (if configured)
8. Audit log

#### Transpiler (No Hooks)

**Architecture**:
- Manual workflow execution or code generation
- No event-driven workflow triggering
- Direct invocation via CLI

**Analysis**:
- Agent Queue: **Event-driven workflow system**
- Transpiler: **Direct invocation system**
- **Overlap**: None
- **Recommendation**: This is a clear agent-queue differentiator. Keep hook system in agent-queue.

---

## Critical Overlaps and Redundancies

### 1. YAML Schema (Shared Format)
- **Status**: Both use YAML for workflow definition
- **Issue**: Schemas are **not compatible**
- **Agent Queue Schema**:
  ```yaml
  schema_version: "0.1.0"
  name: "workflow"
  queue:
    category: asap | whenever | scheduled | cron
    priority: critical | high | normal | low
  model:
    provider: llama-cpp
    model_path: ./models/...
  steps:
    - id: step-1
      prompt: { ... }
  ```
- **Transpiler Schema**:
  ```yaml
  workflow_id: ai_code_refactor
  models:
    primary:
      provider: lmstudio
  execution:
    mode: serial | parallel
  agentic_workflow:
    - step: analyze
      validation_loop:
        type: validation
  ```

**Resolution Required**: **Schema compatibility layer** needed
- Agent Queue can transpile to transpiler's schema format
- Transpiler can parse agent-queue's schema format
- OR: Agent Queue adopts transpiler's richer schema

### 2. File Operations (Shared Tools)
- **Status**: Both have file_read and file_write
- **Issue**: Agent Queue's sandbox is more restrictive
- **Agent Queue**:
  - Path restricted to workspace
  - No absolute paths
  - Shell timeout: 60s
  - No network access (MVP)
- **Transpiler**:
  - Require confirmation for writes (UX safety)
  - Backup before overwrite
  - Delete with confirmation
  - Search capability
  - Archive support

**Recommendation**: Defer file operations to transpiler's richer tool framework when available. Use Agent Queue's simple sandbox only for basic operations.

### 3. Logging (Different Approaches)
- **Status**: Both use structured JSON logging
- **Agent Queue**: Queue-centric (state transitions, enqueue, dequeue)
- **Transpiler**: Execution-centric (step calls, tool calls, model loading)

**Recommendation**: Integrate transpiler's 9-level hierarchical logging for richer execution visibility while keeping Agent Queue's queue-focused logging.

---

## What Agent Queue Should Do (Unique Capabilities)

### ✅ Keep (These are Agent Queue's Core Strengths)

1. **Queue Orchestration** (All 25 queue levels, meta-scheduler)
   - Rationale: Sophisticated scheduling is unique value
   - Implementation: Already designed, stubbed executor doesn't block this

2. **Multi-Tenancy** (Full tenant isolation, quotas, fair share)
   - Rationale: Enterprise requirement not in transpiler
   - Implementation: Q-018 framework

3. **Hooks** (Event-driven workflows, QL-026)
   - Rationale: Enables automated workflows
   - Implementation: Webhook registry, event matching, routing

4. **State Machine** (Queue lifecycle: NEW → QUEUED → LEASED → RUNNING → DONE/FAILED/DLQ)
   - Rationale: Core to queue semantics
   - Implementation: Already in Phase 2 (02-state-machine.md)

5. **Backpressure & Admission Control** (QE-26, QE-28)
   - Rationale: System stability under load
   - Implementation: Queue depth limits, pending state

6. **Audit Trail** (QE-25)
   - Rationale: Compliance and debugging
   - Implementation: Append-only SQLite table

7. **DLQ** (QE-10)
   - Rationale: Triage and recovery for failed workflows
   - Implementation: Separate DLQ table, manual retry

8. **Idempotency** (QE-11)
   - Rationale: Safe retries and duplicate prevention
   - Implementation: Dedupe window on idempotency key

9. **Lease Management** (QE-08)
   - Rationale: Prevents double-execution, crash detection
   - Implementation: Heartbeat every 15s, 60s TTL

10. **Retry with Backoff** (QE-09)
   - Rationale: Handles transient failures automatically
   - Implementation: Exponential backoff with jitter

11. **CLI** (QE-15 through QE-20)
   - Rationale: Primary interface for users
   - Implementation: Clap-based CLI commands

12. **Structured Logging** (QE-14)
   - Rationale: Complete observability
   - Implementation: JSON logs with correlation IDs

13. **Due-Time Gates** (QE-27)
   - Rationale: Time-based scheduling
   - Implementation: Scheduled tasks wait until due_ts

14. **Queue Routing** (QE-22)
   - Rationale: Automatic queue assignment
   - Implementation: Route based on YAML metadata

15. **Scheduled Windows** (QL-006)
   - Rationale: Maintenance operations
   - Implementation: Time-gated task eligibility

16. **Spillover Queues** (QL-023)
   - Rationale: Overflow handling
   - Implementation: Redirect to alternate queue when full

17. **Rate Limiting** (QL-011)
   - Rationale: Throughput control
   - Implementation: Per-tenant token or operation limits

18. **Quotas** (QL-012)
   - Rationale: Multi-tenant resource governance
   - Implementation: Per-tenant compute/storage/concurrency limits

19. **Work Stealing** (QL-037)
   - Rationale: Resource utilization
   - Implementation: Cross-pool task redistribution

20. **Deadline-Driven Scheduling** (QL-035)
   - Rationale: SLA compliance
   - Implementation: EDF algorithm

21. **Fair Share** (QL-022)
   - Rationale: Multi-tenant balance
   - Implementation: Weighted allocation with min_share

22. **Priority Inversion Guard** (QL-012)
   - Rationale: Correctness guarantee
   - Implementation: Temporary boost for blocking deps

23. **Token Bucket Rate Limiting** (QL-043)
   - Rationale: API rate limiting
   - Implementation: Token bucket refill strategy

24. **Cost-Aware Scheduling** (QL-018)
   - Rationale: Cost optimization
   - Implementation: Priority based on estimated cost

---

### ❌ Remove or Simplify (Defer to Transpiler)

1. **Remove from MVP**:
   - Parallel step execution (YA-05) - replace with transpiler execution modes
   - Tool invocation framework (YA-06) - use transpiler's built-in tools
   - Prompt template rendering (YA-08) - use transpiler's prompt handling
   - Step timeout (YA-09) - use transpiler's timeout enforcement
   - Step retry (YA-10) - use transpiler's retry mechanism
   - Context window management (YA-11) - use transpiler's budgeting
   - Output parsing (YA-12) - use transpiler's output extraction
   - Artifact collection (YA-16) - use transpiler's artifact system
   - Metrics (YA-17) - use transpiler's metrics
   - Validation mode (YA-18) - use transpiler's dry-run

2. **Replace Stubbed Executor**:
   - Current stubbed executor (mock LLM, mock tools) is for MVP testing
   - Replace with calls to transpiler's execution engine (real LLM calls)
   - Keep Agent Queue's orchestration, integrate transpiler for execution

3. **Simplify Schema**:
   - Adopt transpiler's richer YAML schema
   - OR create compatibility layer to translate between schemas

4. **Remove Duplicate Requirements**:
   - QL-001 through QL-025 (25 queue levels) - KEEP (core differentiator)
   - Any queue requirement that duplicates transpiler execution features - REMOVE

5. **Clarify Integration Points**:
   - IN-01 (Queue → Agent): Should call transpiler execution engine directly
   - IN-02 (Agent → State): Should report transpiler execution state
   - IN-04 (Agent → Workflow Input): Pass transpiler-compatible YAML
   - IN-05 (Agent → Artifacts): Store transpiler-generated artifacts

---

## Integration Strategy

### Short-Term: MVP (Current Plan)

1. **Keep Stubbed Executor** for queue engine testing
2. **Define clear integration point** for future transpiler replacement:
   ```rust
   pub trait WorkflowExecutor {
       async fn execute(&self, run_id: &str, workflow: &Workflow) -> Result<ExecutionResult>;
   }
   
   pub struct StubbedExecutor { /* Current implementation */ }
   pub struct TranspilerExecutor { /* Future implementation */ }
   ```
3. **Maintain schema compatibility**: Document expected workflow format for transpiler

### Long-Term: Production

1. **Replace stubbed executor** with transpiler integration
2. **Use transpiler's features**:
   - Multi-model support
   - Rich tool framework
   - Code generation mode
   - Validation loops
   - Hierarchical logging
   - Metrics collection
3. **Keep agent-queue as orchestrator**: Queue engine schedules, transpiler executes
4. **Benefits**:
   - Best-of-both: Sophisticated scheduling + Rich execution
   - Separation of concerns: Queue = orchestration, Transpiler = execution
   - Easier evolution: Can upgrade transpiler without affecting queue logic

---

## Updated Requirements Summary

### Agent Queue Unique Capabilities (After Removing Overlaps)

| Category | Count | Examples |
|----------|-------|----------|
| **Queue Orchestration** | 25+ | All queue levels, meta-scheduler, priority aging, work stealing |
| **Multi-Tenancy** | Full | Tenant isolation, quotas, fair share, RBAC |
| **Hooks & Events** | Yes | Webhooks, git hooks, file watchers, message queues |
| **State Machine** | Yes | Queue lifecycle (10 states) |
| **Backpressure** | Yes | Queue depth limits, admission control |
| **DLQ** | Yes | Triage, manual retry |
| **Audit Log** | Yes | State transition history |
| **Idempotency** | Yes | Duplicate prevention |
| **Lease Management** | Yes | Heartbeat, expiry, reclaim |
| **Retry Logic** | Yes | Exponential backoff |
| **CLI** | Yes | Enqueue, list, inspect, cancel, retry, drain |
| **Due-Time Gates** | Yes | Time-based scheduling |
| **Queue Routing** | Yes | Automatic queue assignment |

### Delegated to Transpiler (When Available)

| Category | Status | Rationale |
|----------|--------|----------|
| **Workflow Execution** | Defer | Transpiler has richer execution modes, multi-model support, validation loops |
| **Tool Invocation** | Defer | Transpiler has built-in tools + custom hooks |
| **LLM Integration** | Keep for now | Single provider sufficient for MVP, but integrate transpiler later |
| **Prompt Rendering** | Defer | Transpiler handles complex prompt workflows |
| **Context Management** | Defer | Transpiler has sophisticated budgeting |
| **Output Parsing** | Defer | Transpiler has structured output handling |
| **Artifact Collection** | Simplify | Store transpiler-generated artifacts directly |
| **Metrics** | Defer | Use transpiler's richer metrics |
| **Validation Mode** | Keep | Simple validation is useful for MVP |

---

## Updated MVP Definition

### Revised Scope

**MVP Focus**: Queue orchestration and workflow coordination
**Execution**: Stubbed agent executor (sleep-based mocks) for testing
**Integration**: Replace with transpiler execution engine in post-MVP

### Timeline Adjustments

- **Phase 3 (Stubbed Agent Executor)**: Keep as-is for MVP testing
- **Phase 4 (CLI + Integration)**: Add transpiler integration option
- **Phase 5 (Testing + Verification)**: Add integration tests
- **Post-MVP**: Replace stubbed executor with real transpiler integration

---

## Next Steps

### Immediate Actions

1. ✅ **Create this analysis document** - Done
2. ✅ **Commit analysis** - Will commit after saving
3. ✅ **Update MVP implementation plan** - Update to focus on queue-unique features
4. ⏳ **Update requirements documents** - Remove overlapping requirements from agent-queue
5. ⏳ **Create integration specification** - Define interface for transpiler integration

### Questions for Decision

1. **Schema Strategy**: Should agent-queue adopt transpiler's schema, or should transpiler support agent-queue's simpler schema?
2. **Integration Point**: Should IN-01 (Queue → Agent) call transpiler CLI or use API?
3. **Tool Strategy**: Should agent-queue use transpiler's tool framework, or keep simple sandbox?
4. **Migration Path**: Should agent-queue call transpiler's code generation mode and compile workflows?

### Recommendations

1. **Keep stubbed executor for MVP testing** - Don't remove it, use it as integration test
2. **Define clear integration contract** - Interface for swapping executors
3. **Document transpiler dependencies** - What agent-queue needs from transpiler
4. **Plan phased migration** - Queue first, then integrate transpiler for execution
5. **Focus documentation** - Emphasize agent-queue as orchestrator, not execution engine

---

## Conclusion

**Agent Queue is fundamentally a queue orchestration system** that coordinates workflow execution. The yaml-to-rust-agentsdk transpiler is a workflow execution engine. These are complementary systems.

**Key Takeaway**: Agent Queue should focus on what it does uniquely (queue orchestration, multi-tenancy, advanced scheduling, hooks) and delegate execution concerns to transpiler.

**Current MVP Plan**: Valid - stubbed executor is appropriate for testing queue engine without real LLM dependencies. Plan to integrate real transpiler execution engine in post-MVP.

---

**Document Status**: Draft - Ready for review and finalization
