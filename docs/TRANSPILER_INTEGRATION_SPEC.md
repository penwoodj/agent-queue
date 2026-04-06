# Transpiler Integration Specification

## Document Purpose

This document defines the integration contract between `agent-queue` (queue orchestration system) and `yaml-to-rust-agentsdk` (workflow execution engine). It specifies the interfaces, protocols, data flows, and error handling mechanisms that enable agent-queue to delegate workflow execution to the transpiler.

---

## Table of Contents

1. [Integration Architecture](#integration-architecture)
2. [Integration Modes](#integration-modes)
3. [Interface Specification](#interface-specification)
4. [Data Flow Protocol](#data-flow-protocol)
5. [Environment Variable Management](#environment-variable-management)
6. [Error Handling Strategy](#error-handling-strategy)
7. [Heartbeat Protocol](#heartbeat-protocol)
8. [Artifact Management](#artifact-management)
9. [Validation Integration](#validation-integration)
10. [Configuration](#configuration)
11. [Testing Strategy](#testing-strategy)
12. [Post-MVP Evolution](#post-mvp-evolution)

---

## Integration Architecture

### High-Level Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                     agent-queue                           │
│                                                            │
│  ┌─────────────┐    ┌──────────────┐    ┌───────────┐ │
│  │ Queue Engine │───▶│   Scheduler  │───▶│ Transpiler│ │
│  │             │◀───│              │◀───│ Integration│ │
│  └─────────────┘    └──────────────┘    └─────┬─────┘ │
│       │                                         │         │
│       │                                         ▼         │
│   ┌───▼────┐                            ┌──────────────┐│
│   │ SQLite  │                            │ yaml-to-    ││
│   │ (State) │                            │ local-rust-  ││
│   └─────────┘                            │ agentsdk     ││
│                                          │ (External)   ││
│                                          └──────────────┘│
└─────────────────────────────────────────────────────────────┘
```

### Component Responsibilities

| Component | Responsibility | Not Responsible |
|-----------|---------------|-----------------|
| **agent-queue** | Queue orchestration, scheduling, state management, lease management, retry logic | Workflow execution, LLM calls, tool invocation |
| **yaml-to-rust-agentsdk** | Workflow parsing, step execution, LLM integration, tool invocation, output generation | Queue management, scheduling, state persistence |

---

## Integration Modes

### MVP: CLI Invocation (Synchronous)

For MVP, agent-queue invokes transpiler as a subprocess:

**Pros:**
- Simple implementation
- Clear process boundaries
- Easy to debug

**Cons:**
- Process spawning overhead
- Limited interactivity during execution

**Use Case:** MVP where simplicity is priority over performance

### Post-MVP: Direct Library API (Asynchronous)

For production, agent-queue uses transpiler's Rust library directly:

**Pros:**
- Zero process overhead
- Direct error propagation
- Fine-grained control

**Cons:**
- Tight coupling (same Rust version)
- More complex error handling

**Use Case:** Production deployments where performance matters

---

## Interface Specification

### MVP: CLI Invocation Interface

#### Execute Workflow

```rust
pub struct TranspilerCliConfig {
    /// Path to transpiler CLI executable
    pub binary_path: PathBuf,

    /// Working directory for transpiler execution
    pub working_dir: PathBuf,

    /// Timeout for entire workflow execution
    pub timeout: Duration,
}

impl TranspilerCliConfig {
    pub async fn execute_workflow(
        &self,
        workflow_yaml: &Path,
        env_vars: HashMap<String, String>,
    ) -> Result<TranspilerResult, TranspilerError> {
        // 1. Spawn transpiler CLI process
        let mut cmd = Command::new(&self.binary_path);
        cmd.arg("execute")
            .arg("--workflow")
            .arg(workflow_yaml)
            .arg("--format") // Request JSON output
            .arg("json");

        // 2. Set environment variables
        for (key, value) in env_vars {
            cmd.env(key, value);
        }

        // 3. Set working directory
        cmd.current_dir(&self.working_dir);

        // 4. Execute with timeout
        let output = tokio::time::timeout(
            self.timeout,
            cmd.output()
        ).await
        .map_err(|_| TranspilerError::Timeout)?;

        // 5. Parse output
        if output.status.success() {
            let result: TranspilerResult = serde_json::from_slice(&output.stdout)?;
            Ok(result)
        } else {
            let error: TranspilerErrorDetail = serde_json::from_slice(&output.stderr)
                .unwrap_or_else(|_| TranspilerErrorDetail::parse_failed(&output.stderr));
            Err(TranspilerError::ExecutionFailed(error))
        }
    }
}
```

#### Validate Workflow

```rust
impl TranspilerCliConfig {
    pub async fn validate_workflow(
        &self,
        workflow_yaml: &Path,
    ) -> Result<ValidationResult, TranspilerError> {
        let mut cmd = Command::new(&self.binary_path);
        cmd.arg("validate")
            .arg("--workflow")
            .arg(workflow_yaml)
            .arg("--format")
            .arg("json");

        let output = cmd.output().await?;

        if output.status.success() {
            let result: ValidationResult = serde_json::from_slice(&output.stdout)?;
            Ok(result)
        } else {
            let error: ValidationErrorDetail = serde_json::from_slice(&output.stderr)?;
            Err(TranspilerError::ValidationFailed(error))
        }
    }
}
```

### Post-MVP: Library API Interface

```rust
// In agent-queue
use yaml_to_rust_agentsdk::{Engine, Workflow, ExecutionResult};

pub struct TranspilerLibraryConfig {
    /// Transpiler engine instance
    pub engine: Engine,

    /// Timeout for workflow execution
    pub timeout: Duration,
}

impl TranspilerLibraryConfig {
    pub async fn execute_workflow(
        &self,
        workflow: &Workflow,
        env_vars: HashMap<String, String>,
    ) -> Result<ExecutionResult, TranspilerError> {
        // 1. Load workflow
        let workflow = self.engine.load_workflow(workflow)?;

        // 2. Set environment variables
        for (key, value) in env_vars {
            std::env::set_var(key, value);
        }

        // 3. Execute with timeout
        let result = tokio::time::timeout(
            self.timeout,
            self.engine.execute(workflow)
        ).await
        .map_err(|_| TranspilerError::Timeout)??;

        Ok(result)
    }
}
```

---

## Data Flow Protocol

### Request Flow (Queue → Transpiler)

```yaml
# 1. Queue prepares execution request
{
  "run_id": "TASK-001",
  "workflow_yaml_path": "./workflows/code-review.yaml",
  "queue_config": {
    "category": "asap",
    "priority": "high"
  },
  "environment": {
    "API_KEY": "${API_KEY}",  # Resolved from process env
    "MODEL_PATH": "/models/llama-3.2-3b.gguf"
  },
  "deadline": "2025-04-06T15:30:00Z",
  "artifacts_dir": "./artifacts/TASK-001/"
}

# 2. Transpiler executes workflow
#    (Transpiler handles LLM calls, tools, prompts, etc.)

# 3. Transpiler returns result
{
  "run_id": "TASK-001",
  "status": "completed",
  "duration_ms": 45230,
  "steps": [
    {
      "step_id": "analyze-input",
      "status": "completed",
      "duration_ms": 23100,
      "token_usage": {
        "prompt_tokens": 1234,
        "completion_tokens": 567,
        "total_tokens": 1801
      },
      "output": "Analysis complete..."
    }
  ],
  "artifacts": [
    {
      "step_id": "analyze-input",
      "path": "./artifacts/TASK-001/step-1/output.md",
      "size_bytes": 2048
    }
  ],
  "error": null
}
```

### Error Flow (Transpiler → Queue)

```yaml
# 4. Transpiler reports error
{
  "run_id": "TASK-001",
  "status": "failed",
  "duration_ms": 15230,
  "error": {
    "type": "ContextWindowExceeded",
    "message": "Context window exceeded at step step-3-summarize",
    "step_id": "step-3-summarize",
    "retryable": true,
    "details": {
      "context_limit": 4096,
      "actual_tokens": 5678
    }
  }
}

# 5. Queue decides: Retry or DLQ
#    - If retryable: Queue requeues task with backoff
#    - If non-retryable: Queue moves task to DLQ
```

---

## Environment Variable Management

### Passthrough Protocol

Agent-queue is responsible for:

1. **Collecting env vars** from workflow YAML `env:` section
2. **Resolving env vars** from process environment (not inline in YAML)
3. **Passing env vars** to transpiler (via CLI or API)
4. **Sanitizing env vars** (never log secrets)

```rust
// Queue-side env var resolution
pub fn resolve_env_vars(
    workflow_env: &[String],
    process_env: &HashMap<String, String>,
) -> Result<HashMap<String, String>, EnvError> {
    let mut resolved = HashMap::new();

    for var_name in workflow_env {
        // 1. Check if var exists in process environment
        if let Some(value) = process_env.get(var_name) {
            resolved.insert(var_name.clone(), value.clone());
        } else {
            return Err(EnvError::NotFound(var_name.clone()));
        }
    }

    Ok(resolved)
}

// Usage
let workflow_env = workflow.env.as_ref().unwrap();
let process_env = std::env::vars().collect();
let resolved = resolve_env_vars(workflow_env, &process_env)?;

// Pass to transpiler
transpiler.execute_workflow(&workflow, resolved).await?;
```

### Secret Handling Rules

| Rule | Rationale |
|-------|-----------|
| **No inline secrets in YAML** | Prevents accidental commits |
| **Always resolve from process env** | Secure by default |
| **Never log resolved values** | Prevents secret leakage |
| **Validate required vars exist** | Fail fast with clear error |
| **Sanitize before error messages** | Prevents secret exposure in logs |

---

## Error Handling Strategy

### Error Classification

Agent-queue classifies transpiler errors for retry/DLQ decision:

```rust
pub enum TranspilerError {
    /// Retryable: Transient error, try again
    Retryable {
        error_type: String,
        message: String,
        retry_after: Duration,
    },

    /// Non-retryable: Permanent error, send to DLQ
    NonRetryable {
        error_type: String,
        message: String,
    },

    /// Timeout: Workflow exceeded deadline
    Timeout,

    /// Validation: Workflow YAML is invalid
    ValidationFailed {
        errors: Vec<String>,
    },
}

impl TranspilerError {
    /// Determines if error is retryable
    pub fn is_retryable(&self) -> bool {
        match self {
            TranspilerError::Retryable { .. } => true,
            TranspilerError::Timeout => true,  // Timeout may be transient
            TranspilerError::NonRetryable { .. } => false,
            TranspilerError::ValidationFailed { .. } => false,
        }
    }

    /// Extracts error message for logging
    pub fn log_message(&self) -> String {
        match self {
            TranspilerError::Retryable { message, .. } => {
                format!("Retryable transpiler error: {}", message)
            }
            TranspilerError::NonRetryable { message, .. } => {
                format!("Non-retryable transpiler error: {}", message)
            }
            TranspilerError::Timeout => "Workflow execution timeout".to_string(),
            TranspilerError::ValidationFailed { errors } => {
                format!("Validation failed: {}", errors.join(", "))
            }
        }
    }
}
```

### Retry Decision Matrix

| Transpiler Error Type | Agent Queue Action | Rationale |
|---------------------|-------------------|-----------|
| `LlmError::Timeout` | **Retry** (exponential backoff) | LLM may be overloaded |
| `LlmError::ContextWindowExceeded` | **DLQ** (non-retryable) | Workflow needs redesign |
| `LlmError::ModelNotLoaded` | **DLQ** (non-retryable) | Configuration issue |
| `ToolError::Timeout` | **Retry** (exponential backoff) | Tool may be slow |
| `ToolError::PermissionDenied` | **DLQ** (non-retryable) | Workflow needs fix |
| `WorkflowError::InvalidYaml` | **DLQ** (non-retryable) | Workflow needs fixing |
| `ExecutionTimeout` | **Retry** (exponential backoff) | System may be overloaded |

---

## Heartbeat Protocol

### Purpose

During long-running transpiler executions, agent-queue must keep its lease alive to prevent reclamation.

### Implementation

```rust
pub struct HeartbeatManager {
    task_id: String,
    heartbeat_tx: mpsc::Sender<Heartbeat>,
    interval: Duration,  // Default: 15 seconds
}

impl HeartbeatManager {
    pub async fn run_with_heartbeat<F, T>(
        task_id: String,
        heartbeat_tx: mpsc::Sender<Heartbeat>,
        interval: Duration,
        operation: F,
    ) -> Result<T, ExecutionError>
    where
        F: Future<Output = Result<T, ExecutionError>>,
    {
        // 1. Spawn heartbeat task
        let task_id_clone = task_id.clone();
        let heartbeat_handle = tokio::spawn(async move {
            let mut timer = tokio::time::interval(interval);
            loop {
                timer.tick().await;
                let _ = heartbeat_tx.send(Heartbeat {
                    task_id: task_id_clone.clone(),
                    timestamp: Utc::now(),
                }).await;
            }
        });

        // 2. Execute operation (may take minutes)
        let result = operation.await;

        // 3. Stop heartbeat
        heartbeat_handle.abort();

        result
    }
}

// Usage in queue engine
async fn execute_workflow_with_heartbeat(
    &self,
    task: &Task,
    workflow: &Workflow,
) -> Result<TranspilerResult, ExecutionError> {
    HeartbeatManager::run_with_heartbeat(
        task.id.clone(),
        self.heartbeat_tx.clone(),
        Duration::from_secs(15),
        async {
            self.transpiler.execute_workflow(workflow).await
        }
    ).await
}
```

### Heartbeat Timing

| Parameter | Default | Rationale |
|-----------|---------|-----------|
| `heartbeat_interval` | 15s | 4 heartbeats per 60s lease TTL |
| `lease_ttl` | 60s | Sufficient for LLM inference |
| `max_missed_heartbeats` | 2 | 30s grace period before reclaim |

---

## Artifact Management

### Artifact Collection Flow

```
1. Transpiler executes workflow
   ↓
2. Transpiler writes artifacts to designated directory
   ↓
3. Transpiler returns artifact paths in result JSON
   ↓
4. Agent-queue validates artifacts exist and records in database
   ↓
5. Agent-queue manages artifact retention (cleanup after X days)
```

### Artifact Metadata

```sql
-- Artifacts table in agent-queue SQLite database
CREATE TABLE artifacts (
    id TEXT PRIMARY KEY,
    run_id TEXT NOT NULL REFERENCES runs(id),
    step_id TEXT,                          -- From transpiler result
    path TEXT NOT NULL,                      -- Relative to artifacts dir
    size_bytes INTEGER NOT NULL,
    content_type TEXT,                        -- e.g., "text/markdown"
    retention_days INTEGER NOT NULL DEFAULT 30,
    created_at TEXT NOT NULL DEFAULT (datetime('now'))
);
```

### Artifact Cleanup

Agent-queue runs periodic cleanup of expired artifacts:

```rust
pub async fn cleanup_expired_artifacts(
    &self,
    retention_days: i32,
) -> Result<CleanupStats, StorageError> {
    // 1. Query expired artifacts
    let expired = self.db.query(
        "SELECT id, path FROM artifacts
         WHERE datetime(created_at) < datetime('now', ?)
         ORDER BY created_at",
        &[&format!("-{} days", retention_days)]
    ).await?;

    // 2. Delete files
    let mut deleted_count = 0;
    for artifact in expired {
        fs::remove_file(&artifact.path).await?;
        deleted_count += 1;
    }

    // 3. Delete database records
    self.db.execute(
        "DELETE FROM artifacts
         WHERE datetime(created_at) < datetime('now', ?)",
        &[&format!("-{} days", retention_days)]
    ).await?;

    Ok(CleanupStats {
        artifacts_deleted: deleted_count,
        bytes_freed: /* calculate from files */,
    })
}
```

---

## Validation Integration

### Dry-Run Mode

Agent-queue provides a `validate` command that calls transpiler's validation mode:

```bash
# Agent queue CLI
agent-queue validate my-workflow.yaml

# Internally calls:
transpiler validate --workflow my-workflow.yaml --format json

# Returns validation result:
{
  "valid": false,
  "errors": [
    {
      "path": "/steps/0/prompt/user",
      "message": "Required variable {{input_file}} is not defined",
      "severity": "error"
    },
    {
      "path": "/model",
      "message": "provider must be one of: llama-cpp, ollama, lm-studio",
      "severity": "error"
    }
  ]
}
```

### Queue-Specific Validation

Before calling transpiler, agent-queue validates queue-specific configuration:

```rust
pub fn validate_queue_config(workflow: &Workflow) -> Result<(), QueueValidationError> {
    // 1. Validate queue category
    match workflow.queue.category {
        "asap" | "whenever" | "scheduled" | "cron" => {},
        _ => return Err(QueueValidationError::InvalidCategory {
            category: workflow.queue.category.clone()
        }),
    }

    // 2. Validate priority
    match workflow.queue.priority {
        "critical" | "high" | "normal" | "low" => {},
        _ => return Err(QueueValidationError::InvalidPriority {
            priority: workflow.queue.priority.clone()
        }),
    }

    // 3. Validate cron expression if category is cron
    if workflow.queue.category == "cron" {
        if workflow.queue.schedule.cron.is_none() {
            return Err(QueueValidationError::MissingCronExpression);
        }
        // Validate cron syntax
        cron_parse(&workflow.queue.schedule.cron.as_ref().unwrap())
            .map_err(|e| QueueValidationError::InvalidCron { error: e })?;
    }

    Ok(())
}
```

---

## Configuration

### Agent-Queue Configuration

```toml
# config/agent-queue.toml
[queue]
database_path = "./agent-queue.db"
max_in_flight = 10
queue_depth_limit = 1000

[scheduler]
tick_interval_ms = 1000

[lease]
ttl_seconds = 60
heartbeat_interval_seconds = 15
max_missed_heartbeats = 2

[transpiler]
# MVP: CLI invocation
integration_mode = "cli"  # "cli" | "library"
binary_path = "/usr/local/bin/yaml-to-rust-agentsdk"
working_dir = "./workdir"
timeout_seconds = 3600

# Post-MVP: Library API
# integration_mode = "library"
# library_path = "./libyaml_to_rust_agentsdk.so"

[artifacts]
base_dir = "./artifacts"
retention_days = 30
```

### Transpiler Configuration

The transpiler has its own configuration (not managed by agent-queue):

```toml
# config/transpiler.toml (transpiler's config)
[models]
default_provider = "llama-cpp"
model_dir = "./models"

[logging]
level = "info"
format = "json"
```

---

## Testing Strategy

### Unit Tests

Test transpiler integration interface in isolation:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[tokio::test]
    async fn test_resolve_env_vars_success() {
        let mut process_env = HashMap::new();
        process_env.insert("API_KEY".to_string(), "sk-test-123".to_string());

        let workflow_env = vec!["API_KEY".to_string()];
        let result = resolve_env_vars(&workflow_env, &process_env);

        assert!(result.is_ok());
        let resolved = result.unwrap();
        assert_eq!(resolved.get("API_KEY").unwrap(), "sk-test-123");
    }

    #[tokio::test]
    async fn test_resolve_env_vars_missing() {
        let process_env = HashMap::new();  // Empty

        let workflow_env = vec!["API_KEY".to_string()];
        let result = resolve_env_vars(&workflow_env, &process_env);

        assert!(result.is_err());
        match result.unwrap_err() {
            EnvError::NotFound(var) => assert_eq!(var, "API_KEY"),
            _ => panic!("Expected NotFound error"),
        }
    }
}
```

### Integration Tests

Test full queue → transpiler flow:

```rust
#[tokio::test]
async fn test_transpiler_execution_integration() {
    // 1. Setup test database
    let db = SqliteStore::open_in_memory().await.unwrap();
    let queue_engine = QueueEngine::new(db.clone(), default_queue_config());
    let transpiler = MockTranspiler::new();

    // 2. Enqueue workflow
    let workflow = load_test_workflow("test-workflow.yaml");
    let task = queue_engine.enqueue(&workflow).await.unwrap();

    // 3. Execute workflow (scheduler picks it up)
    let result = transpiler.execute_workflow(&workflow).await.unwrap();

    // 4. Verify result
    assert_eq!(result.status, "completed");
    assert!(!result.steps.is_empty());

    // 5. Verify task state in database
    let updated_task = db.get_task(&task.id).await.unwrap();
    assert_eq!(updated_task.state, TaskState::Done);
}
```

### Mock Transpiler for Testing

```rust
pub struct MockTranspiler {
    latency_ms: u64,
    failure_rate: f32,
}

impl MockTranspiler {
    pub fn new() -> Self {
        Self {
            latency_ms: 1000,
            failure_rate: 0.1,  // 10% failure rate
        }
    }

    pub async fn execute_workflow(
        &self,
        workflow: &Workflow,
    ) -> Result<TranspilerResult, TranspilerError> {
        // 1. Simulate latency
        tokio::time::sleep(Duration::from_millis(self.latency_ms)).await;

        // 2. Simulate random failure
        if rand::random::<f32>() < self.failure_rate {
            return Err(TranspilerError::Retryable {
                error_type: "SimulatedFailure".to_string(),
                message: "Random failure for testing".to_string(),
                retry_after: Duration::from_secs(1),
            });
        }

        // 3. Return mock result
        Ok(TranspilerResult {
            run_id: "MOCK-RUN-001".to_string(),
            status: "completed".to_string(),
            duration_ms: self.latency_ms,
            steps: vec![],
            artifacts: vec![],
            error: None,
        })
    }
}
```

---

## Post-MVP Evolution

### Phase 1: Direct Library API (Post-MVP +2 weeks)

Replace CLI invocation with direct Rust library integration:

| Task | Effort | Deliverable |
|------|--------|-------------|
| Add transpiler as Cargo dependency | 4h | Update Cargo.toml |
| Implement library-based transpiler interface | 8h | Replace CLI with Engine trait |
| Remove subprocess spawning code | 4h | Clean up integration layer |
| Add unit tests for library integration | 6h | Test coverage |
| Performance comparison (CLI vs Library) | 4h | Benchmark results |

### Phase 2: Streaming Integration (Post-MVP +3 weeks)

Enable real-time progress updates from transpiler:

| Task | Effort | Deliverable |
|------|--------|-------------|
| Implement streaming protocol | 12h | WebSocket or SSE channel |
| Update queue engine to handle progress events | 8h | Real-time state updates |
| Add CLI progress indicators | 4h | Show step progress |
| Add streaming tests | 6h | Integration tests |

### Phase 3: Multi-Model Coordination (Post-MVP +4 weeks)

Coordinate model selection and load balancing:

| Task | Effort | Deliverable |
|------|--------|-------------|
| Model capability discovery API | 8h | Query transpiler for available models |
| Queue routing based on model type | 12h | Route to specialized queues |
| Model pool management | 8h | Track model utilization |
| Add model-aware scheduling tests | 8h | Test coverage |

---

## Appendix A: Error Reference

### Transpiler Error Types

| Error Type | Category | Retryable | Queue Action |
|-------------|------------|------------|---------------|
| `LlmError::Timeout` | Transient | Yes | Retry with backoff |
| `LlmError::ContextWindowExceeded` | Permanent | No | DLQ |
| `LlmError::ModelNotLoaded` | Configuration | No | DLQ |
| `LlmError::RateLimited` | Transient | Yes | Retry with backoff |
| `ToolError::Timeout` | Transient | Yes | Retry with backoff |
| `ToolError::PermissionDenied` | Configuration | No | DLQ |
| `ToolError::CommandNotFound` | Configuration | No | DLQ |
| `WorkflowError::InvalidYaml` | Configuration | No | DLQ |
| `WorkflowError::StepNotFound` | Configuration | No | DLQ |
| `ExecutionError::Timeout` | Transient | Yes | Retry with backoff |
| `ExecutionError::OOM` | Transient | Yes | Retry with backoff |

---

## Appendix B: Configuration Reference

### Environment Variables

| Variable | Required | Description |
|-----------|-----------|-------------|
| `AGENT_QUEUE_DB` | No | Path to SQLite database (default: ./agent-queue.db) |
| `AGENT_QUEUE_CONFIG` | No | Path to config file (default: ./config/agent-queue.toml) |
| `TRANSPILER_BINARY` | No | Path to transpiler CLI (default: yaml-to-rust-agentsdk) |
| `ARTIFACTS_DIR` | No | Path to artifacts directory (default: ./artifacts) |

### CLI Flags

| Flag | Description | Default |
|-------|-------------|----------|
| `--db <PATH>` | SQLite database path | ./agent-queue.db |
| `--config <PATH>` | Configuration file path | ./config/agent-queue.toml |
| `--transpiler <PATH>` | Transpiler binary path | yaml-to-rust-agentsdk |
| `--artifacts <PATH>` | Artifacts directory | ./artifacts |
| `--verbose` | Enable verbose logging | false |
| `--quiet` | Reduce log output | false |

---

## Document History

| Version | Date | Author | Changes |
|---------|--------|--------|---------|
| 0.1.0 | 2026-04-06 | Initial version - CLI integration spec |
| 0.1.1 | 2026-04-06 | Added library integration, error classification, testing strategy |

---

**End of Document**
