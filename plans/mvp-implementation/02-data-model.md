# Data Model and SQLite Schema

> **Ownership Boundary** (per IN-11): agent-queue tracks **orchestration state**
> (run lifecycle, scheduling, retry, audit). The transpiler tracks **execution state**
> (step progress, checkpoints, token usage, intermediate results). agent-queue's
> Step records are **metadata-only summaries** — populated from transpiler's final
> result, not updated during execution.

## Entity Types

### Workflow

Represents a YAML workflow definition.

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Workflow {
    pub id: String,
    pub name: String,
    pub yaml_path: String,
    pub schema_version: String,
    pub yaml_content: String,
    pub config_json: String,  // Serialized WorkflowConfig
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}
```

### Run

Represents a single execution of a workflow.

```rust
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
```

### Step

Represents a **metadata-only summary** of a transpiler workflow step.
Populated from the transpiler's final result payload — not updated in real-time during execution.

> **Why metadata-only?** The transpiler handles step-level execution internally
> (step progress, checkpoints, retries within a step). agent-queue only needs
> the summary for audit trail and user inspection. Per IN-11, detailed step
> state belongs to the transpiler.

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Step {
    pub id: String,
    pub run_id: String,
    pub step_id: String,        // From transpiler result
    pub step_index: u32,        // Execution order from transpiler
    pub state: StepState,       // Final state reported by transpiler
    pub output_summary: Option<String>,  // Truncated output (first 1KB)
    pub token_usage_json: Option<String>,  // Serialized TokenUsage from transpiler
    pub error_message: Option<String>,     // Error if step failed
    pub duration_ms: Option<u64>,         // Reported by transpiler
    pub created_at: DateTime<Utc>,
}
```

> **Removed fields vs. original design**:
> - `input_text` — transpiler manages inputs
> - `started_at`, `completed_at` — transpiler manages timing
> - `attempts` — transpiler manages step-level retry (disabled per IN-09 for MVP)

### Artifact

Represents **metadata** about an output file from a transpiler execution.
Per IN-12, agent-queue tracks metadata (path, size, retention) but does not
copy or manage the actual files — the transpiler owns artifact lifecycle.

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Artifact {
    pub id: String,
    pub run_id: String,
    pub step_id: Option<String>,
    pub path: String,              // Path as reported by transpiler
    pub size_bytes: u64,
    pub content_type: String,      // Inferred from extension
    pub retention_days: u32,       // Default: 30
    pub created_at: DateTime<Utc>,
}
```

> **Artifact lifecycle** (per IN-12):
> 1. Transpiler creates artifacts during execution
> 2. Transpiler reports artifact paths in result payload
> 3. agent-queue stores metadata only
> 4. Cleanup is optional — agent-queue can delete based on retention policy

### AuditLog

Represents an immutable audit trail entry.

```rust
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

## Enums

### QueueCategory

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash, Serialize, Deserialize)]
#[serde(rename_all = "lowercase")]
pub enum QueueCategory {
    Asap,
    Whenever,
    Scheduled,
    Cron,
}
```

### Priority

```rust
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
```

### RunState

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash, Serialize, Deserialize)]
#[serde(rename_all = "lowercase")]
pub enum RunState {
    New,          // Created, not yet validated
    Queued,       // Validated, waiting for scheduler
    Scheduled,     // Has due_ts, waiting for time gate
    Leased,       // Claimed by agent, not yet executing
    Running,       // Agent is actively executing
    Done,         // Terminal: success
    Failed,       // Non-terminal: may retry
    Dlq,          // Terminal: exhausted retries
    Canceled,     // Terminal: user cancelled
    Expired,      // Lease expired, reclaimable
}
```

### StepState

> **Note**: These are the states reported by the transpiler in its final result.
> agent-queue does NOT track step-level transitions in real-time.

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash, Serialize, Deserialize)]
#[serde(rename_all = "lowercase")]
pub enum StepState {
    Completed,     // Step succeeded
    Failed,        // Step failed (error details in error_message)
    Skipped,       // Step was skipped (conditional execution)
}
```

## SQLite Schema

```sql
-- ============================================================
-- Core entities
-- ============================================================

-- Workflows: YAML workflow definitions
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

-- Runs: Workflow executions
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

-- Indexes for scheduler queries
CREATE INDEX idx_runs_state ON runs(state);
CREATE INDEX idx_runs_queue ON runs(queue_category, priority DESC, enqueued_at ASC);
CREATE INDEX idx_runs_due ON runs(due_ts) WHERE state = 'scheduled';
CREATE INDEX idx_runs_lease ON runs(lease_expiry) WHERE state IN ('leased', 'running');
CREATE INDEX idx_runs_idempotency ON runs(idempotency_key);
CREATE INDEX idx_runs_workflow ON runs(workflow_id);

-- Steps: Metadata-only step summaries from transpiler results
CREATE TABLE steps (
    id TEXT PRIMARY KEY,
    run_id TEXT NOT NULL REFERENCES runs(id),
    step_id TEXT NOT NULL,
    step_index INTEGER NOT NULL,
    state TEXT NOT NULL CHECK(state IN ('completed', 'failed', 'skipped')),
    output_summary TEXT,
    token_usage_json TEXT,
    error_message TEXT,
    duration_ms INTEGER,
    created_at TEXT NOT NULL DEFAULT (datetime('now'))
);

CREATE INDEX idx_steps_run ON steps(run_id, step_index);
CREATE INDEX idx_steps_state ON steps(state);

-- Artifacts: Metadata-only records (per IN-12)
CREATE TABLE artifacts (
    id TEXT PRIMARY KEY,
    run_id TEXT NOT NULL REFERENCES runs(id),
    step_id TEXT,
    path TEXT NOT NULL,
    size_bytes INTEGER NOT NULL,
    content_type TEXT NOT NULL DEFAULT 'application/octet-stream',
    retention_days INTEGER NOT NULL DEFAULT 30,
    created_at TEXT NOT NULL DEFAULT (datetime('now'))
);

CREATE INDEX idx_artifacts_run ON artifacts(run_id);
CREATE INDEX idx_artifacts_step ON artifacts(step_id);

-- ============================================================
-- Audit log (append-only)
-- ============================================================

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

-- ============================================================
-- Views for common queries
-- ============================================================

-- View: Active tasks (queued, leased, running)
CREATE VIEW active_tasks AS
SELECT
    r.id,
    r.workflow_id,
    w.name AS workflow_name,
    r.queue_category,
    r.priority,
    r.state,
    r.attempts,
    r.max_attempts,
    r.lease_agent_id,
    r.lease_expiry,
    r.last_heartbeat,
    r.enqueued_at,
    r.started_at,
    r.error_message
FROM runs r
JOIN workflows w ON r.workflow_id = w.id
WHERE r.state IN ('queued', 'leased', 'running');

-- View: Task history (recent runs)
CREATE VIEW task_history AS
SELECT
    r.id,
    r.workflow_id,
    w.name AS workflow_name,
    r.queue_category,
    r.priority,
    r.state,
    r.attempts,
    r.enqueued_at,
    r.started_at,
    r.completed_at,
    CASE
        WHEN r.completed_at IS NOT NULL THEN
            CAST((julianday(r.completed_at) - julianday(r.started_at)) * 8640000 AS INTEGER)
        ELSE NULL
    END AS duration_ms
FROM runs r
JOIN workflows w ON r.workflow_id = w.id
WHERE r.state IN ('done', 'failed', 'dlq', 'canceled')
ORDER BY r.completed_at DESC
LIMIT 100;

-- View: Failed tasks needing attention
CREATE VIEW failed_tasks AS
SELECT
    r.id,
    r.workflow_id,
    w.name AS workflow_name,
    r.queue_category,
    r.priority,
    r.state,
    r.attempts,
    r.max_attempts,
    r.error_message,
    r.error_step_id,
    r.enqueued_at,
    r.started_at,
    r.completed_at
FROM runs r
JOIN workflows w ON r.workflow_id = w.id
WHERE r.state IN ('failed', 'dlq')
ORDER BY r.completed_at DESC;
```

## Migration Scripts

### Migration 001: Initial Schema

File: `migrations/001_initial.sql`

```sql
PRAGMA foreign_keys = ON;
PRAGMA journal_mode = WAL;
PRAGMA synchronous = NORMAL;
PRAGMA cache_size = -64000;  -- 64MB cache
PRAGMA temp_store = MEMORY;
PRAGMA mmap_size = 268435456;  -- 256MB mmap

-- All tables and indexes from above schema
```

## Data Access Patterns

### Scheduler Queries

```sql
-- 1. Get scheduled tasks that are due
SELECT * FROM runs
WHERE state = 'scheduled' AND due_ts <= ?
ORDER BY due_ts ASC;

-- 2. Get expired leases
SELECT * FROM runs
WHERE state = 'leased' AND lease_expiry <= ?
ORDER BY lease_expiry ASC;

-- 3. Get DLQ candidates
SELECT * FROM runs
WHERE state = 'failed' AND attempts >= max_attempts
ORDER BY updated_at ASC;

-- 4. Apply backpressure
SELECT COUNT(*) FROM runs
WHERE state IN ('leased', 'running');

-- 5. Meta-scheduler: select next task
SELECT * FROM runs
WHERE state = 'queued' AND queue_category = ?
ORDER BY priority DESC, enqueued_at ASC
LIMIT 1;
```

### Agent Queries

```sql
-- 1. Claim lease
UPDATE runs SET
    state = 'leased',
    lease_agent_id = ?,
    lease_expiry = ?,
    last_heartbeat = ?,
    updated_at = datetime('now')
WHERE id = ? AND state = 'queued';

-- 2. Start execution
UPDATE runs SET
    state = 'running',
    started_at = datetime('now'),
    updated_at = datetime('now')
WHERE id = ? AND state = 'leased';

-- 3. Send heartbeat
UPDATE runs SET
    last_heartbeat = datetime('now'),
    lease_expiry = datetime('now', '+60 seconds'),
    updated_at = datetime('now')
WHERE id = ? AND state IN ('leased', 'running');

-- 4. Complete run
UPDATE runs SET
    state = 'done',
    completed_at = datetime('now'),
    updated_at = datetime('now')
WHERE id = ? AND state = 'running';

-- 5. Fail run
UPDATE runs SET
    state = 'failed',
    attempts = attempts + 1,
    error_message = ?,
    error_step_id = ?,
    completed_at = datetime('now'),
    updated_at = datetime('now')
WHERE id = ? AND state = 'running';
```

### CLI Queries

```sql
-- 1. List tasks with filters
SELECT * FROM active_tasks
WHERE (:state IS NULL OR state = :state)
  AND (:queue IS NULL OR queue_category = :queue)
  AND (:priority IS NULL OR priority = :priority)
ORDER BY priority DESC, enqueued_at ASC
LIMIT :limit;

-- 2. Inspect task details
SELECT
    r.*,
    w.name AS workflow_name,
    w.yaml_path
FROM runs r
JOIN workflows w ON r.workflow_id = w.id
WHERE r.id = ?;

-- 3. Get step history for run
SELECT * FROM steps
WHERE run_id = ?
ORDER BY step_index ASC;

-- 4. Get audit trail for run
SELECT * FROM audit_log
WHERE entity_type = 'run' AND entity_id = ?
ORDER BY ts ASC;
```

## Transaction Patterns

### Enqueue Transaction

```sql
BEGIN IMMEDIATE;

-- 1. Check idempotency key
SELECT id FROM runs WHERE idempotency_key = ?;

-- 2. Insert workflow if new
INSERT OR IGNORE INTO workflows (...) VALUES (...);

-- 3. Insert run
INSERT INTO runs (...) VALUES (...);

-- 4. Audit log
INSERT INTO audit_log (...) VALUES (...);

COMMIT;
```

### State Transition Transaction

```sql
BEGIN IMMEDIATE;

-- 1. Update run state
UPDATE runs SET state = ?, updated_at = datetime('now')
WHERE id = ? AND state = ?;

-- 2. Audit log
INSERT INTO audit_log (...) VALUES (...);

COMMIT;
```

## Performance Considerations

### Index Strategy

- **Covering indexes**: Include all columns needed for common queries
- **Partial indexes**: Only index rows matching WHERE clause
- **Composite indexes**: Multi-column indexes for common filter/sort combinations

### Query Optimization

- **Prepared statements**: All queries use prepared statements
- **Parameterized queries**: Prevent SQL injection, enable caching
- **EXPLAIN QUERY PLAN**: Analyze slow queries

### Connection Pooling

- Single writer connection for transactions
- Multiple reader connections for SELECT queries
- Connection pool size: 5 readers + 1 writer

## Data Retention

### Automatic Cleanup

```sql
-- Delete artifacts older than retention period
DELETE FROM artifacts
WHERE created_at < datetime('now', '-' || retention_days || ' days');

-- Delete old audit logs (keep 90 days)
DELETE FROM audit_log
WHERE ts < datetime('now', '-90 days');

-- Delete old completed runs (keep 30 days)
DELETE FROM runs
WHERE state IN ('done', 'canceled')
  AND completed_at < datetime('now', '-30 days');
```

### Manual Cleanup CLI

```bash
# Clean up old artifacts
agent-queue cleanup artifacts --older-than 30d

# Clean up old audit logs
agent-queue cleanup audit --older-than 90d

# Clean up completed runs
agent-queue cleanup runs --older-than 30d
```
