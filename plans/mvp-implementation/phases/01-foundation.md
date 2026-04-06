# Phase 1: Foundation (Week 1)

## Goal

Establish the core data structures, state machine, persistence layer, and validation infrastructure. This phase creates the skeleton that all other components depend on.

## Deliverables

| Task | File | Description |
|------|------|-------------|
| Entity definitions | `src/state/mod.rs` | All entity structs and enums |
| State machine | `src/state/machine.rs` | State transitions with validation |
| SQLite store | `src/state/store.rs` | Database operations |
| YAML schema | `src/workflow/schema.rs` | Workflow YAML structs |
| YAML parser | `src/workflow/parser.rs` | Parsing + validation |
| Error types | `src/error.rs` | Thiserror-based errors |
| Logging setup | `src/main.rs` | Tracing initialization |

## Task 1.1: Define Entity Types (4h)

### File: `src/state/mod.rs`

```rust
use chrono::{DateTime, Utc};
use serde::{Deserialize, Serialize};
use std::fmt;

// Queue category enum
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash, Serialize, Deserialize)]
#[serde(rename_all = "lowercase")]
pub enum QueueCategory {
    Asap,
    Whenever,
    Scheduled,
    Cron,
}

impl fmt::Display for QueueCategory {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            QueueCategory::Asap => write!(f, "asap"),
            QueueCategory::Whenever => write!(f, "whenever"),
            QueueCategory::Scheduled => write!(f, "scheduled"),
            QueueCategory::Cron => write!(f, "cron"),
        }
    }
}

// Priority enum
#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord, Serialize, Deserialize)]
#[serde(rename_all = "lowercase")]
pub enum Priority {
    Critical = 10,
    High = 5,
    Normal = 3,
    Low = 1,
}

impl Default for Priority {
    fn default() -> Self {
        Priority::Normal
    }
}

impl fmt::Display for Priority {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            Priority::Critical => write!(f, "critical"),
            Priority::High => write!(f, "high"),
            Priority::Normal => write!(f, "normal"),
            Priority::Low => write!(f, "low"),
        }
    }
}

// Run state enum
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash, Serialize, Deserialize)]
#[serde(rename_all = "lowercase")]
pub enum RunState {
    New,
    Queued,
    Scheduled,
    Leased,
    Running,
    Done,
    Failed,
    Dlq,
    Canceled,
    Expired,
}

impl fmt::Display for RunState {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            RunState::New => write!(f, "new"),
            RunState::Queued => write!(f, "queued"),
            RunState::Scheduled => write!(f, "scheduled"),
            RunState::Leased => write!(f, "leased"),
            RunState::Running => write!(f, "running"),
            RunState::Done => write!(f, "done"),
            RunState::Failed => write!(f, "failed"),
            RunState::Dlq => write!(f, "dlq"),
            RunState::Canceled => write!(f, "canceled"),
            RunState::Expired => write!(f, "expired"),
        }
    }
}

impl RunState {
    pub fn is_terminal(&self) -> bool {
        matches!(self, RunState::Done | RunState::Dlq | RunState::Canceled)
    }

    pub fn is_active(&self) -> bool {
        !self.is_terminal()
    }
}

// Step state enum
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash, Serialize, Deserialize)]
#[serde(rename_all = "lowercase")]
pub enum StepState {
    Pending,
    Running,
    Completed,
    Failed,
}

// Workflow entity
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Workflow {
    pub id: String,
    pub name: String,
    pub yaml_path: String,
    pub schema_version: String,
    pub yaml_content: String,
    pub config_json: String,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

// Run entity
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Run {
    pub id: String,
    pub workflow_id: String,
    pub queue_category: QueueCategory,
    pub priority: Priority,
    pub state: RunState,
    pub idempotency_key: Option<String>,
    pub due_ts: Option<DateTime<Utc>>,
    pub schedule_cron: Option<String>,
    pub schedule_timezone: Option<String>,
    pub attempts: u32,
    pub max_attempts: u32,
    pub error_message: Option<String>,
    pub error_step_id: Option<String>,
    pub lease_agent_id: Option<String>,
    pub lease_expiry: Option<DateTime<Utc>>,
    pub last_heartbeat: Option<DateTime<Utc>>,
    pub enqueued_at: DateTime<Utc>,
    pub started_at: Option<DateTime<Utc>>,
    pub completed_at: Option<DateTime<Utc>>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

impl Run {
    pub fn new(
        id: String,
        workflow_id: String,
        queue_category: QueueCategory,
        priority: Priority,
    ) -> Self {
        let now = Utc::now();
        Run {
            id,
            workflow_id,
            queue_category,
            priority,
            state: RunState::New,
            idempotency_key: None,
            due_ts: None,
            schedule_cron: None,
            schedule_timezone: None,
            attempts: 0,
            max_attempts: 3,
            error_message: None,
            error_step_id: None,
            lease_agent_id: None,
            lease_expiry: None,
            last_heartbeat: None,
            enqueued_at: now,
            started_at: None,
            completed_at: None,
            created_at: now,
            updated_at: now,
        }
    }
}

// Step entity
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Step {
    pub id: String,
    pub run_id: String,
    pub step_id: String,
    pub step_index: u32,
    pub state: StepState,
    pub input_text: Option<String>,
    pub output_text: Option<String>,
    pub token_usage_json: Option<String>,
    pub error_message: Option<String>,
    pub started_at: Option<DateTime<Utc>>,
    pub completed_at: Option<DateTime<Utc>>,
    pub duration_ms: Option<u64>,
    pub attempts: u32,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

// Artifact entity
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Artifact {
    pub id: String,
    pub run_id: String,
    pub step_id: Option<String>,
    pub path: String,
    pub size_bytes: u64,
    pub retention_days: u32,
    pub created_at: DateTime<Utc>,
}

// Audit log entity
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct AuditLog {
    pub id: i64,
    pub ts: DateTime<Utc>,
    pub actor: String,
    pub action: String,
    pub entity_type: String,
    pub entity_id: String,
    pub prev_state: Option<String>,
    pub new_state: Option<String>,
    pub diff_json: Option<String>,
    pub reason: Option<String>,
}
```

### Acceptance Criteria

- [ ] All enums have Display implementations
- [ ] All structs are serializable
- [ ] Run::new() creates valid defaults
- [ ] is_terminal() returns correct values

---

## Task 1.2: Implement State Machine (6h)

### File: `src/state/machine.rs`

```rust
use crate::state::RunState;
use thiserror::Error;

#[derive(Debug, Error)]
pub enum StateError {
    #[error("invalid state transition: {from} → {to}")]
    InvalidTransition { from: RunState, to: RunState },

    #[error("state {0} is terminal, cannot transition")]
    TerminalState(RunState),
}

impl RunState {
    pub fn from_new(self, new_state: RunState) -> Result<RunState, StateError> {
        match new_state {
            RunState::Queued | RunState::Canceled => Ok(new_state),
            _ => Err(StateError::InvalidTransition {
                from: RunState::New,
                to: new_state,
            }),
        }
    }

    pub fn from_queued(self, new_state: RunState) -> Result<RunState, StateError> {
        match new_state {
            RunState::Leased | RunState::Scheduled | RunState::Canceled => Ok(new_state),
            _ => Err(StateError::InvalidTransition {
                from: RunState::Queued,
                to: new_state,
            }),
        }
    }

    pub fn from_scheduled(self, new_state: RunState) -> Result<RunState, StateError> {
        match new_state {
            RunState::Queued | RunState::Canceled => Ok(new_state),
            _ => Err(StateError::InvalidTransition {
                from: RunState::Scheduled,
                to: new_state,
            }),
        }
    }

    pub fn from_leased(self, new_state: RunState) -> Result<RunState, StateError> {
        match new_state {
            RunState::Running | RunState::Expired | RunState::Queued | RunState::Canceled => Ok(new_state),
            _ => Err(StateError::InvalidTransition {
                from: RunState::Leased,
                to: new_state,
            }),
        }
    }

    pub fn from_running(self, new_state: RunState) -> Result<RunState, StateError> {
        match new_state {
            RunState::Done | RunState::Failed | RunState::Canceled | RunState::Expired => Ok(new_state),
            _ => Err(StateError::InvalidTransition {
                from: RunState::Running,
                to: new_state,
            }),
        }
    }

    pub fn from_failed(self, new_state: RunState) -> Result<RunState, StateError> {
        match new_state {
            RunState::Leased | RunState::Dlq | RunState::Queued | RunState::Canceled => Ok(new_state),
            _ => Err(StateError::InvalidTransition {
                from: RunState::Failed,
                to: new_state,
            }),
        }
    }

    pub fn from_expired(self, new_state: RunState) -> Result<RunState, StateError> {
        match new_state {
            RunState::Queued | RunState::Leased | RunState::Failed | RunState::Canceled => Ok(new_state),
            _ => Err(StateError::InvalidTransition {
                from: RunState::Expired,
                to: new_state,
            }),
        }
    }

    pub fn transition_to(self, new_state: RunState) -> Result<RunState, StateError> {
        if self.is_terminal() {
            return Err(StateError::TerminalState(self));
        }

        match self {
            RunState::New => self.from_new(new_state),
            RunState::Queued => self.from_queued(new_state),
            RunState::Scheduled => self.from_scheduled(new_state),
            RunState::Leased => self.from_leased(new_state),
            RunState::Running => self.from_running(new_state),
            RunState::Failed => self.from_failed(new_state),
            RunState::Expired => self.from_expired(new_state),
            RunState::Done | RunState::Dlq | RunState::Canceled => {
                Err(StateError::TerminalState(self))
            }
        }
    }
}
```

### Acceptance Criteria

- [ ] All valid transitions succeed
- [ ] All invalid transitions return error
- [ ] Terminal states cannot transition
- [ ] Unit tests cover all transitions

---

## Task 1.3: SQLite Persistence (8h)

### File: `migrations/001_initial.sql`

```sql
PRAGMA foreign_keys = ON;
PRAGMA journal_mode = WAL;
PRAGMA synchronous = NORMAL;
PRAGMA cache_size = -64000;
PRAGMA temp_store = MEMORY;
PRAGMA mmap_size = 268435456;

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

CREATE INDEX idx_workflows_name ON workflows(name);
CREATE INDEX idx_workflows_schema ON workflows(schema_version);

CREATE TABLE runs (
    id TEXT PRIMARY KEY,
    workflow_id TEXT NOT NULL REFERENCES workflows(id),
    queue_category TEXT NOT NULL CHECK(queue_category IN ('asap', 'whenever', 'scheduled', 'cron')),
    priority INTEGER NOT NULL DEFAULT 3 CHECK(priority IN (1, 3, 5, 10)),
    state TEXT NOT NULL DEFAULT 'new' CHECK(
        state IN ('new', 'queued', 'scheduled', 'leased', 'running', 'done', 'failed', 'dlq', 'canceled', 'expired')
    ),
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
CREATE INDEX idx_runs_workflow ON runs(workflow_id);

CREATE TABLE steps (
    id TEXT PRIMARY KEY,
    run_id TEXT NOT NULL REFERENCES runs(id),
    step_id TEXT NOT NULL,
    step_index INTEGER NOT NULL,
    state TEXT NOT NULL DEFAULT 'pending' CHECK(state IN ('pending', 'running', 'completed', 'failed')),
    input_text TEXT,
    output_text TEXT,
    token_usage_json TEXT,
    error_message TEXT,
    started_at TEXT,
    completed_at TEXT,
    duration_ms INTEGER,
    attempts INTEGER NOT NULL DEFAULT 0,
    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now'))
);

CREATE INDEX idx_steps_run ON steps(run_id, step_index);
CREATE INDEX idx_steps_state ON steps(state);

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
CREATE INDEX idx_artifacts_step ON artifacts(step_id);

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
CREATE INDEX idx_audit_action ON audit_log(action);
CREATE INDEX idx_audit_actor ON audit_log(actor);
```

### File: `src/state/store.rs`

```rust
use crate::state::{Artifact, AuditLog, Run, Step, Workflow};
use chrono::{DateTime, Utc};
use rusqlite::{Connection, params, Result as SqliteResult};
use std::path::Path;
use std::sync::Arc;
use thiserror::Error;

#[derive(Debug, Error)]
pub enum StoreError {
    #[error("database error: {0}")]
    Database(#[from] rusqlite::Error),

    #[error("not found: {0}")]
    NotFound(String),

    #[error("constraint violation: {0}")]
    ConstraintViolation(String),
}

pub type Result<T> = std::result::Result<T, StoreError>;

pub struct SqliteStore {
    conn: Arc<Connection>,
}

impl SqliteStore {
    pub fn open<P: AsRef<Path>>(path: P) -> Result<Self> {
        let conn = Connection::open(path)?;
        conn.execute("PRAGMA foreign_keys = ON", [])?;
        conn.execute("PRAGMA journal_mode = WAL", [])?;

        Ok(Self {
            conn: Arc::new(conn),
        })
    }

    pub fn begin_transaction(&self) -> Result<Transaction> {
        Ok(Transaction::new(Arc::clone(&self.conn)))
    }

    pub async fn insert_workflow(&self, workflow: &Workflow) -> Result<()> {
        let conn = Arc::clone(&self.conn);
        tokio::task::spawn_blocking(move || {
            conn.execute(
                "INSERT INTO workflows (id, name, yaml_path, schema_version, yaml_content, config_json, created_at, updated_at)
                 VALUES (?1, ?2, ?3, ?4, ?5, ?6, ?7, ?8)",
                params![
                    workflow.id,
                    workflow.name,
                    workflow.yaml_path,
                    workflow.schema_version,
                    workflow.yaml_content,
                    workflow.config_json,
                    workflow.created_at.to_rfc3339(),
                    workflow.updated_at.to_rfc3339(),
                ],
            )?;
            Ok::<(), StoreError>(())
        })
        .await?
    }

    pub async fn get_workflow(&self, id: &str) -> Result<Option<Workflow>> {
        let conn = Arc::clone(&self.conn);
        let id = id.to_string();
        tokio::task::spawn_blocking(move || {
            let mut stmt = conn.prepare("SELECT * FROM workflows WHERE id = ?1")?;
            let mut rows = stmt.query(params![id])?;

            if let Some(row) = rows.next()? {
                Ok(Some(Workflow {
                    id: row.get(0)?,
                    name: row.get(1)?,
                    yaml_path: row.get(2)?,
                    schema_version: row.get(3)?,
                    yaml_content: row.get(4)?,
                    config_json: row.get(5)?,
                    created_at: DateTime::parse_from_rfc3339(row.get::<_, String>(6)?.as_str())?.with_timezone(&Utc),
                    updated_at: DateTime::parse_from_rfc3339(row.get::<_, String>(7)?.as_str())?.with_timezone(&Utc),
                }))
            } else {
                Ok(None)
            }
        })
        .await?
    }

    // ... similar methods for runs, steps, artifacts, audit_log
}

pub struct Transaction {
    conn: Arc<Connection>,
}

impl Transaction {
    fn new(conn: Arc<Connection>) -> Self {
        Self { conn }
    }

    pub async fn insert_run(&self, run: &Run) -> Result<()> {
        let conn = Arc::clone(&self.conn);
        let run = run.clone();
        tokio::task::spawn_blocking(move || {
            conn.execute(
                "INSERT INTO runs (id, workflow_id, queue_category, priority, state, attempts, max_attempts, enqueued_at, created_at, updated_at)
                 VALUES (?1, ?2, ?3, ?4, ?5, ?6, ?7, ?8, ?9, ?10)",
                params![
                    run.id,
                    run.workflow_id,
                    run.queue_category.to_string(),
                    run.priority as i32,
                    run.state.to_string(),
                    run.attempts,
                    run.max_attempts,
                    run.enqueued_at.to_rfc3339(),
                    run.created_at.to_rfc3339(),
                    run.updated_at.to_rfc3339(),
                ],
            )?;
            Ok::<(), StoreError>(())
        })
        .await?
    }

    pub async fn commit(self) -> Result<()> {
        let conn = Arc::clone(&self.conn);
        tokio::task::spawn_blocking(move || {
            conn.execute("COMMIT", [])?;
            Ok::<(), StoreError>(())
        })
        .await?
    }
}
```

### Acceptance Criteria

- [ ] Database opens and creates schema
- [ ] Workflow CRUD operations work
- [ ] Run CRUD operations work
- [ ] Transaction support works
- [ ] WAL mode is enabled
- [ ] Foreign key constraints enforced

---

## Task 1.4: YAML Schema (4h)

### File: `src/workflow/schema.rs`

```rust
use serde::{Deserialize, Serialize};
use std::collections::HashMap;

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct WorkflowSchema {
    pub schema_version: String,
    pub name: String,
    pub description: Option<String>,
    pub version: Option<String>,
    pub queue: QueueConfig,
    pub retry: Option<RetryConfig>,
    pub model: ModelConfig,
    pub tools: ToolsConfig,
    pub env: Option<Vec<String>>,
    pub steps: Vec<StepSchema>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct QueueConfig {
    pub category: String,
    pub priority: Option<String>,
    #[serde(flatten)]
    pub schedule: Option<ScheduleConfig>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ScheduleConfig {
    pub cron: Option<String>,
    pub timezone: Option<String>,
    pub catchup: Option<String>,
    pub due_ts: Option<String>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct RetryConfig {
    pub max_attempts: Option<u32>,
    pub base_delay_ms: Option<u64>,
    pub max_delay_ms: Option<u64>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ModelConfig {
    pub provider: String,
    pub model_path: String,
    pub temperature: Option<f32>,
    pub max_tokens: Option<u32>,
    pub context_budget: Option<u32>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ToolsConfig {
    pub allowed: Vec<String>,
    pub blocked: Vec<String>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct StepSchema {
    pub id: String,
    pub description: Option<String>,
    pub prompt: PromptSchema,
    pub model: Option<ModelConfig>,
    pub tools: Option<Vec<String>>,
    pub timeout_s: Option<u64>,
    pub retry: Option<RetryConfig>,
    pub on_error: Option<String>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct PromptSchema {
    pub system: Option<String>,
    pub user: String,
    pub assistant: Option<String>,
}
```

### Acceptance Criteria

- [ ] All YAML fields are represented
- [ ] Optional fields are properly typed
- [ ] Deserialization works with serde_yaml

---

## Task 1.5: YAML Parser + Validation (6h)

### File: `src/workflow/parser.rs`

```rust
use crate::workflow::schema::{WorkflowSchema, StepSchema};
use crate::error::Error;
use serde_yaml;
use std::fs;
use std::path::Path;
use thiserror::Error;

#[derive(Debug, Error)]
pub enum ValidationError {
    #[error("missing required field: {0}")]
    MissingField(String),

    #[error("invalid value for {field}: {message}")]
    InvalidValue { field: String, message: String },

    #[error("schema version not supported: {0}")]
    UnsupportedSchemaVersion(String),

    #[error("step id not unique: {0}")]
    DuplicateStepId(String),

    #[error("step reference not found: {0}")]
    StepNotFound(String),
}

pub struct WorkflowParser;

impl WorkflowParser {
    pub fn parse<P: AsRef<Path>>(path: P) -> Result<WorkflowSchema, Error> {
        let yaml_content = fs::read_to_string(path)?;
        let schema: WorkflowSchema = serde_yaml::from_str(&yaml_content)?;

        // Validate
        Self::validate(&schema)?;

        Ok(schema)
    }

    fn validate(schema: &WorkflowSchema) -> Result<(), ValidationError> {
        // Validate schema version
        if schema.schema_version != "0.1.0" {
            return Err(ValidationError::UnsupportedSchemaVersion(schema.schema_version.clone()));
        }

        // Validate required fields
        if schema.name.is_empty() {
            return Err(ValidationError::MissingField("name".to_string()));
        }

        if schema.steps.is_empty() {
            return Err(ValidationError::MissingField("steps".to_string()));
        }

        // Validate queue category
        let valid_categories = vec!["asap", "whenever", "scheduled", "cron"];
        if !valid_categories.contains(&schema.queue.category.as_str()) {
            return Err(ValidationError::InvalidValue {
                field: "queue.category".to_string(),
                message: format!("must be one of: {}", valid_categories.join(", ")),
            });
        }

        // Validate step IDs are unique
        let mut step_ids = std::collections::HashSet::new();
        for step in &schema.steps {
            if !step_ids.insert(&step.id) {
                return Err(ValidationError::DuplicateStepId(step.id.clone()));
            }
        }

        // Validate tool permissions
        let allowed_tools: std::collections::HashSet<_> = schema.tools.allowed.iter().collect();
        let blocked_tools: std::collections::HashSet<_> = schema.tools.blocked.iter().collect();

        for step in &schema.steps {
            if let Some(tools) = &step.tools {
                for tool in tools {
                    if blocked_tools.contains(tool) {
                        return Err(ValidationError::InvalidValue {
                            field: format!("step.{}.tools", step.id),
                            message: format!("tool '{}' is blocked", tool),
                        });
                    }
                    if !allowed_tools.contains(tool) && !blocked_tools.is_empty() {
                        return Err(ValidationError::InvalidValue {
                            field: format!("step.{}.tools", step.id),
                            message: format!("tool '{}' is not allowed", tool),
                        });
                    }
                }
            }
        }

        Ok(())
    }
}
```

### Acceptance Criteria

- [ ] Valid YAML parses successfully
- [ ] Invalid YAML returns clear error
- [ ] All validations are enforced
- [ ] Error messages are actionable

---

## Task 1.6: Error Types (2h)

### File: `src/error.rs`

```rust
use thiserror::Error;

#[derive(Debug, Error)]
pub enum Error {
    #[error("state machine error: {0}")]
    StateMachine(#[from] crate::state::machine::StateError),

    #[error("store error: {0}")]
    Store(#[from] crate::state::store::StoreError),

    #[error("validation error: {0}")]
    Validation(#[from] crate::workflow::parser::ValidationError),

    #[error("workflow not found: {0}")]
    WorkflowNotFound(String),

    #[error("run not found: {0}")]
    RunNotFound(String),

    #[error("not leaseable: state is {0}")]
    NotLeaseable(String),

    #[error("not completable: state is {0}")]
    NotCompletable(String),

    #[error("not failable: state is {0}")]
    NotFailable(String),

    #[error("lease expired")]
    LeaseExpired,

    #[error("max retries exceeded")]
    MaxRetriesExceeded,

    #[error("context window exceeded")]
    ContextWindowExceeded,
}
```

### Acceptance Criteria

- [ ] All error types are covered
- [ ] Errors chain correctly
- [ ] Error messages are clear

---

## Task 1.7: Logging Setup (2h)

### File: `src/main.rs`

```rust
use tracing::{info, Level};
use tracing_subscriber::{EnvFilter, fmt, prelude::*};

fn init_logging() {
    let env_filter = EnvFilter::from_default_env()
        .add_directive("agent_queue=debug".parse().unwrap())
        .add_directive(Level::INFO.into());

    tracing_subscriber::registry()
        .with(env_filter)
        .with(fmt::layer().json())
        .init();

    info!("Logging initialized");
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    init_logging();

    // TODO: Initialize other components
    info!("Agent Queue starting");

    Ok(())
}
```

### Acceptance Criteria

- [ ] Logs output as JSON
- [ ] Log level is configurable via env
- [ ] Correlation IDs are supported

---

## Testing

### Unit Tests

Create `tests/state_test.rs`:

```rust
#[cfg(test)]
mod tests {
    use crate::state::*;

    #[test]
    fn test_queue_category_display() {
        assert_eq!(QueueCategory::Asap.to_string(), "asap");
        assert_eq!(QueueCategory::Whenever.to_string(), "whenever");
    }

    #[test]
    fn test_priority_ordering() {
        assert!(Priority::Critical > Priority::High);
        assert!(Priority::High > Priority::Normal);
        assert!(Priority::Normal > Priority::Low);
    }

    #[test]
    fn test_run_state_terminal() {
        assert!(RunState::Done.is_terminal());
        assert!(RunState::Dlq.is_terminal());
        assert!(RunState::Canceled.is_terminal());
        assert!(!RunState::Running.is_terminal());
    }

    #[test]
    fn test_run_new() {
        let run = Run::new(
            "run-123".to_string(),
            "workflow-456".to_string(),
            QueueCategory::Asap,
            Priority::High,
        );

        assert_eq!(run.id, "run-123");
        assert_eq!(run.state, RunState::New);
        assert_eq!(run.max_attempts, 3);
    }
}
```

### Integration Tests

Create `tests/integration/store_test.rs`:

```rust
#[tokio::test]
async fn test_insert_and_get_workflow() {
    let store = SqliteStore::open(":memory:").unwrap();

    let workflow = Workflow {
        id: "wf-123".to_string(),
        name: "test-workflow".to_string(),
        yaml_path: "/test/workflow.yaml".to_string(),
        schema_version: "0.1.0".to_string(),
        yaml_content: "test: yaml".to_string(),
        config_json: "{}".to_string(),
        created_at: Utc::now(),
        updated_at: Utc::now(),
    };

    store.insert_workflow(&workflow).await.unwrap();
    let retrieved = store.get_workflow("wf-123").await.unwrap();

    assert!(retrieved.is_some());
    assert_eq!(retrieved.unwrap().name, "test-workflow");
}
```

---

## Phase 1 Completion Checklist

- [ ] All entity types defined
- [ ] State machine implemented
- [ ] SQLite persistence working
- [ ] YAML schema defined
- [ ] YAML parser + validation working
- [ ] Error types defined
- [ ] Logging configured
- [ ] Unit tests pass
- [ ] Integration tests pass
- [ ] Documentation complete

**Phase 1 Estimated Time**: 32 hours

**Phase 1 Deliverable**: Working foundation that can enqueue workflows to queue
