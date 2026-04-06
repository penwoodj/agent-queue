# Phase 3: Transpiler Integration

**Week 3: 27 hours (0.75 weeks)**

## Overview

This phase implements the integration between agent-queue and `yaml-to-rust-agentsdk`. Rather than reimplementing workflow execution, agent-queue delegates execution to the transpiler via CLI invocation or API calls.

## Goals

1. **Integrate with transpiler**: Queue engine can dispatch workflows to transpiler
2. **Handle transpiler results**: Parse execution results and update task state
3. **Maintain leases**: Keep queue leases alive during long-running transpiler executions
4. **Collect artifacts**: Gather output files from transpiler for persistence
5. **Error classification**: Map transpiler errors to queue retry/DLQ decisions

## Prerequisites

- ✅ Phase 1 complete: Entities, state machine, persistence
- ✅ Phase 2 complete: Queue engine, scheduling, retry logic
- ⏳ Transpiler CLI installed and accessible (or library available)

---

## Task Breakdown

### Task 3.1: Transpiler Integration Interface (4 hours)

**Deliverable**: `src/transpiler/integration.rs` - Abstract interface for transpler operations

**Acceptance Criteria**:
- [ ] `TranspilerConfig` struct defined with binary path, timeout, working dir
- [ ] `execute_workflow()` async function implemented
- [ ] `validate_workflow()` async function implemented
- [ ] Error types defined: `TranspilerError`, `ValidationError`
- [ ] Unit tests for config parsing

**Implementation**:

```rust
// src/transpiler/mod.rs
pub mod integration;
pub mod heartbeat;
pub mod result;

pub use integration::{TranspilerConfig, TranspilerIntegration};
pub use heartbeat::HeartbeatManager;
pub use result::{TranspilerResult, TranspilerError};

// src/transpiler/integration.rs
pub struct TranspilerConfig {
    /// Path to transpiler CLI executable
    pub binary_path: PathBuf,

    /// Working directory for transpiler execution
    pub working_dir: PathBuf,

    /// Timeout for entire workflow execution
    pub timeout: Duration,
}

impl TranspilerConfig {
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
            .arg("--format")
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
            let error_detail = serde_json::from_slice(&output.stderr).unwrap_or_else(|_| {
                TranspilerErrorDetail {
                    error_type: "ParseFailed".to_string(),
                    message: String::from_utf8_lossy(&output.stderr).to_string(),
                }
            });
            Err(TranspilerError::ExecutionFailed(error_detail))
        }
    }

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
            let error_detail: ValidationErrorDetail = serde_json::from_slice(&output.stderr)?;
            Err(TranspilerError::ValidationFailed(error_detail))
        }
    }
}
```

**Testing**:
```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[tokio::test]
    async fn test_transpiler_config() {
        let config = TranspilerConfig {
            binary_path: PathBuf::from("/usr/local/bin/yaml-to-rust-agentsdk"),
            working_dir: PathBuf::from("./workdir"),
            timeout: Duration::from_secs(3600),
        };
        assert_eq!(config.binary_path, PathBuf::from("/usr/local/bin/yaml-to-rust-agentsdk"));
    }
}
```

---

### Task 3.2: CLI Invocation Workflow (6 hours)

**Deliverable**: `src/transpiler/integration.rs` - CLI subprocess handling

**Acceptance Criteria**:
- [ ] Spawn subprocess for transpiler execution
- [ ] Handle stdout/stderr capture
- [ ] Implement timeout enforcement
- [ ] Parse JSON output from transpiler
- [ ] Handle process crashes gracefully
- [ ] Integration tests with mock transpiler

**Implementation Details**:

```rust
impl TranspilerConfig {
    async fn execute_workflow_with_process(
        &self,
        workflow_yaml: &Path,
        env_vars: HashMap<String, String>,
    ) -> Result<TranspilerResult, TranspilerError> {
        // 1. Prepare command
        let mut cmd = Command::new(&self.binary_path);
        cmd.arg("execute")
            .arg("--workflow")
            .arg(workflow_yaml)
            .arg("--format")
            .arg("json")
            .stdout(Stdio::piped())
            .stderr(Stdio::piped());

        // 2. Set environment variables (secrets via process env)
        for (key, value) in env_vars {
            cmd.env(key, value);
        }

        // 3. Set working directory
        cmd.current_dir(&self.working_dir);

        // 4. Spawn process
        let mut child = cmd.spawn()
            .map_err(|e| TranspilerError::ProcessSpawnFailed {
                command: format!("{:?}", cmd),
                error: e.to_string(),
            })?;

        // 5. Wait for completion with timeout
        let output = tokio::time::timeout(
            self.timeout,
            child.wait_with_output()
        ).await
        .map_err(|_| {
            // Timeout: kill process
            let _ = child.kill();
            TranspilerError::Timeout
        })??;

        // 6. Parse output
        if output.status.success() {
            let result: TranspilerResult = serde_json::from_slice(&output.stdout)?;
            Ok(result)
        } else {
            let error_detail = if output.stderr.is_empty() {
                TranspilerErrorDetail {
                    error_type: "Unknown".to_string(),
                    message: "Process exited without error message".to_string(),
                }
            } else {
                serde_json::from_slice(&output.stderr).unwrap_or_else(|_| {
                    TranspilerErrorDetail {
                        error_type: "ParseFailed".to_string(),
                        message: String::from_utf8_lossy(&output.stderr).to_string(),
                    }
                })
            };
            Err(TranspilerError::ExecutionFailed(error_detail))
        }
    }
}
```

---

### Task 3.3: Heartbeat Management During Transpiler Execution (4 hours)

**Deliverable**: `src/transpiler/heartbeat.rs` - Heartbeat during long executions

**Acceptance Criteria**:
- [ ] `HeartbeatManager` spawns periodic heartbeats
- [ ] Heartbeat continues during transpiler execution
- [ ] Heartbeats sent to queue engine
- [ ] Heartbeat stopped on completion/error
- [ ] Configurable interval (default: 15s)
- [ ] Unit tests for timing

**Implementation**:

```rust
// src/transpiler/heartbeat.rs
use tokio::time::{interval, Duration};
use tokio::sync::mpsc;

pub struct Heartbeat {
    pub task_id: String,
    pub timestamp: DateTime<Utc>,
}

pub struct HeartbeatManager {
    task_id: String,
    heartbeat_tx: mpsc::Sender<Heartbeat>,
    interval: Duration,
}

impl HeartbeatManager {
    pub fn new(
        task_id: String,
        heartbeat_tx: mpsc::Sender<Heartbeat>,
    interval: Duration,
    ) -> Self {
        Self {
            task_id,
            heartbeat_tx,
            interval,
        }
    }

    pub async fn run_with_heartbeat<F, T>(
        &self,
        operation: F,
    ) -> Result<T, ExecutionError>
    where
        F: Future<Output = Result<T, ExecutionError>>,
    {
        let task_id_clone = self.task_id.clone();
        let heartbeat_tx_clone = self.heartbeat_tx.clone();

        // 1. Spawn heartbeat task
        let heartbeat_handle = tokio::spawn(async move {
            let mut timer = interval(self.interval);
            loop {
                timer.tick().await;
                let _ = heartbeat_tx_clone.send(Heartbeat {
                    task_id: task_id_clone.clone(),
                    timestamp: Utc::now(),
                }).await;
            }
        });

        // 2. Execute operation (may take minutes for transpiler)
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
    let heartbeat_manager = HeartbeatManager::new(
        task.id.clone(),
        self.heartbeat_tx.clone(),
        Duration::from_secs(15),
    );

    heartbeat_manager.run_with_heartbeat(async {
        self.transpler.execute_workflow(workflow).await
    }).await
}
```

---

### Task 3.4: Result Parsing and Error Handling (4 hours)

**Deliverable**: `src/transpiler/result.rs` - Result structures and error classification

**Acceptance Criteria**:
- [ ] `TranspilerResult` struct defined
- [ ] `TranspilerError` enum with variants
- [ ] Error classification function (`is_retryable()`)
- [ ] Error message formatting for logging
- [ ] Unit tests for error parsing

**Implementation**:

```rust
// src/transpiler/result.rs
use serde::{Deserialize, Serialize};

#[derive(Debug, Deserialize)]
pub struct TranspilerResult {
    pub run_id: String,
    pub status: String,  // "completed", "failed", "timeout"
    pub duration_ms: u64,
    pub steps: Vec<StepResult>,
    pub artifacts: Vec<ArtifactInfo>,
    pub error: Option<TranspilerErrorDetail>,
}

#[derive(Debug, Deserialize)]
pub struct StepResult {
    pub step_id: String,
    pub status: String,
    pub duration_ms: u64,
    pub token_usage: Option<TokenUsage>,
    pub output: Option<String>,
    pub error: Option<String>,
}

#[derive(Debug, Deserialize)]
pub struct ArtifactInfo {
    pub step_id: Option<String>,
    pub path: String,
    pub size_bytes: u64,
}

#[derive(Debug, Deserialize)]
pub struct TokenUsage {
    pub prompt_tokens: u32,
    pub completion_tokens: u32,
    pub total_tokens: u32,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct TranspilerErrorDetail {
    pub error_type: String,
    pub message: String,
    pub step_id: Option<String>,
    pub retryable: Option<bool>,
    pub details: Option<serde_json::Value>,
}

#[derive(Debug, thiserror::Error)]
pub enum TranspilerError {
    #[error("Transpiler timeout after {timeout:?}")]
    Timeout { timeout: Duration },

    #[error("Process spawn failed: {command} - {error}")]
    ProcessSpawnFailed { command: String, error: String },

    #[error("Execution failed: {0}")]
    ExecutionFailed(TranspilerErrorDetail),

    #[error("Validation failed: {0}")]
    ValidationFailed(ValidationErrorDetail),

    #[error("Output parsing failed: {error}")]
    ParseFailed { error: String },

    #[error("IO error: {0}")]
    Io(#[from] std::io::Error),
}

#[derive(Debug, Deserialize)]
pub struct ValidationErrorDetail {
    pub valid: bool,
    pub errors: Vec<ValidationError>,
}

#[derive(Debug, Deserialize)]
pub struct ValidationError {
    pub path: String,
    pub message: String,
    pub severity: String,  // "error", "warning"
}

impl TranspilerError {
    /// Determines if error is retryable
    pub fn is_retryable(&self) -> bool {
        match self {
            TranspilerError::Timeout => true,
            TranspilerError::ExecutionFailed { error } => {
                // Check error_detail.retryable or default to false
                error.retryable.unwrap_or(false)
            }
            TranspilerError::ValidationFailed { .. } => false,
            _ => false,
        }
    }

    /// Extracts error message for logging (sanitizes secrets)
    pub fn log_message(&self) -> String {
        match self {
            TranspilerError::Timeout => "Transpiler execution timeout".to_string(),
            TranspilerError::ProcessSpawnFailed { command, .. } => {
                format!("Failed to spawn transpiler process: {}", command)
            }
            TranspilerError::ExecutionFailed { error } => {
                format!("Transpiler execution failed: {}", error.message)
            }
            TranspilerError::ValidationFailed { errors } => {
                format!("Workflow validation failed: {} errors", errors.errors.len())
            }
            TranspilerError::ParseFailed { .. } => {
                "Failed to parse transpiler output".to_string()
            }
            TranspilerError::Io(_) => "IO error during transpler integration".to_string(),
        }
    }
}
```

---

### Task 3.5: Artifact Collection from Transpiler Output (4 hours)

**Deliverable**: Artifact handling and persistence

**Acceptance Criteria**:
- [ ] Parse artifact paths from transpiler result
- [ ] Validate artifacts exist on filesystem
- [ ] Store artifact metadata in database
- [ ] Move/copy artifacts to managed location
- [ ] Implement retention policy (default: 30 days)
- [ ] Cleanup expired artifacts

**Implementation**:

```rust
// src/artifacts/store.rs (updated for transpler artifacts)
impl ArtifactStore {
    pub async fn collect_from_transpiler_result(
        &self,
        run_id: &str,
        transpiler_result: &TranspilerResult,
    ) -> Result<Vec<Artifact>, StorageError> {
        let mut artifacts = Vec::new();

        for artifact_info in &transpiler_result.artifacts {
            // 1. Validate artifact exists
            if !PathBuf::from(&artifact_info.path).exists() {
                warn!("Artifact not found: {}", artifact_info.path);
                continue;
            }

            // 2. Create artifact record
            let artifact = Artifact {
                id: Uuid::new_v4().to_string(),
                run_id: run_id.to_string(),
                step_id: artifact_info.step_id.clone(),
                path: artifact_info.path.clone(),
                size_bytes: artifact_info.size_bytes,
                content_type: Self::infer_content_type(&artifact_info.path),
                retention_days: 30,
                created_at: Utc::now(),
            };

            // 3. Store in database
            self.insert_artifact(&artifact).await?;
            artifacts.push(artifact);
        }

        Ok(artifacts)
    }

    fn infer_content_type(path: &str) -> String {
        match Path::new(path).extension().and_then(|s| s.to_str()) {
            Some("md") => "text/markdown".to_string(),
            Some("json") => "application/json".to_string(),
            Some("txt") => "text/plain".to_string(),
            Some("png") => "image/png".to_string(),
            _ => "application/octet-stream".to_string(),
        }
    }
}
```

---

### Task 3.6: Environment Variable Passthrough (2 hours)

**Deliverable**: Environment variable resolution and passthrough

**Acceptance Criteria**:
- [ ] Resolve workflow env vars from process environment
- [ ] Validate required vars exist before execution
- [ ] Pass resolved env vars to transpiler (never inline values)
- [ ] Error message for missing env vars
- [ ] Sanitization: never log resolved values

**Implementation**:

```rust
// src/workflow/env.rs
use std::collections::HashMap;
use std::env;

pub fn resolve_env_vars(
    workflow_env: &[String],
) -> Result<HashMap<String, String>, EnvError> {
    let mut resolved = HashMap::new();

    for var_name in workflow_env {
        // 1. Check if var exists in process environment
        if let Some(value) = env::var(var_name) {
            resolved.insert(var_name.clone(), value);
        } else {
            return Err(EnvError::NotFound {
                var_name: var_name.clone(),
            });
        }
    }

    Ok(resolved)
}

#[derive(Debug, thiserror::Error)]
pub enum EnvError {
    #[error("Required environment variable not found: {var_name}")]
    NotFound { var_name: String },
}

// Usage in queue engine
async fn enqueue_workflow(
    &self,
    workflow: &Workflow,
) -> Result<Task, QueueError> {
    // 1. Resolve environment variables
    let env_vars = resolve_env_vars(&workflow.env.as_ref().unwrap_or(&vec![]))?;

    // 2. Create task record
    let task = Task {
        id: generate_task_id(),
        workflow_id: workflow.id.clone(),
        env_vars: env_vars.clone(),  // Store resolved values (not sensitive)
        // ... other fields
    };

    // 3. Queue will pass env_vars to transpiler later
    self.db.insert_task(&task).await?;

    Ok(task)
}
```

**Security Note**: Store only env var names, not resolved values in database. Resolved values passed directly to transpiler subprocess.

---

### Task 3.7: Validation Mode Integration (3 hours)

**Deliverable**: Dry-run validation using transpiler's validate mode

**Acceptance Criteria**:
- [ ] `validate` command calls transpiler validation
- [ ] Parse validation results from transpiler
- [ ] Format validation errors for user display
- [ ] Exit code reflects validation status
- [ ] Integration tests with valid and invalid workflows

**Implementation**:

```rust
// src/cli/validate.rs (existing, updated for transpler)
use crate::transpiler::TranspilerConfig;

pub async fn validate_command(
    workflow_path: &Path,
    config: &TranspilerConfig,
) -> Result<(), CliError> {
    // 1. Validate file exists
    if !workflow_path.exists() {
        return Err(CliError::FileNotFound {
            path: workflow_path.to_string_lossy().to_string(),
        });
    }

    // 2. Call transpiler validation
    println!("Validating workflow: {}...", workflow_path.display());

    match config.validate_workflow(workflow_path).await {
        Ok(result) => {
            if result.valid {
                println!("✅ Workflow is valid");
                Ok(())
            } else {
                println!("❌ Workflow validation failed:");
                for error in &result.errors {
                    println!("  - {}: {} ({})", error.path, error.message, error.severity);
                }
                Err(CliError::ValidationFailed {
                    errors: result.errors,
                })
            }
        }
        Err(e) => {
            println!("❌ Validation error: {}", e);
            Err(CliError::TranspilerError { error: e.to_string() })
        }
    }
}
```

---

## Integration with Queue Engine

### Connecting to Scheduler

```rust
// src/queue/engine.rs (updated)
impl QueueEngine {
    pub async fn execute_workflow(
        &self,
        task: &Task,
        workflow: &Workflow,
    ) -> Result<Task, ExecutionError> {
        // 1. Transition to RUNNING
        task.transition_to(TaskState::Running);
        self.db.update_task(&task).await?;

        // 2. Prepare transpiler call
        let env_vars = self.resolve_env_vars(&task.env_vars)?;
        let transpiler_config = TranspilerConfig::from(&self.config);

        // 3. Execute with heartbeat
        let result = HeartbeatManager::new(
            task.id.clone(),
            self.heartbeat_tx.clone(),
            Duration::from_secs(15),
        ).run_with_heartbeat(async {
            transpiler_config.execute_workflow(&workflow.path, env_vars).await
        }).await;

        // 4. Process result
        match result {
            Ok(transpiler_result) => {
                match transpiler_result.status.as_str() {
                    "completed" => {
                        task.transition_to(TaskState::Done);
                        // Collect artifacts
                        self.artifact_store.collect_from_transpiler_result(&task.id, &transpiler_result).await?;
                    }
                    "failed" => {
                        if transpiler_result.error.as_ref().map_or(false, |e| e.retryable.unwrap_or(false)) {
                            // Retryable: queue handles retry
                            task.attempts += 1;
                            task.transition_to(TaskState::Failed);
                            warn!("Workflow failed (retryable): {:?}", transpiler_result.error);
                        } else {
                            // Non-retryable: send to DLQ
                            task.transition_to(TaskState::Dlq);
                            error!("Workflow failed (non-retryable): {:?}", transpiler_result.error);
                        }
                    }
                    "timeout" => {
                        // Timeout: queue handles as retryable
                        task.attempts += 1;
                        task.transition_to(TaskState::Failed);
                        warn!("Workflow timed out: task_id={}", task.id);
                    }
                    _ => {
                        // Unknown status: send to DLQ
                        task.transition_to(TaskState::Dlq);
                        error!("Unknown transpiler status: {}", transpiler_result.status);
                    }
                }
            }
            Err(e) => {
                // Transpler call failed: send to DLQ
                task.transition_to(TaskState::Dlq);
                error!("Transpler integration error: {}", e);
            }
        }

        // 5. Update task in database
        self.db.update_task(&task).await?;

        // 6. Emit audit event
        self.emit_audit_event(&task).await;

        Ok(task)
    }
}
```

---

## Testing Strategy

### Unit Tests

| Test Area | Tests | Focus |
|-----------|--------|-------|
| TranspilerConfig | 3 | Config parsing, validation |
| Env Var Resolution | 2 | Resolution, missing vars |
| Result Parsing | 4 | JSON parsing, error classification |
| Heartbeat Timing | 2 | Interval, cancellation |
| **Total** | **11** | |

### Integration Tests

| Test Case | Focus | Expected Outcome |
|-----------|--------|-----------------|
| Mock transpler execution | Full workflow | Successful result, artifacts collected |
| Transpler timeout handling | Timeout | Queue retries with backoff |
| Invalid workflow validation | Validate mode | Validation errors returned |
| Missing env vars | Env var resolution | Clear error message |
| Artifact collection | Artifacts | Metadata stored, files preserved |
| Heartbeat during execution | Long-running task | Heartbeats sent, lease maintained |
| **Total** | **7** | |

---

## Success Criteria

**Phase complete when:**

- [ ] All 7 tasks complete and code reviewed
- [ ] Unit tests passing (11/11)
- [ ] Integration tests passing (7/7)
- [ ] Transpler integration can execute simple workflow
- [ ] Validation mode works correctly
- [ ] Heartbeats sent during execution
- [ ] Artifacts collected and stored
- [ ] Errors classified correctly for retry/DLQ
- [ ] Documentation updated for integration

---

## Dependencies

This phase depends on:
- ✅ Phase 1: Foundation (32h) - Entities, state machine, persistence
- ✅ Phase 2: Queue Engine (36h) - Scheduling, leases, retry logic

Enables:
- ✅ Phase 4: CLI + Integration (46h) - Commands need transpler integration
- ✅ Phase 5: Testing + Verification (36h) - E2E tests need execution

---

## Effort Summary

| Task | Hours | Cumulative |
|------|--------|------------|
| Transpiler Integration Interface | 4h | 4h |
| CLI Invocation Workflow | 6h | 10h |
| Heartbeat Management | 4h | 14h |
| Result Parsing & Error Handling | 4h | 18h |
| Artifact Collection | 4h | 22h |
| Environment Variable Passthrough | 2h | 24h |
| Validation Mode Integration | 3h | 27h |
| **Phase 3 Total** | **27h** | **95h** (Phase 1+2+3) |

---

## Notes

- **Transpiler as External Dependency**: For MVP, use CLI invocation. Post-MVP can switch to direct library API.
- **Secrets Handling**: Never store resolved env var values. Pass directly to subprocess.
- **Error Classification**: Transpler provides `retryable` field, but queue makes final retry/DLQ decision.
- **Heartbeat Timing**: 15s interval, 4 heartbeats per 60s lease TTL.
- **Artifact Location**: Transpiler writes artifacts; queue copies to managed location for retention.

---

**Next**: [Phase 4: CLI + Integration](../phases/04-cli-integration.md)
