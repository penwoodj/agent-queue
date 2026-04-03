# Agent Queue System + YAML-to-Rust-Agentsdk — MVP Definition

## Document Purpose

This document defines the **Minimum Viable Product (MVP)** for the combined Agent Queue workflow engine and YAML-to-Rust-Agentsdk transpiler. It answers: *What is the smallest set of features that makes this system fully operational and genuinely useful?*

The MVP is not a prototype or demo — it's a production-grade foundation that a single developer or small team can run locally to automate real LLM-agent workflows with queue-based orchestration.

---

## Table of Contents

1. [MVP Philosophy & Guiding Principles](#mvp-philosophy--guiding-principles)
2. [MVP Scope Decision Framework](#mvp-scope-decision-framework)
3. [MVP Feature Matrix](#mvp-feature-matrix)
4. [MVP Architecture](#mvp-architecture)
5. [Queue Engine MVP](#queue-engine-mvp)
6. [YAML-to-Rust-Agentsdk MVP](#yaml-to-rust-agentsdk-mvp)
7. [Integration Layer MVP](#integration-layer-mvp)
8. [MVP User Stories](#mvp-user-stories)
9. [MVP Requirements Traceability](#mvp-requirements-traceability)
10. [Explicitly Out of MVP](#explicitly-out-of-mvp)
11. [MVP Technical Specification](#mvp-technical-specification)
12. [MVP Data Model](#mvp-data-model)
13. [MVP CLI Reference](#mvp-cli-reference)
14. [MVP YAML Schema](#mvp-yaml-schema)
15. [MVP Implementation Phases](#mvp-implementation-phases)
16. [MVP Success Criteria](#mvp-success-criteria)
17. [MVP Risk Assessment](#mvp-risk-assessment)
18. [Post-MVP Evolution Roadmap](#post-mvp-evolution-roadmap)
19. [MVP Effort Estimation](#mvp-effort-estimation)

---

## MVP Philosophy & Guiding Principles

### What Makes This MVP Useful

The system becomes useful when a developer can:

1. **Write a YAML workflow** defining multi-step agent tasks with tools and prompts
2. **Enqueue that workflow** into a queue with priority and scheduling semantics
3. **Watch it execute** with structured logs, state transitions, and artifact output
4. **Handle failures** automatically via retries and manually via CLI inspection/requeue
5. **Schedule recurring work** via cron expressions for periodic automation

If all five of these work reliably, the system is useful. Everything else is optimization.

### Guiding Principles

| Principle | Description | Implication |
|-----------|-------------|-------------|
| **Runs Locally First** | Single-machine, single-process, no cloud dependencies | SQLite, single Tokio runtime, no distributed consensus |
| **YAML-Driven** | All configuration and workflow definition via YAML files | Schema validation is critical path |
| **LLM-Agnostic MVP** | One LLM provider works, but the abstraction allows more | llama-cpp-2 for MVP, provider trait for future |
| **Observable by Default** | Every state transition, every tool call, every error is logged | Structured JSON logs with correlation IDs |
| **Fail Gracefully** | No silent failures, no corrupted state, no lost work | DLQ, idempotency, ACID transactions |
| **CLI-First UX** | All operations via CLI, no GUI for MVP | Clap + Console, hierarchical commands |
| **Extend Without Rewrite** | MVP architecture supports post-MVP features without structural changes | Trait-based providers, pluggable queue strategies |

---

## MVP Scope Decision Framework

### The "Useful Test"

For each requirement, we ask: *Is the system useful WITHOUT this?*

| Answer | Scope Decision | Rationale |
|--------|---------------|-----------|
| **No — system breaks** | **MVP (P0)** | Must-have for basic operation |
| **No — major use case blocked** | **MVP (P0)** | One of the 5 core workflows fails |
| **Degraded but usable** | **Post-MVP (P1)** | Can work around it temporarily |
| **Nice to have** | **Post-MVP (P2+)** | Optimization, not foundation |

### The "Dependency Test"

For each MVP feature, we ask: *What does this depend on?*

If a feature has no dependencies and passes the Useful Test, it's P0.
If a feature depends on another P0 feature, it cascades into MVP.

---

## MVP Feature Matrix

### Queue Engine MVP (30 Features)

| ID | Feature | MVP Priority | Rationale |
|----|---------|-------------|-----------|
| QE-01 | **Entity definitions** (Workflow, Run, Task, Step, Agent, Queue, Lease) | P0 | Foundation — nothing works without entities |
| QE-02 | **State machine** (queued → leased → running → done/failed/canceled/dlq) | P0 | Core lifecycle — every task follows this |
| QE-03 | **SQLite persistence** (WAL mode, ACID, single-writer) | P0 | Survives restarts — durability is non-negotiable |
| QE-04 | **YAML validation** (strict parsing, clear errors, deny_unknown_fields) | P0 | Invalid YAML must fail fast with helpful message |
| QE-05 | **3 queue categories** (ASAP, Whenever, Scheduled-Once) | P0 | Covers urgent, background, and timed work |
| QE-06 | **Priority bands** (critical:10, high:5, normal:3, low:1) | P0 | Urgent tasks must run before background |
| QE-07 | **FIFO tie-breaking** (enqueue_ts, task_id for same priority) | P0 | Deterministic ordering prevents starvation |
| QE-08 | **Lease + heartbeat** (60s TTL, 15s heartbeat interval) | P0 | Prevents double-execution, detects crashes |
| QE-09 | **Retry with backoff** (exponential, jitter, max 3, classifiable errors) | P0 | Transient failures must be handled automatically |
| QE-10 | **Dead-letter queue** (exhausted retries → DLQ, triage, requeue) | P0 | Failed tasks must be inspectable and recoverable |
| QE-11 | **Idempotency key** (dedupe_window, same key = same effect) | P0 | Safe retries require idempotency guarantees |
| QE-12 | **Cancellation** (run/task level, propagated to leased tasks) | P0 | Users must be able to stop work |
| QE-13 | **Basic concurrency limits** (global max_in_flight) | P0 | Prevents system overload |
| QE-14 | **Structured logging** (trace_id, run_id, task_id, JSON format) | P0 | Debuggability — every event traceable |
| QE-15 | **CLI: enqueue** (workflow YAML path, queue, priority, tags) | P0 | Primary interface for submitting work |
| QE-16 | **CLI: list** (filter by queue, state, priority, limit) | P0 | Users must see what's in the system |
| QE-17 | **CLI: inspect** (task/run details, state history, error info) | P0 | Debugging failed tasks requires visibility |
| QE-18 | **CLI: cancel** (cancel running or queued tasks) | P0 | Stop unwanted work |
| QE-19 | **CLI: retry** (requeue from DLQ or failed state) | P0 | Recover from failures |
| QE-20 | **CLI: drain** (wait for all queued tasks to complete) | P0 | Graceful shutdown support |
| QE-21 | **Meta-scheduler** (priority-based queue selection across queues) | P0 | Multiple queues need coordinated scheduling |
| QE-22 | **Queue routing** (route tasks to queue by YAML metadata) | P0 | Automatic queue assignment from workflow definition |
| QE-23 | **Repeat-Cron** (cron expression, timezone, bounded catchup) | P0 | Most common use case for automation |
| QE-24 | **Artifact storage** (file-based, per-step outputs, retention days) | P0 | Agent output must be persisted |
| QE-25 | **Audit log** (append-only, state transitions, who/what/when) | P0 | Compliance and debugging |
| QE-26 | **Backpressure** (queue depth limit, reject when full) | P0 | Prevents unbounded memory growth |
| QE-27 | **Due-time gate** (scheduled tasks not eligible before due_ts) | P0 | Time-based scheduling requires this |
| QE-28 | **Admission control** (basic: accept/reject on enqueue) | P0 | System stability under load |
| QE-29 | **Step-level error classification** (retryable vs non-retryable) | P0 | Retry logic depends on error classification |
| QE-30 | **Schema version header** (schema_version in YAML) | P0 | Forward compatibility from day one |

### YAML-to-Rust-Agentsdk MVP (18 Features)

| ID | Feature | MVP Priority | Rationale |
|----|---------|-------------|-----------|
| YA-01 | **YAML schema for workflows** (steps, tools, prompts, outputs, model config) | P0 | The input format — without this, nothing runs |
| YA-02 | **Workflow IR** (typed intermediate representation from parsed YAML) | P0 | Internal representation for execution engine |
| YA-03 | **Single LLM provider** (llama.cpp via llama-cpp-2) | P0 | One provider that works is enough for MVP |
| YA-04 | **LLM provider trait** (trait LlmProvider with generate(), stream()) | P0 | Abstraction allows future providers without rewrite |
| YA-05 | **Step execution engine** (sequential, input/output passing) | P0 | Core execution — steps must run in order |
| YA-06 | **Tool invocation framework** (built-in: shell, file_read, file_write) | P0 | Agents need tools to be useful |
| YA-07 | **Tool permission model** (allowlist per workflow, default: deny) | P0 | Safety — uncontrolled tool access is dangerous |
| YA-08 | **Prompt template rendering** (variable substitution, system/user/assistant) | P0 | LLM interactions need structured prompts |
| YA-09 | **Step timeout** (configurable per-step, default 120s) | P0 | Prevents hung workflows |
| YA-10 | **Step retry** (per-step retry with configurable max) | P0 | Individual step failures shouldn't kill entire workflow |
| YA-11 | **Context window management** (token counting, truncation strategy) | P0 | Local models have limited context — must manage it |
| YA-12 | **Output parsing** (structured extraction from LLM responses) | P0 | Raw LLM output must be structured for step chaining |
| YA-13 | **Workflow configuration** (model selection, temperature, max_tokens) | P0 | Users need to control LLM behavior |
| YA-14 | **Environment variables** (secrets via env vars, not inline) | P0 | API keys and secrets must not be in YAML |
| YA-15 | **Error handling per step** (continue vs abort on error) | P0 | Workflow resilience |
| YA-16 | **Artifact collection** (gather step outputs into run artifacts) | P0 | Users need access to results |
| YA-17 | **Basic metrics** (step count, total duration, token usage) | P0 | Users need to understand cost and performance |
| YA-18 | **Deterministic validation mode** (dry-run, no LLM calls) | P0 | Validate YAML without spending compute |

### Integration MVP (7 Features)

| ID | Feature | MVP Priority | Rationale |
|----|---------|-------------|-----------|
| IN-01 | **Queue dispatches to agent** (queue engine calls agentsdk executor) | P0 | The bridge — queue must trigger execution |
| IN-02 | **Agent reports back** (completion/failure → state transition) | P0 | Queue must know when work is done |
| IN-03 | **Lease extension via heartbeat** (agent heartbeats during execution) | P0 | Long-running agent tasks must keep their lease alive |
| IN-04 | **Workflow input from queue** (queue provides workflow YAML path + params) | P0 | Queue must pass context to agent |
| IN-05 | **Artifact handoff** (agent writes artifacts → queue stores them) | P0 | Results must flow back to queue |
| IN-06 | **Error propagation** (agent error → queue retry/DLQ decision) | P0 | Agent failures must trigger queue error handling |
| IN-07 | **Single process architecture** (queue + agent in one Tokio runtime) | P0 | MVP simplicity — no inter-process communication |

---

## MVP Architecture

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    agent-queue (single process)          │
│                                                         │
│  ┌──────────┐    ┌──────────────┐    ┌───────────────┐  │
│  │   CLI    │───▶│ Queue Engine  │───▶│  Agent SDK    │  │
│  │  (Clap)  │◀───│  (Scheduler)  │◀───│ (Executor)    │  │
│  └──────────┘    └──────┬───────┘    └───────┬───────┘  │
│                         │                    │           │
│                    ┌────▼────┐         ┌─────▼─────┐    │
│                    │ SQLite  │         │ llama.cpp │    │
│                    │  (WAL)  │         │ (llama-   │    │
│                    │         │         │  cpp-2)   │    │
│                    └─────────┘         └───────────┘    │
│                                                         │
│  ┌──────────┐    ┌──────────────┐    ┌───────────────┐  │
│  │  Logs    │    │  Artifacts   │    │  Audit Trail  │  │
│  │ (tracing)│    │  (files)     │    │  (append-only)│  │
│  └──────────┘    └──────────────┘    └───────────────┘  │
└─────────────────────────────────────────────────────────┘
```

### Component Responsibilities

| Component | Responsibility | Tech |
|-----------|---------------|------|
| **CLI** | User interface: enqueue, list, inspect, cancel, retry, drain | Clap 4.x + console |
| **Queue Engine** | Scheduling, state management, lease management, routing | Custom (Tokio) |
| **Scheduler** | Meta-scheduling across queues, priority ordering, time gates | Custom (Tokio) |
| **Agent SDK** | YAML parsing, step execution, LLM calls, tool invocation | Custom (serde) |
| **LLM Provider** | Model loading, inference, streaming | llama-cpp-2 |
| **Storage** | Persistent state for all entities | SQLite (rusqlite, WAL) |
| **Logger** | Structured JSON logging with correlation IDs | tracing + tracing-subscriber |
| **Artifact Store** | File-based output storage | std::fs |
| **Audit Log** | Append-only state change records | SQLite table |

---

## Queue Engine MVP

### MVP Queue Categories

| Category | Ordering | Preemption | SLA | Typical Use |
|----------|----------|-----------|-----|-------------|
| **ASAP** | Priority desc → FIFO | No | Soft: target_latency | Urgent fixes, interactive requests |
| **Whenever** | FIFO (no priority) | No | None: eventual | Background processing, batch jobs |
| **Scheduled-Once** | EDF within due_ts gate | No | Hard: due_ts | "Run at 2am" one-shot tasks |
| **Repeat-Cron** | Per-slot FIFO | No | Configurable | Nightly backups, periodic syncs |

**Why these 4 (not 3):** I originally proposed 3 (ASAP, Whenever, Scheduled-Once) but Repeat-Cron is the single most common automation pattern. Excluding it would make the system useless for the primary use case of periodic agent workflows. The incremental cost is low — it's Scheduled-Once with a cron parser and deduplication.

### MVP State Machine

```
                    ┌──────────────────┐
                    │                  │
                    ▼                  │
┌──────┐     ┌──────────┐     ┌──────────┐     ┌───────────┐
│ NEW  │────▶│  QUEUED  │────▶│  LEASED  │────▶│  RUNNING  │
└──────┘     └────┬─────┘     └────┬─────┘     └─────┬─────┘
                  │                │                  │
                  │                │            ┌─────┴─────┐
                  │                │            ▼           ▼
                  │           ┌────▼────┐  ┌────────┐  ┌────────┐
                  │           │ EXPIRED │  │   DONE  │  │ FAILED │
                  │           └────┬────┘  └────────┘  └───┬────┘
                  │                │                        │
                  │                │                  ┌────┴────┐
                  │                │                  ▼         ▼
                  │           ┌────▼────┐        ┌──────┐  ┌────┐
                  │           │ REQUEUE │◀───────│RETRY │  │DLQ │
                  │           └─────────┘        └──────┘  └────┘
                  │
             ┌────┴────┐
             ▼         ▼
        ┌────────┐  ┌──────────┐
        │CANCELED│  │ SCHEDULED│
        └────────┘  └──────────┘
```

**MVP States (10):**
- `NEW` — Created, not yet validated
- `QUEUED` — Validated, waiting for scheduler
- `SCHEDULED` — Has due_ts, waiting for time gate
- `LEASED` — Claimed by agent, not yet executing
- `RUNNING` — Agent is actively executing
- `DONE` — Terminal: success
- `FAILED` — Non-terminal: may retry
- `DLQ` — Terminal: exhausted retries
- `CANCELED` — Terminal: user cancelled
- `EXPIRED` — Lease expired, reclaimable

### MVP Scheduler Logic

```rust
// Pseudocode for MVP scheduler tick
fn scheduler_tick(&mut self) {
    // 1. Check scheduled tasks - promote if due_ts <= now
    for task in self.tasks.where(state=SCHEDULED, due_ts <= now) {
        task.transition_to(QUEUED);
    }

    // 2. Check expired leases - reclaim to QUEUED
    for task in self.tasks.where(state=LEASED, lease_expiry <= now) {
        task.transition_to(QUEUED);
        emit_metric("lease_expired");
    }

    // 3. Reap DLQ candidates - move FAILED tasks with attempts >= max
    for task in self.tasks.where(state=FAILED, attempts >= max_retries) {
        task.transition_to(DLQ);
        emit_audit("task_to_dlq", reason: "exhausted_retries");
    }

    // 4. Apply backpressure - check global concurrency limit
    let in_flight = self.tasks.where(state=LEASED or RUNNING).count();
    if in_flight >= self.config.max_in_flight {
        return; // Skip scheduling this tick
    }

    // 5. Meta-scheduler: select next queue
    let queue = self.meta_scheduler.select_next(
        queues: [ASAP, Whenever, Scheduled, Cron],
        strategy: PriorityWeightedRoundRobin
    );

    // 6. Select next task from chosen queue
    let task = queue.next_candidate(
        ordering: effective_priority DESC, enqueue_ts ASC, task_id ASC
    );

    // 7. Lease the task
    if let Some(task) = task {
        task.lease(ttl: 60s, agent: self.agent_id);
        task.transition_to(LEASED);
    }
}
```

### MVP Lease Protocol

| Parameter | Default | Rationale |
|-----------|---------|-----------|
| `lease_ttl` | 60s | Long enough for LLM inference, short enough for crash detection |
| `heartbeat_interval` | 15s | 4 heartbeats per lease — good signal/noise ratio |
| `max_missed_heartbeats` | 2 | 30s grace before reclaim — tolerates brief network stalls |
| `reclaim_delay` | 5s | Delay before re-leasing to allow agent cleanup |

### MVP Retry Configuration

| Parameter | Default | Rationale |
|-----------|---------|-----------|
| `max_retries` | 3 | Industry standard — catches transient failures without masking systemic issues |
| `base_delay_ms` | 1000 | 1 second initial delay |
| `max_delay_ms` | 30000 | 30 second max delay |
| `backoff_multiplier` | 2 | Exponential backoff |
| `jitter_factor` | 0.2 | 20% randomization prevents thundering herd |

### Retry Delay Sequence (MVP defaults)

```
Attempt 1: 1000ms ± 200ms (jitter)
Attempt 2: 2000ms ± 400ms
Attempt 3: 4000ms ± 800ms
→ DLQ (exhausted)
```

---

## YAML-to-Rust-Agentsdk MVP

### MVP YAML Schema

```yaml
# schema_version is REQUIRED — enables future migrations
schema_version: "0.1.0"

# Workflow identity
name: my-workflow
description: "Does something useful with an LLM agent"
version: "1.0.0"

# Queue configuration (routes to agent-queue)
queue:
  category: whenever          # asap | whenever | scheduled | cron
  priority: normal            # critical | high | normal | low
  # Optional scheduling
  schedule:
    cron: "0 2 * * *"         # For cron category
    timezone: "America/Chicago"
    catchup: bounded(5)
  due_ts: "2025-04-03T02:00:00-05:00"  # For scheduled category

# Retry policy (overrides defaults)
retry:
  max_attempts: 3
  base_delay_ms: 1000
  max_delay_ms: 30000

# LLM configuration
model:
  provider: llama-cpp          # Only provider for MVP
  model_path: ./models/llama-3.2-3b-q4_k_m.gguf
  temperature: 0.7
  max_tokens: 4096
  context_budget: 4096         # Tokens reserved for this workflow

# Tool permissions (default: deny unless listed)
tools:
  allowed:
    - shell                   # Execute shell commands
    - file_read               # Read files
    - file_write              # Write files
  blocked:
    - network                 # No network access in MVP

# Environment (secrets via env vars, never inline)
env:
  - API_KEY                   # Resolved from process environment

# Steps (executed sequentially in MVP)
steps:
  - id: analyze-input
    description: "Analyze the input file"
    prompt:
      system: "You are a code analyst. Be concise."
      user: "Analyze this file and identify issues:\n{{input_file}}"
    model:
      temperature: 0.3         # Override per-step
    timeout_s: 120
    retry:
      max_attempts: 2
    on_error: abort            # abort | continue | retry

  - id: generate-fix
    description: "Generate fix for identified issues"
    prompt:
      system: "You are a senior developer."
      user: |
        Based on this analysis:
        {{steps.analyze-input.output}}
        
        Generate a fix for the identified issues in:
        {{input_file}}
    tools:
      - file_read
      - file_write
    timeout_s: 300

  - id: validate-fix
    description: "Validate the generated fix"
    prompt:
      user: |
        Validate this fix compiles and passes tests:
        {{steps.generate-fix.output}}
    on_error: continue         # Don't abort on validation failure
```

### MVP Tool Framework

| Tool | MVP Support | Safety | Description |
|------|------------|--------|-------------|
| `shell` | Yes | Sandboxed (timeout + no network) | Execute shell commands with timeout |
| `file_read` | Yes | Path restricted to workspace | Read file contents |
| `file_write` | Yes | Path restricted to workspace | Write files |
| `network` | No (blocked) | N/A | HTTP requests — deferred to post-MVP |
| `search` | No | N/A | Grep/ripgrep — nice to have |
| `browser` | No | N/A | Browser automation — far post-MVP |

### MVP LLM Provider Trait

```rust
#[async_trait]
pub trait LlmProvider: Send + Sync {
    /// Generate a completion (blocking until done)
    async fn generate(&self, request: LlmRequest) -> Result<LlmResponse, LlmError>;
    
    /// Stream a completion (for long-running generations)
    async fn stream(&self, request: LlmRequest) -> Result<Pin<Box<dyn Stream<Item = Result<String, LlmError>>>>, LlmError>;
    
    /// Count tokens in a message (for context window management)
    fn count_tokens(&self, messages: &[LlmMessage]) -> usize;
    
    /// Provider name (for logging)
    fn name(&self) -> &str;
}

pub struct LlmRequest {
    pub messages: Vec<LlmMessage>,
    pub temperature: f32,
    pub max_tokens: u32,
    pub stop: Vec<String>,
}

pub struct LlmResponse {
    pub content: String,
    pub token_count: TokenUsage,
    pub duration: Duration,
}

pub struct TokenUsage {
    pub prompt_tokens: u32,
    pub completion_tokens: u32,
    pub total_tokens: u32,
}
```

### MVP Context Window Management

Local models have limited context (typically 4K-8K tokens). The MVP must handle this:

| Strategy | MVP Support | Description |
|----------|------------|-------------|
| **Token counting** | Yes | Count tokens before sending to LLM |
| **Truncation** | Yes | Truncate oldest messages when over budget |
| **Summarization** | No | Summarize old context — post-MVP |
| **Sliding window** | No | Fixed window of recent messages — post-MVP |
| **Chunking** | No | Split large inputs into chunks — post-MVP |

```
Context Budget Allocation (per step):
├── System prompt: ~200 tokens (fixed)
├── Step prompt template: ~500 tokens (variable)
├── Input variables: variable (counted)
├── Previous step outputs: variable (counted, truncated first)
├── Tool results: variable (counted)
└── Reserved for response: min 512 tokens
```

---

## Integration Layer MVP

### Execution Flow

```
┌─────┐    ┌───────────┐    ┌──────────┐    ┌──────────┐    ┌──────┐
│ CLI │───▶│  Queue    │───▶│ Scheduler│───▶│  Agent   │───▶│ LLM  │
│     │    │  Engine   │    │  Tick    │    │ Executor │    │      │
└─────┘    └───────────┘    └──────────┘    └────┬─────┘    └──┬───┘
     ▲                                              │             │
     │                                              ▼             ▼
     │                                        ┌──────────┐  ┌──────────┐
     │                                        │  Tools   │  │ Artifacts│
     │                                        │(shell,   │  │ (files)  │
     │                                        │ fs r/w)  │  │          │
     │                                        └──────────┘  └──────────┘
     │
     │  ┌─────────────────────────────────────────┐
     └──│  State Machine Transitions (SQLite)      │
        │  QUEUED → LEASED → RUNNING → DONE/FAILED │
        │  FAILED → RETRY → DLQ                    │
        └─────────────────────────────────────────┘
```

### Single Process Integration

The MVP runs everything in one Tokio current-thread runtime:

```rust
#[tokio::main(flavor = "current_thread")]
async fn main() -> Result<()> {
    // Initialize components
    let db = SqliteStore::open("agent-queue.db")?;
    let queue_engine = QueueEngine::new(db.clone(), config.queue);
    let llm_provider = LlamaCppProvider::new(&config.model)?;
    let agent_executor = AgentExecutor::new(llm_provider, config.agent);
    let cli = Cli::parse();
    
    // Run scheduler loop
    let scheduler = queue_engine.scheduler_loop();
    
    // Handle CLI command
    match cli.command {
        Enqueue { workflow, queue, priority } => {
            let task = queue_engine.enqueue(workflow, queue, priority)?;
            println!("Enqueued task: {}", task.id);
        }
        Run => {
            // Start the agent execution loop
            agent_executor.run(queue_engine.clone()).await?;
        }
        // ... other commands
    }
    
    scheduler.await?;
    Ok(())
}
```

### Heartbeat During Execution

When the agent is executing a long-running LLM call or tool invocation, it must keep its lease alive:

```rust
async fn execute_step_with_heartbeat(
    &self,
    task: &mut Task,
    step: &Step,
    heartbeat_tx: mpsc::Sender<Heartbeat>,
) -> Result<StepOutput> {
    // Spawn heartbeat task
    let task_id = task.id.clone();
    let heartbeat_handle = tokio::spawn(async move {
        let mut interval = tokio::time::interval(Duration::from_secs(15));
        loop {
            interval.tick().await;
            let _ = heartbeat_tx.send(Heartbeat { task_id }).await;
        }
    });
    
    // Execute the step (may take minutes for LLM calls)
    let result = self.execute_step(step).await;
    
    // Stop heartbeat
    heartbeat_handle.abort();
    
    result
}
```

---

## MVP User Stories

### Story 1: "I want to run a one-shot agent workflow right now"

**As a** developer
**I want to** run a YAML-defined agent workflow immediately
**So that** I can automate a task with an LLM agent

**Flow:**
```bash
# 1. Write a workflow
cat > my-workflow.yaml << 'EOF'
schema_version: "0.1.0"
name: code-review
queue:
  category: asap
  priority: high
model:
  provider: llama-cpp
  model_path: ./models/llama-3.2-3b-q4_k_m.gguf
tools:
  allowed: [file_read, shell]
steps:
  - id: review
    prompt:
      system: "You are a senior code reviewer."
      user: "Review this PR diff:\n{{diff_content}}"
EOF

# 2. Enqueue it
agent-queue enqueue my-workflow.yaml --queue asap --priority high

# 3. Watch it execute
agent-queue list --state running
# → TASK-001  RUNNING  asap/high  code-review  12s ago

# 4. Check results
agent-queue inspect TASK-001
# → State: DONE
# → Steps: 1/1 completed
# → Duration: 45.2s
# → Tokens: 1,234 prompt + 567 completion
# → Artifacts: ./artifacts/TASK-001/review/output.md
```

**Requirements exercised:** QE-01, QE-02, QE-04, QE-05, QE-06, QE-07, QE-08, QE-14, QE-15, QE-16, QE-17, IN-01, IN-02, IN-04, YA-01, YA-03, YA-05, YA-06, YA-08, YA-09, YA-11, YA-12, YA-13, YA-16, YA-17

### Story 2: "I want to schedule a recurring workflow"

**As a** developer
**I want to** run a workflow every night at 2am
**So that** I can automate nightly analysis without manual intervention

**Flow:**
```bash
# 1. Define a cron workflow
cat > nightly-report.yaml << 'EOF'
schema_version: "0.1.0"
name: nightly-report
queue:
  category: cron
  schedule:
    cron: "0 2 * * *"
    timezone: "America/Chicago"
    catchup: bounded(5)
model:
  provider: llama-cpp
  model_path: ./models/phi-3-mini-q4_k_m.gguf
steps:
  - id: gather-data
    tools: [shell]
    prompt:
      user: "Run `git log --since yesterday --oneline` and summarize changes"
    timeout_s: 60
  - id: write-report
    tools: [file_write]
    prompt:
      user: "Based on {{steps.gather-data.output}}, write a daily report to ./reports/daily-{{date}}.md"
EOF

# 2. Register the schedule
agent-queue schedule nightly-report.yaml

# 3. Check upcoming runs
agent-queue list --queue cron --state scheduled
# → CRON-001  SCHEDULED  cron  nightly-report  next: 2025-04-03T02:00:00

# 4. After running, check history
agent-queue list --queue cron --state done --limit 5
# → CRON-003  DONE  cron  nightly-report  completed 6h ago
# → CRON-002  DONE  cron  nightly-report  completed 1d ago
# → CRON-001  DONE  cron  nightly-report  completed 2d ago
```

**Requirements exercised:** QE-01, QE-02, QE-03, QE-04, QE-08, QE-09, QE-10, QE-11, QE-14, QE-15, QE-16, QE-17, QE-21, QE-22, QE-23, QE-24, QE-25, QE-27, QE-29, QE-30, IN-01, IN-02, IN-04, IN-05, YA-01, YA-03, YA-05, YA-06, YA-08, YA-10, YA-11, YA-13, YA-15, YA-16, YA-17

### Story 3: "I want to recover from a failure"

**As a** developer
**I want to** inspect failed tasks, understand what went wrong, and retry
**So that** transient failures don't require manual re-execution

**Flow:**
```bash
# 1. A task failed
agent-queue list --state failed
# → TASK-042  FAILED  whenever  data-process  3 attempts exhausted

# 2. Inspect the failure
agent-queue inspect TASK-042
# → State: DLQ (exhausted retries)
# → Error: LlmError::ContextWindowExceeded
# → Step: step-3-summarize
# → Attempt 3/3 failed at: 2025-04-02T14:32:00
# → Retry delays: 1.2s → 2.1s → 4.3s
# → Last heartbeat: 2025-04-02T14:31:45

# 3. The issue was transient (model was overloaded)
#    Requeue from DLQ with override priority
agent-queue retry TASK-042 --priority high --reason "transient overload"

# 4. Monitor the retry
agent-queue inspect TASK-043  # New task ID from requeue
# → State: RUNNING
# → Attempts: 0/3
```

**Requirements exercised:** QE-02, QE-08, QE-09, QE-10, QE-11, QE-14, QE-16, QE-17, QE-19, QE-25, QE-26, QE-29, IN-01, IN-02, IN-03, IN-06, YA-05, YA-09, YA-11, YA-17

### Story 4: "I want to run multiple workflows and prioritize"

**As a** developer
**I want to** submit multiple workflows with different priorities
**So that** urgent work always runs first

**Flow:**
```bash
# 1. Submit background work
agent-queue enqueue embedding-index.yaml --queue whenever
agent-queue enqueue log-cleanup.yaml --queue whenever
agent-queue enqueue docs-update.yaml --queue whenever

# 2. Submit urgent work
agent-queue enqueue incident-response.yaml --queue asap --priority critical

# 3. Meta-scheduler picks ASAP first
agent-queue list --state running
# → TASK-100  RUNNING  asap/critical  incident-response  0s ago

# 4. After ASAP completes, Whenever tasks run in FIFO order
agent-queue list --state running
# → TASK-097  RUNNING  whenever  embedding-index  0s ago
```

**Requirements exercised:** QE-01, QE-05, QE-06, QE-07, QE-08, QE-13, QE-14, QE-15, QE-16, QE-21, IN-01, IN-02

---

## MVP Requirements Traceability

### P0 Requirements by Feature Area

| Area | MVP Requirements | Total |
|------|-----------------|-------|
| **Entities & State** | QE-01, QE-02, QE-30 | 3 |
| **Persistence** | QE-03 | 1 |
| **Validation** | QE-04 | 1 |
| **Queue Categories** | QE-05, QE-27 | 2 |
| **Priority** | QE-06, QE-07 | 2 |
| **Leases** | QE-08 | 1 |
| **Retry & DLQ** | QE-09, QE-10, QE-11, QE-29 | 4 |
| **Cancellation** | QE-12 | 1 |
| **Concurrency** | QE-13 | 1 |
| **Observability** | QE-14, QE-25 | 2 |
| **CLI** | QE-15, QE-16, QE-17, QE-18, QE-19, QE-20 | 6 |
| **Scheduling** | QE-21, QE-22, QE-23 | 3 |
| **Artifacts** | QE-24 | 1 |
| **Backpressure** | QE-26 | 1 |
| **Admission** | QE-28 | 1 |
| **YAML Schema** | YA-01 | 1 |
| **Workflow IR** | YA-02 | 1 |
| **LLM Provider** | YA-03, YA-04 | 2 |
| **Execution Engine** | YA-05, YA-10, YA-15 | 3 |
| **Tools** | YA-06, YA-07 | 2 |
| **Prompts** | YA-08, YA-13 | 2 |
| **Timeouts** | YA-09 | 1 |
| **Context** | YA-11 | 1 |
| **Output** | YA-12 | 1 |
| **Secrets** | YA-14 | 1 |
| **Validation Mode** | YA-18 | 1 |
| **Integration** | IN-01 through IN-07 | 7 |
| **Total MVP** | | **55** |

### Requirements Coverage (142 total → 55 MVP)

| Priority | Requirements | In MVP | Coverage |
|----------|-------------|--------|----------|
| **P0** | 30 | 30 | 100% |
| **P1** | 47 | 25 | 53% |
| **P2** | 5 | 0 | 0% |
| **Total** | 82 (OpenCode) | 55 | 67% |

### Requirements Mapped to Full Spec

| MVP Feature | Full Spec Requirement ID |
|------------|------------------------|
| Entity definitions | Q-001, R0001 |
| State machine | Q-002, R0004 |
| SQLite persistence | Q-024, R0003 |
| YAML validation | Q-027, Q-026 |
| ASAP queue | QL-001 |
| Whenever queue | QL-003 |
| Scheduled-Once | QL-005 |
| Repeat-Cron | QL-007, Q-048 |
| Priority bands | Q-012 |
| FIFO ordering | Q-003 |
| Lease + heartbeat | Q-005, Q-038 |
| Retry + backoff | Q-008 |
| DLQ | Q-009 |
| Idempotency | Q-004 |
| Cancellation | Q-029 |
| Concurrency limit | Q-006 |
| Structured logging | Q-022 |
| CLI commands | Q-053 |
| Meta-scheduler | Q-033, QL-021 |
| Queue routing | Q-032 |
| Artifacts | Q-040 |
| Audit log | Q-020 |
| Backpressure | Q-007 |
| Due-time gate | Q-011 |
| Admission control | (subset of Q-007) |
| Error classification | (subset of Q-008) |
| Schema version | Q-026 |
| Workflow YAML | Q-057 |
| LLM provider | Q-058 |
| Tool framework | Q-016 |
| Secret env vars | Q-017 |
| Context management | Q-058 |
| Retry per step | Q-008 |
| Deterministic validation | Q-028 |
| Metrics | Q-021 (basic) |

---

## Explicitly Out of MVP

### Deferred to Post-MVP Phase 1 (Next sprint)

| Feature | Requirement ID | Why Deferred |
|---------|---------------|--------------|
| DAG dependencies | Q-010 | Sequential execution sufficient for MVP |
| Multi-tenancy | Q-018 | Single-user MVP — no tenant isolation needed |
| RBAC | Q-019 | Single-user MVP — CLI auth not needed |
| Rate limiting | Q-043 | Not needed for single-user local system |
| Quotas | Q-044 | Not needed for single-user local system |
| Human gates | Q-047 | CLI confirmation is sufficient for MVP |
| Work stealing | Q-037 | Single agent pool in MVP |
| Agent pools | Q-036 | Single agent in MVP |
| Preemption | Q-013 | Not needed with single agent |
| Checkpointing | Q-039 | Restart from beginning is acceptable |
| Caching | Q-041 | Recompute is acceptable |
| Input hashing | Q-042 | Not needed without caching |
| Distributed tracing | Q-023 | Structured logging is sufficient |
| Search/filter | Q-054 | CLI grep is sufficient |

### Deferred to Post-MVP Phase 2 (Later)

| Feature | Requirement ID | Why Deferred |
|---------|---------------|--------------|
| Batching | Q-014 | Optimization, not foundation |
| Priority aging | Q-012 (aging) | FIFO is sufficient without aging for small queues |
| Deadline-driven scheduling | Q-035 | ASAP priority covers urgency |
| Fair share / WRR / DRR | Q-034 | Simple priority ordering is sufficient |
| Spillover queues | QL-023 | Backpressure reject is sufficient |
| Scheduled windows | QL-006 | Not needed for MVP use cases |
| Repeat fixed-delay/rate | QL-008, QL-009 | Cron covers recurring use cases |
| Hook triggers | QL-026 | Manual enqueue is sufficient |
| Sandbox enforcement | QL-016 | Tool allowlist is sufficient |
| Cost-aware scheduling | QL-018 | Single provider, no cost optimization |
| Schema registry | Q-050 | schema_version header is sufficient |
| CI linting | Q-051 | Nice to have, not blocking |
| Migration framework | Q-052 | No data to migrate in MVP |
| Tagging/labeling | Q-055 | Not needed for single-user |
| Policy overlays | Q-056 | Single environment |
| Multi-provider LLM | Q-059 | Single provider is sufficient |
| Queue groups | Q-060 | Single consumer |
| REST API | Q-053 (API part) | CLI only for MVP |
| Pausing | Q-030 | Cancel + re-enqueue covers pause/resume |
| Overrides | Q-031 | Cancel + re-enqueue with different priority covers this |
| Exactly-once outcome | Q-025 | Idempotency key is sufficient for local system |
| Security boundaries | Q-045 | Local-only, no network access |
| Deterministic replay | Q-028 (full) | Dry-run validation is sufficient |
| Deterministic plan | Q-057 | Single workflow execution is sufficient |
| Parallel step execution | Q-058 | Sequential is simpler and sufficient |

---

## MVP Technical Specification

### Rust Crate Structure

```
agent-queue/
├── Cargo.toml
├── src/
│   ├── main.rs                    # CLI entrypoint
│   ├── cli/
│   │   ├── mod.rs
│   │   ├── enqueue.rs             # enqueue command
│   │   ├── list.rs                # list command
│   │   ├── inspect.rs             # inspect command
│   │   ├── cancel.rs              # cancel command
│   │   ├── retry.rs               # retry command
│   │   ├── schedule.rs            # schedule command
│   │   └── drain.rs               # drain command
│   ├── queue/
│   │   ├── mod.rs
│   │   ├── engine.rs              # Queue engine core
│   │   ├── scheduler.rs           # Meta-scheduler
│   │   ├── categories.rs          # Queue category definitions
│   │   ├── lease.rs               # Lease management
│   │   ├── retry.rs               # Retry logic + DLQ
│   │   ├── routing.rs             # Queue routing rules
│   │   └── backpressure.rs        # Admission control
│   ├── state/
│   │   ├── mod.rs
│   │   ├── machine.rs             # State machine definition
│   │   ├── transitions.rs         # Valid transition table
│   │   └── store.rs               # SQLite state store
│   ├── agent/
│   │   ├── mod.rs
│   │   ├── executor.rs            # Agent executor
│   │   ├── step_runner.rs         # Step execution loop
│   │   ├── heartbeat.rs           # Heartbeat management
│   │   └── context.rs             # Context window management
│   ├── model/
│   │   ├── mod.rs
│   │   ├── provider.rs            # LlmProvider trait
│   │   ├── llama_cpp.rs           # llama-cpp-2 implementation
│   │   └── token.rs               # Token counting
│   ├── tools/
│   │   ├── mod.rs
│   │   ├── registry.rs            # Tool registry
│   │   ├── shell.rs               # Shell tool
│   │   ├── file_read.rs           # File read tool
│   │   └── file_write.rs          # File write tool
│   ├── workflow/
│   │   ├── mod.rs
│   │   ├── schema.rs              # YAML schema structs
│   │   ├── parser.rs              # YAML parser + validation
│   │   └── ir.rs                  # Workflow IR
│   ├── artifacts/
│   │   ├── mod.rs
│   │   └── store.rs               # File-based artifact store
│   ├── audit/
│   │   ├── mod.rs
│   │   └── log.rs                 # Audit log writer
│   └── error.rs                   # Error types (thiserror)
├── migrations/
│   └── 001_initial.sql            # SQLite schema
└── tests/
    ├── integration/
    │   ├── queue_test.rs
    │   ├── scheduler_test.rs
    │   └── agent_test.rs
    └── e2e/
        └── workflow_test.rs       # End-to-end workflow tests
```

### MVP Dependencies (Cargo.toml)

```toml
[dependencies]
# CLI
clap = { version = "4", features = ["derive"] }
console = "0.15"

# Async runtime
tokio = { version = "1", features = ["rt", "time", "sync", "fs", "process", "macros"] }

# YAML
serde = { version = "1", features = ["derive"] }
serde_yaml = "0.9"
serde_json = "1"

# Database
rusqlite = { version = "0.31", features = ["bundled", "wal"] }

# LLM
llama-cpp-2 = "0.1"  # Check latest version

# Logging
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["json", "env-filter"] }
tracing-appender = "0.2"

# Error handling
thiserror = "1"
anyhow = "1"

# Time
chrono = { version = "0.4", features = ["serde"] }

# Misc
uuid = { version = "1", features = ["v4", "serde"] }
sha2 = "0.10"
parking_lot = "0.12"

[dev-dependencies]
tempfile = "3"
assert_cmd = "2"
predicates = "3"
```

---

## MVP Data Model

### SQLite Schema

```sql
-- Core entities
CREATE TABLE workflows (
    id TEXT PRIMARY KEY,
    name TEXT NOT NULL,
    yaml_path TEXT NOT NULL,
    schema_version TEXT NOT NULL,
    yaml_content TEXT NOT NULL,
    config_json TEXT NOT NULL,
    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now'))
);

CREATE TABLE runs (
    id TEXT PRIMARY KEY,
    workflow_id TEXT NOT NULL REFERENCES workflows(id),
    queue_category TEXT NOT NULL,
    priority INTEGER NOT NULL DEFAULT 3,
    state TEXT NOT NULL DEFAULT 'new',
    idempotency_key TEXT UNIQUE,
    due_ts TEXT,
    schedule_cron TEXT,
    schedule_timezone TEXT,
    attempts INTEGER NOT NULL DEFAULT 0,
    max_attempts INTEGER NOT NULL DEFAULT 3,
    error_message TEXT,
    error_step_id TEXT,
    lease_agent_id TEXT,
    lease_expiry TEXT,
    last_heartbeat TEXT,
    enqueued_at TEXT NOT NULL DEFAULT (datetime('now')),
    started_at TEXT,
    completed_at TEXT,
    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now'))
);

CREATE INDEX idx_runs_state ON runs(state);
CREATE INDEX idx_runs_queue ON runs(queue_category, priority DESC, enqueued_at ASC);
CREATE INDEX idx_runs_due ON runs(due_ts) WHERE state = 'scheduled';
CREATE INDEX idx_runs_lease ON runs(lease_expiry) WHERE state IN ('leased', 'running');
CREATE INDEX idx_runs_idempotency ON runs(idempotency_key);

-- Step results
CREATE TABLE steps (
    id TEXT PRIMARY KEY,
    run_id TEXT NOT NULL REFERENCES runs(id),
    step_id TEXT NOT NULL,
    step_index INTEGER NOT NULL,
    state TEXT NOT NULL DEFAULT 'pending',
    input_text TEXT,
    output_text TEXT,
    token_usage_json TEXT,
    error_message TEXT,
    started_at TEXT,
    completed_at TEXT,
    duration_ms INTEGER,
    attempts INTEGER NOT NULL DEFAULT 0
);

CREATE INDEX idx_steps_run ON steps(run_id, step_index);

-- Artifacts
CREATE TABLE artifacts (
    id TEXT PRIMARY KEY,
    run_id TEXT NOT NULL REFERENCES runs(id),
    step_id TEXT,
    path TEXT NOT NULL,
    size_bytes INTEGER NOT NULL,
    retention_days INTEGER NOT NULL DEFAULT 30,
    created_at TEXT NOT NULL DEFAULT (datetime('now'))
);

CREATE INDEX idx_artifacts_run ON artifacts(run_id);

-- Audit log (append-only)
CREATE TABLE audit_log (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    ts TEXT NOT NULL DEFAULT (datetime('now')),
    actor TEXT NOT NULL DEFAULT 'system',
    action TEXT NOT NULL,
    entity_type TEXT NOT NULL,
    entity_id TEXT NOT NULL,
    prev_state TEXT,
    new_state TEXT,
    diff_json TEXT,
    reason TEXT
);

CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_ts ON audit_log(ts);
```

---

## MVP CLI Reference

```
agent-queue <COMMAND>

Commands:
  enqueue     Submit a workflow to the queue
  schedule    Register a recurring (cron) workflow
  list        List tasks with optional filters
  inspect     Show detailed task/run information
  cancel      Cancel a queued or running task
  retry       Requeue a failed or DLQ task
  drain       Wait for all in-flight tasks to complete
  run         Start the queue engine (blocks until exit)
  validate    Validate a workflow YAML without executing
  version     Show version information

Flags:
  -v, --verbose    Increase log verbosity
  -q, --quiet      Reduce log output
  --db <PATH>      SQLite database path (default: ./agent-queue.db)
  --config <PATH>  Configuration file path
  -h, --help       Show help
```

### enqueue

```
agent-queue enqueue <WORKFLOW_PATH>

Options:
  --queue <CATEGORY>     Queue category: asap | whenever | scheduled | cron
                          [default: whenever]
  --priority <LEVEL>     Priority: critical | high | normal | low
                          [default: normal]
  --idempotency-key <KEY>  Deduplication key (auto-generated if omitted)
  --due-ts <TIMESTAMP>   Due time for scheduled category
  --tag <KEY=VALUE>      Add tags for filtering (repeatable)
  --dry-run              Validate but don't enqueue

Examples:
  agent-queue enqueue my-workflow.yaml
  agent-queue enqueue my-workflow.yaml --queue asap --priority critical
  agent-queue enqueue backup.yaml --queue scheduled --due-ts "2025-04-03T02:00:00"
```

### list

```
agent-queue list [FILTERS]

Filters:
  --state <STATE>        Filter by state: new, queued, scheduled, leased,
                         running, done, failed, dlq, canceled
  --queue <CATEGORY>     Filter by queue category
  --priority <LEVEL>     Filter by priority level
  --tag <KEY=VALUE>      Filter by tag
  --limit <N>            Max results [default: 20]
  --sort <FIELD>         Sort by: priority, enqueued, started, completed
  --json                 Output as JSON
```

### inspect

```
agent-queue inspect <RUN_ID>

Options:
  --steps                Show step-by-step details
  --history              Show state transition history
  --audit                Show audit trail for this run
  --json                 Output as JSON
```

### cancel

```
agent-queue cancel <RUN_ID>

Options:
  --reason <TEXT>        Reason for cancellation
  --force                Cancel even if running (sends cancel signal)
```

### retry

```
agent-queue retry <RUN_ID>

Options:
  --priority <LEVEL>     Override priority
  --queue <CATEGORY>     Override queue
  --reason <TEXT>        Reason for retry
  --max-attempts <N>     Override max retry attempts
```

### drain

```
agent-queue drain

Options:
  --timeout <SECONDS>    Max wait time [default: 300]
  --cancel-remaining     Cancel remaining tasks after timeout
```

### validate

```
agent-queue validate <WORKFLOW_PATH>

Validates YAML schema without executing.
Checks: schema_version, required fields, tool permissions, step references.

Exit codes:
  0: Valid
  1: Validation error (details in stderr)
```

---

## MVP Implementation Phases

### Phase 1: Foundation (Week 1-2)

**Goal**: Entities, state machine, persistence, validation — the skeleton.

| Task | Effort | Deliverable |
|------|--------|-------------|
| Define entity structs (Workflow, Run, Task, Step, Agent, Queue, Lease) | 4h | `src/state/mod.rs` with all types |
| Implement state machine with compile-time transitions | 6h | `src/state/machine.rs` with exhaustive match |
| SQLite store with WAL mode + migrations | 8h | `src/state/store.rs` + `migrations/001_initial.sql` |
| YAML schema structs (serde derive) | 4h | `src/workflow/schema.rs` |
| YAML parser + strict validation | 6h | `src/workflow/parser.rs` |
| Error types (thiserror) | 2h | `src/error.rs` |
| Structured logging setup (tracing) | 2h | `main.rs` logging init |
| **Phase 1 Total** | **32h** | |

### Phase 2: Queue Engine (Week 2-3)

**Goal**: Scheduling, leases, retries, DLQ — the heartbeat.

| Task | Effort | Deliverable |
|------|--------|-------------|
| Queue categories (ASAP, Whenever, Scheduled, Cron) | 4h | `src/queue/categories.rs` |
| Priority ordering with FIFO tie-breaking | 3h | `src/queue/scheduler.rs` (selection logic) |
| Meta-scheduler (priority-weighted queue selection) | 4h | `src/queue/scheduler.rs` (meta-scheduling) |
| Lease management (claim, heartbeat, expire, reclaim) | 6h | `src/queue/lease.rs` |
| Retry logic (exponential backoff + jitter) | 4h | `src/queue/retry.rs` |
| DLQ management (exhaust → DLQ, requeue from DLQ) | 3h | `src/queue/retry.rs` (DLQ section) |
| Queue routing (YAML metadata → queue assignment) | 3h | `src/queue/routing.rs` |
| Backpressure + admission control | 3h | `src/queue/backpressure.rs` |
| Audit log (append-only SQLite writes) | 2h | `src/audit/log.rs` |
| Scheduler tick loop (Tokio interval) | 4h | `src/queue/engine.rs` (tick loop) |
| **Phase 2 Total** | **36h** | |

### Phase 3: Agent SDK (Week 3-4)

**Goal**: LLM integration, step execution, tools — the brain.

| Task | Effort | Deliverable |
|------|--------|-------------|
| LlmProvider trait definition | 2h | `src/model/provider.rs` |
| llama-cpp-2 provider implementation | 8h | `src/model/llama_cpp.rs` |
| Token counting | 3h | `src/model/token.rs` |
| Context window management (budget, truncation) | 4h | `src/agent/context.rs` |
| Step runner (sequential execution, input/output) | 6h | `src/agent/step_runner.rs` |
| Prompt template rendering | 4h | `src/agent/step_runner.rs` (prompt section) |
| Tool registry + permission model | 4h | `src/tools/registry.rs` |
| Shell tool implementation | 3h | `src/tools/shell.rs` |
| File read/write tools | 3h | `src/tools/file_read.rs`, `file_write.rs` |
| Agent executor (orchestrates step runner + heartbeat) | 6h | `src/agent/executor.rs` |
| Heartbeat management during execution | 3h | `src/agent/heartbeat.rs` |
| Artifact collection + file storage | 3h | `src/artifacts/store.rs` |
| Output parsing (structured extraction from LLM) | 4h | `src/agent/step_runner.rs` (output section) |
| **Phase 3 Total** | **53h** | |

### Phase 4: CLI + Integration (Week 4-5)

**Goal**: User interface, wire everything together — the face.

| Task | Effort | Deliverable |
|------|--------|-------------|
| CLI framework (Clap derive, global args) | 3h | `src/cli/mod.rs` |
| enqueue command | 4h | `src/cli/enqueue.rs` |
| list command (filters, sorting, formatting) | 4h | `src/cli/list.rs` |
| inspect command (details, steps, history) | 4h | `src/cli/inspect.rs` |
| cancel command | 2h | `src/cli/cancel.rs` |
| retry command | 3h | `src/cli/retry.rs` |
| schedule command (cron registration) | 3h | `src/cli/schedule.rs` |
| drain command | 2h | `src/cli/drain.rs` |
| validate command (dry-run) | 2h | `src/cli/validate.rs` |
| Integration: queue → agent dispatch | 6h | Wire queue engine to agent executor |
| Integration: agent → state transitions | 4h | Report back completion/failure |
| Integration: error propagation | 3h | Agent errors → retry/DLQ |
| Cron scheduler (parse cron, slot dedup) | 6h | Cron trigger within scheduler |
| **Phase 4 Total** | **46h** | |

### Phase 5: Testing + Polish (Week 5-6)

**Goal**: Reliability, correctness, documentation — the shield.

| Task | Effort | Deliverable |
|------|--------|-------------|
| Unit tests: state machine transitions | 4h | `tests/state_machine_test.rs` |
| Unit tests: retry logic | 3h | `tests/retry_test.rs` |
| Unit tests: YAML validation | 3h | `tests/validation_test.rs` |
| Integration tests: queue engine | 6h | `tests/integration/queue_test.rs` |
| Integration tests: scheduler | 4h | `tests/integration/scheduler_test.rs` |
| Integration tests: agent executor | 4h | `tests/integration/agent_test.rs` |
| End-to-end test: full workflow | 4h | `tests/e2e/workflow_test.rs` |
| Error scenario tests (crash, timeout, OOM) | 4h | `tests/e2e/failure_test.rs` |
| README + usage documentation | 4h | `README.md` |
| **Phase 5 Total** | **36h** | |

### Total MVP Effort

| Phase | Effort | Duration |
|-------|--------|----------|
| Phase 1: Foundation | 32h | 1 week |
| Phase 2: Queue Engine | 36h | 1 week |
| Phase 3: Agent SDK | 53h | 1.5 weeks |
| Phase 4: CLI + Integration | 46h | 1.5 weeks |
| Phase 5: Testing + Polish | 36h | 1 week |
| **Total** | **203h** | **~6 weeks** |

---

## MVP Success Criteria

### Must-Have (System is useful ONLY if all pass)

| # | Criterion | Verification |
|---|-----------|-------------|
| 1 | Enqueue a YAML workflow and it executes to completion | `agent-queue enqueue test.yaml && agent-queue list --state done` |
| 2 | ASAP tasks run before Whenever tasks | Enqueue both, verify ASAP completes first |
| 3 | Failed tasks retry automatically and reach DLQ | Force a failure, observe 3 retries → DLQ |
| 4 | DLQ tasks can be inspected and requeued | `agent-queue retry <id>` succeeds |
| 5 | Cron workflow runs on schedule | Schedule a 1-minute cron, verify execution |
| 6 | System survives process restart | Kill and restart, queued tasks resume |
| 7 | Structured logs trace every state transition | Grep logs for task_id, see full lifecycle |
| 8 | Artifact output is persisted and accessible | Check `./artifacts/<run_id>/` |
| 9 | CLI validate catches invalid YAML | `agent-queue validate bad.yaml` returns non-zero |
| 10 | Cancel stops a queued task | `agent-queue cancel <id>` sets state to canceled |

### Should-Have (System is significantly better if these pass)

| # | Criterion | Verification |
|---|-----------|-------------|
| 11 | Lease expiry reclaims stuck tasks | Block heartbeat, observe reclaim |
| 12 | Backpressure rejects when queue is full | Fill queue, verify rejection with clear error |
| 13 | Idempotency key prevents duplicate enqueue | Enqueue same key twice, second is rejected |
| 14 | Audit log records all state changes | Query audit_log table |
| 15 | Step-level errors show which step failed | `agent-queue inspect <id> --steps` |

---

## MVP Risk Assessment

### High Risk

| Risk | Impact | Mitigation |
|------|--------|-----------|
| **llama-cpp-2 API instability** | Agent execution broken | Pin version, abstract behind trait, fallback to mock provider for testing |
| **Context window management complexity** | LLM calls fail or produce garbage | Conservative budgets, clear truncation strategy, extensive testing |
| **SQLite concurrency under load** | WAL contention, slow writes | Single-writer pattern, batch writes, connection pooling |
| **LLM output parsing reliability** | Structured extraction fails | Robust error handling, regex + JSON fallback, retry on parse failure |

### Medium Risk

| Risk | Impact | Mitigation |
|------|--------|-----------|
| **Cron library accuracy** | Missed or duplicate cron slots | Use well-tested cron library, add deduplication, test DST transitions |
| **Tool execution security** | Shell tool does something bad | Timeout enforcement, tool allowlist, no network in MVP |
| **Large workflow YAML parsing** | OOM or slow parse | Stream parse, size limits, validation before full load |

### Low Risk

| Risk | Impact | Mitigation |
|------|--------|-----------|
| **CLI UX confusion** | Users don't know how to use it | Good help text, examples in --help, README |
| **Artifact storage filling disk** | Disk full | Retention days, cleanup on startup |

---

## Post-MVP Evolution Roadmap

### Phase 6: Robustness (Post-MVP +2 weeks)

| Feature | Effort | Why |
|---------|--------|-----|
| DAG dependencies (Q-010) | 16h | Parallel step execution unlocks throughput |
| Checkpointing (Q-039) | 12h | Long-running workflows can resume |
| Priority aging (Q-012) | 8h | Prevents Whenever starvation under load |
| Pause/resume (Q-030) | 8h | Operations control |

### Phase 7: Multi-User (Post-MVP +4 weeks)

| Feature | Effort | Why |
|---------|--------|-----|
| Multi-tenancy (Q-018) | 24h | Shared instance for teams |
| RBAC (Q-019) | 16h | Access control |
| Rate limiting (Q-043) | 12h | Fair resource usage |
| Quotas (Q-044) | 12h | Resource governance |

### Phase 8: Scale (Post-MVP +6 weeks)

| Feature | Effort | Why |
|---------|--------|-----|
| Agent pools (Q-036) | 20h | Multiple capability classes |
| Work stealing (Q-037) | 16h | Better utilization |
| Batching (Q-014) | 12h | Throughput for compatible tasks |
| Preemption (Q-013) | 12h | SLA guarantees |

### Phase 9: Ecosystem (Post-MVP +8 weeks)

| Feature | Effort | Why |
|---------|--------|-----|
| REST API (Q-053) | 24h | Programmatic access |
| Hook triggers (QL-026) | 20h | Event-driven workflows |
| Multiple LLM providers (Q-059) | 16h | Cloud + local hybrid |
| Schema registry (Q-050) | 12h | CI enforcement |
| CI linting (Q-051) | 8h | Quality gates |

---

## MVP Effort Estimation

### By Feature Area

| Area | Hours | % of Total |
|------|-------|-----------|
| State & Persistence | 40 | 20% |
| Queue Engine | 36 | 18% |
| Scheduling & Routing | 20 | 10% |
| Agent SDK (LLM) | 30 | 15% |
| Tools Framework | 14 | 7% |
| Step Execution | 20 | 10% |
| CLI | 26 | 13% |
| Integration | 13 | 6% |
| Testing | 32 | 16% |
| **Contingency (20%)** | 40 | — |
| **Total with contingency** | **~245h** | |

### By Developer Profile

| Profile | Phase | Effort | Notes |
|---------|-------|--------|-------|
| Senior Rust + LLM | Phase 1-3 | 121h | Can parallelize state + agent work |
| Mid-level Rust | Phase 4-5 | 82h | CLI + testing can proceed with mocks |
| Solo developer | All | 203h + 40h contingency | ~6-7 weeks full-time |

### Velocity Assumptions

- **Developer**: Solo, senior-level Rust
- **Hours per week**: 35 productive hours
- **Weeks**: 6 (foundation) + 1 (buffer) = **7 weeks total**
- **Model availability**: Local LLM model downloaded and ready
- **No blockers**: No external dependency issues

---

## End of Document

This MVP definition covers:
- **55 features** (30 queue engine + 18 agentsdk + 7 integration)
- **203 hours** of implementation effort (~6 weeks)
- **4 queue categories** (ASAP, Whenever, Scheduled-Once, Repeat-Cron)
- **3 MVP tools** (shell, file_read, file_write)
- **1 LLM provider** (llama-cpp-2) with trait abstraction for more
- **7 CLI commands** (enqueue, list, inspect, cancel, retry, schedule, drain)
- **10 success criteria** (must-have) + 5 should-have
- **Post-MVP roadmap** through 4 evolution phases

For implementation, start with Phase 1 (Foundation) and follow the phase dependencies sequentially.
