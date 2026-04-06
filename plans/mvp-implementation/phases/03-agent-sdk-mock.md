# Phase 3: Stubbed Agent Executor (Week 3)

## Goal

Implement a **stubbed execution engine** that simulates real agent workflow execution using sleep-based mocks. This allows thorough testing of the queue engine without needing the actual workflow execution engine or real LLM calls.

## Approach

Instead of calling real LLM providers or executing real tools, the stubbed executor:

1. **Simulates LLM latency**: Sleeps for 5-30 seconds per step
2. **Simulates tool execution**: Sleeps for 1-5 seconds per tool call
3. **Simulates failures**: 10% random failure rate
4. **Simulates context window exhaustion**: 1% rare event
5. **Tracks all operations** with extensive logging

This allows comprehensive testing of:
- Queue scheduling and priority
- Lease management and heartbeat
- Retry logic and backoff
- DLQ and error handling
- State transitions

## Deliverables

| Task | File | Description |
|------|------|-------------|
| Mock LLM provider | `src/mock/llm_provider.rs` | Sleep-based LLM simulation |
| Mock tools | `src/mock/tools.rs` | Latency-based tool simulation |
| Stubbed executor | `src/agent/mock/executor.rs` | Orchestrates step execution |
| Step runner | `src/agent/step_runner.rs` | Sequential step execution |
| Heartbeat manager | `src/agent/heartbeat.rs` | Lease heartbeat during execution |
| Context manager | `src/agent/context.rs` | Context window management (stubbed) |
| Artifact collector | `src/artifacts/store.rs` | File-based artifact storage |

---

## Task 3.1: Mock LLM Provider (8h)

### File: `src/mock/llm_provider.rs`

```rust
use async_trait::async_trait;
use chrono::Utc;
use rand::Rng;
use std::pin::Pin;
use std::time::Duration;
use tokio_stream::{Stream, StreamExt};
use tracing::{debug, info, warn};

/// Mock LLM request
#[derive(Debug, Clone)]
pub struct MockLlmRequest {
    pub messages: Vec<MockLlmMessage>,
    pub temperature: f32,
    pub max_tokens: u32,
}

/// Mock LLM message
#[derive(Debug, Clone)]
pub struct MockLlmMessage {
    pub role: String,
    pub content: String,
}

/// Mock LLM response
#[derive(Debug, Clone)]
pub struct MockLlmResponse {
    pub content: String,
    pub token_count: TokenUsage,
    pub duration: Duration,
}

/// Token usage tracking
#[derive(Debug, Clone)]
pub struct TokenUsage {
    pub prompt_tokens: u32,
    pub completion_tokens: u32,
    pub total_tokens: u32,
}

/// Mock LLM error
#[derive(Debug, thiserror::Error)]
pub enum MockLlmError {
    #[error("context window exceeded")]
    ContextWindowExceeded,

    #[error("timeout after {0}s")]
    Timeout(u64),

    #[error("simulated failure")]
    SimulatedFailure,
}

/// Mock LLM provider with sleep simulation
pub struct MockLlmProvider {
    pub min_latency_ms: u64,
    pub max_latency_ms: u64,
    pub failure_rate: f32,
    pub context_window_exceed_rate: f32,
}

impl Default for MockLlmProvider {
    fn default() -> Self {
        Self {
            min_latency_ms: 5000,   // 5 seconds
            max_latency_ms: 30000,  // 30 seconds
            failure_rate: 0.10,      // 10% failure rate
            context_window_exceed_rate: 0.01,  // 1% rate
        }
    }
}

#[async_trait]
pub trait MockLlmProviderTrait: Send + Sync {
    async fn generate(&self, request: MockLlmRequest) -> Result<MockLlmResponse, MockLlmError>;

    async fn stream(
        &self,
        request: MockLlmRequest,
    ) -> Result<Pin<Box<dyn Stream<Item = Result<String, MockLlmError>>>, MockLlmError>;

    fn count_tokens(&self, messages: &[MockLlmMessage]) -> usize;
}

#[async_trait]
impl MockLlmProviderTrait for MockLlmProvider {
    async fn generate(&self, request: MockLlmRequest) -> Result<MockLlmResponse, MockLlmError> {
        let start = std::time::Instant::now();

        info!(
            messages_count = request.messages.len(),
            temperature = request.temperature,
            max_tokens = request.max_tokens,
            "Mock LLM generation started"
        );

        // Simulate context window exhaustion (1% chance)
        if rand::thread_rng().gen::<f32>() < self.context_window_exceed_rate {
            warn!("Simulating context window exceeded");
            return Err(MockLlmError::ContextWindowExceeded);
        }

        // Simulate random failure (10% chance)
        if rand::thread_rng().gen::<f32>() < self.failure_rate {
            warn!("Simulating LLM failure");
            return Err(MockLlmError::SimulatedFailure);
        }

        // Calculate sleep duration (5-30 seconds)
        let latency_ms = rand::thread_rng().gen_range(self.min_latency_ms..=self.max_latency_ms);
        let latency = Duration::from_millis(latency_ms);

        debug!(latency_ms = latency_ms, "Sleeping to simulate LLM latency");

        tokio::time::sleep(latency).await;

        // Generate mock response
        let prompt_tokens = self.count_tokens(&request.messages);
        let completion_tokens = rand::thread_rng().gen_range(100..=500);

        let content = generate_mock_response(&request.messages);

        let duration = start.elapsed();

        info!(
            prompt_tokens = prompt_tokens,
            completion_tokens = completion_tokens,
            duration_ms = duration.as_millis(),
            "Mock LLM generation completed"
        );

        Ok(MockLlmResponse {
            content,
            token_count: TokenUsage {
                prompt_tokens: prompt_tokens as u32,
                completion_tokens,
                total_tokens: (prompt_tokens + completion_tokens as usize) as u32,
            },
            duration,
        })
    }

    async fn stream(
        &self,
        request: MockLlmRequest,
    ) -> Result<Pin<Box<dyn Stream<Item = Result<String, MockLlmError>>>, MockLlmError> {
        // For MVP, streaming just returns complete response in one chunk
        let response = self.generate(request).await?;

        let stream = async_stream::try_stream! {
            yield Ok(response.content);
        };

        Ok(Box::pin(stream))
    }

    fn count_tokens(&self, messages: &[MockLlmMessage]) -> usize {
        // Rough estimation: ~4 chars per token
        messages
            .iter()
            .map(|msg| msg.content.len() / 4)
            .sum()
    }
}

fn generate_mock_response(messages: &[MockLlmMessage]) -> String {
    // Generate a mock response based on the last user message
    if let Some(last_user_msg) = messages.iter().find(|m| m.role == "user") {
        format!(
            "Mock LLM response to: {}",
            truncate(&last_user_msg.content, 50)
        )
    } else {
        "Mock LLM response".to_string()
    }
}

fn truncate(s: &str, max_len: usize) -> String {
    if s.len() <= max_len {
        s.to_string()
    } else {
        format!("{}...", &s[..max_len])
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[tokio::test]
    async fn test_mock_llm_generation() {
        let provider = MockLlmProvider::default();

        let request = MockLlmRequest {
            messages: vec![MockLlmMessage {
                role: "user".to_string(),
                content: "Test prompt".to_string(),
            }],
            temperature: 0.7,
            max_tokens: 1000,
        };

        let response = provider.generate(request).await.unwrap();

        assert!(!response.content.is_empty());
        assert!(response.token_count.completion_tokens > 0);
        assert!(response.duration.as_secs() >= 5);
    }

    #[tokio::test]
    async fn test_mock_llm_failure() {
        let provider = MockLlmProvider {
            failure_rate: 1.0,  // 100% failure for testing
            ..Default::default()
        };

        let request = MockLlmRequest {
            messages: vec![MockLlmMessage {
                role: "user".to_string(),
                content: "Test prompt".to_string(),
            }],
            temperature: 0.7,
            max_tokens: 1000,
        };

        let result = provider.generate(request).await;
        assert!(result.is_err());
        assert!(matches!(result.unwrap_err(), MockLlmError::SimulatedFailure));
    }
}
```

### Acceptance Criteria

- [ ] Simulates realistic latency (5-30s)
- [ ] Generates mock responses
- [ ] Simulates 10% failure rate
- [ ] Simulates 1% context window exhaustion
- [ ] Tracks token usage

---

## Task 3.2: Mock Tools (3h)

### File: `src/mock/tools.rs`

```rust
use async_trait::async_trait;
use std::time::Duration;
use tracing::{debug, info};

/// Mock tool execution request
#[derive(Debug, Clone)]
pub struct MockToolRequest {
    pub tool_name: String,
    pub parameters: serde_json::Value,
}

/// Mock tool execution response
#[derive(Debug, Clone)]
pub struct MockToolResponse {
    pub output: serde_json::Value,
    pub duration: Duration,
}

/// Mock tool error
#[derive(Debug, thiserror::Error)]
pub enum MockToolError {
    #[error("tool not found: {0}")]
    ToolNotFound(String),

    #[error("timeout after {0}s")]
    Timeout(u64),

    #[error("simulated failure")]
    SimulatedFailure,
}

/// Mock tool registry
pub struct MockToolRegistry {
    pub min_latency_ms: u64,
    pub max_latency_ms: u64,
}

impl Default for MockToolRegistry {
    fn default() -> Self {
        Self {
            min_latency_ms: 1000,   // 1 second
            max_latency_ms: 5000,   // 5 seconds
        }
    }
}

#[async_trait]
pub trait MockTool: Send + Sync {
    async fn execute(&self, request: MockToolRequest) -> Result<MockToolResponse, MockToolError>;
}

/// Mock shell tool
pub struct MockShellTool {
    min_latency_ms: u64,
    max_latency_ms: u64,
}

impl MockShellTool {
    pub fn new(min_ms: u64, max_ms: u64) -> Self {
        Self {
            min_latency_ms: min_ms,
            max_latency_ms: max_ms,
        }
    }
}

#[async_trait]
impl MockTool for MockShellTool {
    async fn execute(&self, request: MockToolRequest) -> Result<MockToolResponse, MockToolError> {
        let start = std::time::Instant::now();

        info!(
            tool = %request.tool_name,
            command = %request.parameters,
            "Mock shell tool execution started"
        );

        // Simulate latency (1-3 seconds)
        let latency_ms = rand::thread_rng().gen_range(self.min_latency_ms..=self.max_latency_ms);
        tokio::time::sleep(Duration::from_millis(latency_ms)).await;

        let output = serde_json::json!({
            "exit_code": 0,
            "stdout": format!("Mock shell output for: {}", truncate(&request.parameters.to_string(), 50)),
            "stderr": ""
        });

        let duration = start.elapsed();

        info!(
            tool = %request.tool_name,
            duration_ms = duration.as_millis(),
            "Mock shell tool execution completed"
        );

        Ok(MockToolResponse { output, duration })
    }
}

/// Mock file read tool
pub struct MockFileReadTool;

#[async_trait]
impl MockTool for MockFileReadTool {
    async fn execute(&self, request: MockToolRequest) -> Result<MockToolResponse, MockToolError> {
        let start = std::time::Instant::now();

        info!(
            tool = %request.tool_name,
            path = %request.parameters,
            "Mock file read started"
        );

        // Simulate latency (0.5-1 seconds)
        let latency_ms = rand::thread_rng().gen_range(500..=1000);
        tokio::time::sleep(Duration::from_millis(latency_ms)).await;

        let output = serde_json::json!({
            "content": format!("Mock file content for: {}", truncate(&request.parameters.to_string(), 50)),
            "size_bytes": 1024
        });

        let duration = start.elapsed();

        debug!(
            tool = %request.tool_name,
            duration_ms = duration.as_millis(),
            "Mock file read completed"
        );

        Ok(MockToolResponse { output, duration })
    }
}

/// Mock file write tool
pub struct MockFileWriteTool;

#[async_trait]
impl MockTool for MockFileWriteTool {
    async fn execute(&self, request: MockToolRequest) -> Result<MockToolResponse, MockToolError> {
        let start = std::time::Instant::now();

        info!(
            tool = %request.tool_name,
            path = %request.parameters,
            "Mock file write started"
        );

        // Simulate latency (1-2 seconds)
        let latency_ms = rand::thread_rng().gen_range(1000..=2000);
        tokio::time::sleep(Duration::from_millis(latency_ms)).await;

        let output = serde_json::json!({
            "success": true,
            "bytes_written": 512
        });

        let duration = start.elapsed();

        debug!(
            tool = %request.tool_name,
            duration_ms = duration.as_millis(),
            "Mock file write completed"
        );

        Ok(MockToolResponse { output, duration })
    }
}

/// Mock tool registry implementation
impl MockToolRegistry {
    pub async fn execute_tool(
        &self,
        request: MockToolRequest,
    ) -> Result<MockToolResponse, MockToolError> {
        match request.tool_name.as_str() {
            "shell" => {
                let tool = MockShellTool::new(self.min_latency_ms, self.max_latency_ms);
                tool.execute(request).await
            }
            "file_read" => {
                let tool = MockFileReadTool;
                tool.execute(request).await
            }
            "file_write" => {
                let tool = MockFileWriteTool;
                tool.execute(request).await
            }
            _ => Err(MockToolError::ToolNotFound(request.tool_name.clone())),
        }
    }
}

fn truncate(s: &str, max_len: usize) -> String {
    if s.len() <= max_len {
        s.to_string()
    } else {
        format!("{}...", &s[..max_len])
    }
}
```

### Acceptance Criteria

- [ ] Shell tool simulates 1-3s latency
- [ ] File read simulates 0.5-1s latency
- [ ] File write simulates 1-2s latency
- [ ] All tools return mock output
- [ ] Unknown tools return error

---

## Task 3.3: Stubbed Executor (6h)

### File: `src/agent/mock/executor.rs`

```rust
use crate::agent::heartbeat::HeartbeatManager;
use crate::agent::step_runner::StepRunner;
use crate::mock::llm_provider::{MockLlmProvider, MockLlmProviderTrait, MockLlmRequest};
use crate::mock::tools::{MockToolRegistry, MockToolRequest};
use crate::state::{Run, RunState, Step, StepState, Workflow};
use crate::store::SqliteStore;
use crate::workflow::schema::WorkflowSchema;
use chrono::Utc;
use std::sync::Arc;
use std::time::Duration;
use tokio::sync::mpsc;
use tracing::{debug, error, info, warn};

/// Stubbed agent executor that uses sleep-based mocks
pub struct StubbedExecutor {
    store: Arc<SqliteStore>,
    llm_provider: MockLlmProvider,
    tool_registry: MockToolRegistry,
    heartbeat_manager: HeartbeatManager,
}

impl StubbedExecutor {
    pub fn new(store: Arc<SqliteStore>, agent_id: String) -> Self {
        Self {
            store,
            llm_provider: MockLlmProvider::default(),
            tool_registry: MockToolRegistry::default(),
            heartbeat_manager: HeartbeatManager::new(agent_id),
        }
    }

    pub async fn execute_run(&self, run_id: &str) -> Result<(), ExecutionError> {
        info!(run_id = %run_id, "Starting stubbed execution");

        // Get run and workflow
        let mut run = self
            .store
            .get_run(run_id)
            .await?
            .ok_or_else(|| ExecutionError::RunNotFound(run_id.to_string()))?;

        let workflow = self
            .store
            .get_workflow(&run.workflow_id)
            .await?
            .ok_or_else(|| ExecutionError::WorkflowNotFound(run.workflow_id.clone()))?;

        // Parse workflow schema
        let workflow_schema: WorkflowSchema =
            serde_yaml::from_str(&workflow.yaml_content).map_err(ExecutionError::ParseError)?;

        // Start heartbeat task
        let heartbeat_tx = self
            .heartbeat_manager
            .start_heartbeat_task(run_id.to_string());

        // Transition to RUNNING
        run.state = RunState::Running;
        run.started_at = Some(Utc::now());
        // self.store.update_run(&run).await?;
        info!(run_id = %run_id, "Run state set to RUNNING");

        // Execute steps
        let mut steps_result = Ok(());

        for (index, step_schema) in workflow_schema.steps.iter().enumerate() {
            info!(
                run_id = %run_id,
                step_id = %step_schema.id,
                step_index = index,
                "Executing step"
            );

            // Create step entity
            let step_id = format!("step-{}", uuid::Uuid::new_v4());
            let mut step = Step {
                id: step_id.clone(),
                run_id: run_id.to_string(),
                step_id: step_schema.id.clone(),
                step_index: index as u32,
                state: StepState::Pending,
                input_text: None,
                output_text: None,
                token_usage_json: None,
                error_message: None,
                started_at: None,
                completed_at: None,
                duration_ms: None,
                attempts: 0,
                created_at: Utc::now(),
                updated_at: Utc::now(),
            };

            // Execute step
            match self.execute_step(&run, &workflow_schema, &step_schema).await {
                Ok((output, token_usage, duration)) => {
                    step.state = StepState::Completed;
                    step.output_text = Some(output);
                    step.token_usage_json = Some(serde_json::to_string(&token_usage)?);
                    step.completed_at = Some(Utc::now());
                    step.duration_ms = Some(duration.as_millis() as u64);
                    info!(
                        run_id = %run_id,
                        step_id = %step_schema.id,
                        duration_ms = duration.as_millis(),
                        "Step completed successfully"
                    );
                }
                Err(e) => {
                    step.state = StepState::Failed;
                    step.error_message = Some(e.to_string());
                    step.completed_at = Some(Utc::now());
                    error!(
                        run_id = %run_id,
                        step_id = %step_schema.id,
                        error = %e,
                        "Step failed"
                    );

                    // Check if should continue or abort
                    if step_schema.on_error.as_deref() == Some("abort") {
                        steps_result = Err(ExecutionError::StepFailed {
                            step_id: step_schema.id.clone(),
                            error: e.to_string(),
                        });
                        break;
                    }
                }
            }

            // Store step
            // self.store.insert_step(&step).await?;

            // Update run error if step failed
            if step.state == StepState::Failed {
                run.error_message = step.error_message.clone();
                run.error_step_id = Some(step.step_id.clone());
            }
        }

        // Stop heartbeat
        heartbeat_tx.send(()).await.ok();

        // Update run state
        match steps_result {
            Ok(()) => {
                run.state = RunState::Done;
                run.completed_at = Some(Utc::now());
                info!(run_id = %run_id, "Run completed successfully");
            }
            Err(e) => {
                run.state = RunState::Failed;
                run.error_message = Some(e.to_string());
                run.completed_at = Some(Utc::now());
                error!(run_id = %run_id, error = %e, "Run failed");
            }
        }

        // self.store.update_run(&run).await?;

        steps_result
    }

    async fn execute_step(
        &self,
        run: &Run,
        workflow: &WorkflowSchema,
        step_schema: &crate::workflow::schema::StepSchema,
    ) -> Result<(String, crate::mock::llm_provider::TokenUsage, Duration), StepError> {
        let start = std::time::Instant::now();

        // Build prompt
        let prompt = self.build_prompt(run, workflow, step_schema)?;

        // Call mock LLM
        let llm_request = MockLlmRequest {
            messages: vec![crate::mock::llm_provider::MockLlmMessage {
                role: "user".to_string(),
                content: prompt,
            }],
            temperature: step_schema
                .model
                .as_ref()
                .and_then(|m| m.temperature)
                .unwrap_or(workflow.model.temperature.unwrap_or(0.7)),
            max_tokens: step_schema
                .model
                .as_ref()
                .and_then(|m| m.max_tokens)
                .unwrap_or(workflow.model.max_tokens.unwrap_or(4096)),
        };

        let llm_response = self.llm_provider.generate(llm_request).await?;
        let duration = start.elapsed();

        // Execute tools if specified (simulated)
        if let Some(tools) = &step_schema.tools {
            for tool_name in tools {
                let tool_request = MockToolRequest {
                    tool_name: tool_name.clone(),
                    parameters: serde_json::json!({}),
                };
                let _tool_response = self.tool_registry.execute_tool(tool_request).await?;
            }
        }

        Ok((
            llm_response.content.clone(),
            llm_response.token_count.clone(),
            duration,
        ))
    }

    fn build_prompt(
        &self,
        _run: &Run,
        workflow: &WorkflowSchema,
        step_schema: &crate::workflow::schema::StepSchema,
    ) -> Result<String, StepError> {
        let mut prompt = String::new();

        // System prompt
        if let Some(system) = &step_schema.prompt.system {
            prompt.push_str(&format!("System: {}\n\n", system));
        }

        // User prompt
        prompt.push_str(&format!("User: {}\n", step_schema.prompt.user));

        Ok(prompt)
    }
}

#[derive(Debug, thiserror::Error)]
pub enum ExecutionError {
    #[error("run not found: {0}")]
    RunNotFound(String),

    #[error("workflow not found: {0}")]
    WorkflowNotFound(String),

    #[error("parse error: {0}")]
    ParseError(#[from] serde_yaml::Error),

    #[error("step failed: {step_id} - {error}")]
    StepFailed { step_id: String, error: String },

    #[error("store error: {0}")]
    Store(#[from] crate::store::StoreError),
}

#[derive(Debug, thiserror::Error)]
pub enum StepError {
    #[error("LLM error: {0}")]
    Llm(#[from] crate::mock::llm_provider::MockLlmError),

    #[error("tool error: {0}")]
    Tool(#[from] crate::mock::tools::MockToolError),
}
```

### Acceptance Criteria

- [ ] Executes workflow steps sequentially
- [ ] Sends heartbeats during execution
- [ ] Handles step failures with continue/abort
- [ ] Records step outputs and token usage
- [ ] Updates run state to DONE or FAILED

---

## Task 3.4: Heartbeat Manager (3h)

### File: `src/agent/heartbeat.rs`

```rust
use crate::store::SqliteStore;
use std::time::Duration;
use tokio::sync::mpsc;
use tracing::{debug, error, info};

pub struct HeartbeatManager {
    agent_id: String,
}

impl HeartbeatManager {
    pub fn new(agent_id: String) -> Self {
        Self { agent_id }
    }

    pub fn start_heartbeat_task(&self, run_id: String) -> mpsc::Sender<()> {
        let agent_id = self.agent_id.clone();
        let (tx, mut rx) = mpsc::channel(1);

        tokio::spawn(async move {
            let mut interval = tokio::time::interval(Duration::from_secs(15));

            loop {
                tokio::select! {
                    _ = interval.tick() => {
                        // Send heartbeat
                        info!(
                            run_id = %run_id,
                            agent_id = %agent_id,
                            "Sending heartbeat"
                        );

                        // TODO: Call store.update_heartbeat(run_id, agent_id)
                    }
                    _ = rx.recv() => {
                        debug!("Stopping heartbeat task");
                        break;
                    }
                }
            }
        });

        tx
    }
}
```

### Acceptance Criteria

- [ ] Sends heartbeat every 15 seconds
- [ ] Stops when shutdown signal received
- [ ] Logs each heartbeat

---

## Phase 3 Completion Checklist

- [ ] Mock LLM provider with sleep simulation
- [ ] Mock tools with latency
- [ ] Stubbed executor orchestrates steps
- [ ] Heartbeat manager keeps lease alive
- [ ] All mock operations logged
- [ ] Unit tests for mock components
- [ ] Integration tests with queue engine

**Phase 3 Estimated Time**: 53 hours

**Phase 3 Deliverable**: Stubbed execution engine that simulates real agent behavior for testing
