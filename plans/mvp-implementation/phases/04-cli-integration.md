# Phase 4: CLI + Integration (Week 4-5)

**46 hours (1.25 weeks)**

## Overview

This phase implements the full CLI interface and wires together the queue engine, state machine, and transpiler integration. After this phase, a user can enqueue workflows, inspect state, manage retries, and the system executes end-to-end.

## Goals

1. **Complete CLI**: All user-facing commands via Clap
2. **Queue-to-transpiler wiring**: Queue engine dispatches to transpiler
3. **Result-to-state mapping**: Transpiler results update run state correctly
4. **Error propagation**: Transpiler errors classified for retry/DLQ
5. **Cron scheduling**: Scheduled workflows run on time

## Prerequisites

- ✅ Phase 1 complete: Entities, state machine, persistence
- ✅ Phase 2 complete: Queue engine, scheduling, retry logic
- ✅ Phase 3 complete: Transpiler integration interface

---

## Task Breakdown

### Task 4.1: CLI Framework (3 hours)

**Deliverable**: `src/cli/mod.rs` — Clap app structure with subcommands

**Acceptance Criteria**:
- [ ] Clap derive macros for all subcommands
- [ ] Global args: `--config`, `--db-path`, `--log-level`
- [ ] Subcommand routing to handlers
- [ ] Error display formatting
- [ ] Unit tests for arg parsing

**Implementation**:

```rust
// src/cli/mod.rs
use clap::{Parser, Subcommand};

#[derive(Parser)]
#[command(name = "agent-queue", version, about = "Queue orchestration for yaml-to-rust-agentsdk")]
pub struct Cli {
    #[command(subcommand)]
    pub command: Commands,

    /// Path to config file
    #[arg(long, global = true)]
    pub config: Option<PathBuf>,

    /// Path to SQLite database
    #[arg(long, global = true, default_value = "./agent-queue.db")]
    pub db_path: PathBuf,

    /// Log level
    #[arg(long, global = true, default_value = "info")]
    pub log_level: String,
}

#[derive(Subcommand)]
pub enum Commands {
    /// Enqueue a workflow for execution
    Enqueue {
        /// Path to workflow YAML
        #[arg(short, long)]
        workflow: PathBuf,

        /// Queue category (asap, whenever, scheduled, cron)
        #[arg(short, long, default_value = "asap")]
        queue: String,

        /// Priority (critical, high, normal, low)
        #[arg(short, long, default_value = "normal")]
        priority: String,

        /// Due timestamp (ISO 8601) for scheduled tasks
        #[arg(short, long)]
        due: Option<String>,

        /// Cron expression for recurring tasks
        #[arg(short, long)]
        cron: Option<String>,

        /// Idempotency key (prevents duplicate enqueue)
        #[arg(long)]
        idempotency_key: Option<String>,
    },

    /// List queued/running/completed workflows
    List {
        /// Filter by state
        #[arg(short, long)]
        state: Option<String>,

        /// Filter by queue category
        #[arg(short = 'q', long)]
        queue: Option<String>,

        /// Maximum results
        #[arg(short = 'n', long, default_value = "20")]
        limit: u32,
    },

    /// Inspect a specific run
    Inspect {
        /// Run ID
        run_id: String,

        /// Include step details from transpiler result
        #[arg(long)]
        steps: bool,
    },

    /// Cancel a queued or running workflow
    Cancel {
        /// Run ID
        run_id: String,

        /// Force cancel even if running
        #[arg(long)]
        force: bool,
    },

    /// Retry a failed or DLQ workflow
    Retry {
        /// Run ID
        run_id: String,

        /// Reset attempt count
        #[arg(long)]
        reset_attempts: bool,
    },

    /// Schedule a workflow for future execution
    Schedule {
        /// Path to workflow YAML
        #[arg(short, long)]
        workflow: PathBuf,

        /// Cron expression
        #[arg(short, long)]
        cron: String,

        /// Timezone for cron
        #[arg(short, long, default_value = "UTC")]
        timezone: String,

        /// Priority
        #[arg(short, long, default_value = "normal")]
        priority: String,
    },

    /// Drain a queue (cancel all tasks)
    Drain {
        /// Queue category to drain
        queue: String,

        /// Skip confirmation
        #[arg(long)]
        yes: bool,
    },

    /// Validate a workflow YAML without enqueuing
    Validate {
        /// Path to workflow YAML
        workflow: PathBuf,
    },

    /// Start the queue engine daemon
    Daemon {
        /// Worker count
        #[arg(short, long, default_value = "1")]
        workers: u32,

        /// Maximum concurrent runs
        #[arg(long, default_value = "3")]
        max_concurrent: u32,
    },

    /// Database maintenance
    Cleanup {
        /// Remove artifacts older than N days
        #[arg(long, default_value = "30")]
        artifacts_older_than: u32,

        /// Remove completed runs older than N days
        #[arg(long, default_value = "30")]
        runs_older_than: u32,

        /// Remove audit logs older than N days
        #[arg(long, default_value = "90")]
        audit_older_than: u32,

        /// Dry run (show what would be deleted)
        #[arg(long)]
        dry_run: bool,
    },
}
```

---

### Task 4.2: Enqueue Command (4 hours)

**Deliverable**: `src/cli/enqueue.rs` — Full enqueue flow

**Acceptance Criteria**:
- [ ] Parse YAML and validate against transpiler schema (per IN-08)
- [ ] Create workflow and run entities
- [ ] Route to correct queue category
- [ ] Handle idempotency key
- [ ] Handle scheduled/cron workflows
- [ ] Print run ID and state to user
- [ ] Integration tests

---

### Task 4.3: List Command (4 hours)

**Deliverable**: `src/cli/list.rs` — Query and display runs

**Acceptance Criteria**:
- [ ] Filter by state, queue, priority
- [ ] Paginated output
- [ ] Human-readable table format
- [ ] JSON output option (`--format json`)
- [ ] Integration tests

---

### Task 4.4: Inspect Command (4 hours)

**Deliverable**: `src/cli/inspect.rs` — Detailed run view

**Acceptance Criteria**:
- [ ] Show run metadata (state, attempts, timing)
- [ ] Show workflow YAML source
- [ ] Show step summaries (from transpiler result)
- [ ] Show artifacts list
- [ ] Show error details (if failed)
- [ ] Integration tests

---

### Task 4.5: Cancel Command (2 hours)

**Deliverable**: `src/cli/cancel.rs` — Cancel workflow

**Acceptance Criteria**:
- [ ] Cancel QUEUED/SCHEDULED runs (immediate)
- [ ] Cancel LEASED/RUNNING runs (signal transpiler, then transition)
- [ ] Validate state allows cancellation
- [ ] Audit log entry
- [ ] Integration tests

---

### Task 4.6: Retry Command (3 hours)

**Deliverable**: `src/cli/retry.rs` — Retry failed workflow

**Acceptance Criteria**:
- [ ] Retry FAILED runs (reset to QUEUED)
- [ ] Requeue DLQ runs (reset attempts, move to QUEUED)
- [ ] Optional attempt count reset
- [ ] Audit log entry
- [ ] Integration tests

---

### Task 4.7: Schedule Command (3 hours)

**Deliverable**: `src/cli/schedule.rs` — Schedule recurring workflow

**Acceptance Criteria**:
- [ ] Validate cron expression
- [ ] Create workflow + run with cron fields
- [ ] List active schedules
- [ ] Integration tests

---

### Task 4.8: Drain Command (2 hours)

**Deliverable**: `src/cli/drain.rs` — Drain queue

**Acceptance Criteria**:
- [ ] Cancel all non-terminal runs in a queue
- [ ] Confirmation prompt (unless `--yes`)
- [ ] Report how many runs were drained
- [ ] Integration tests

---

### Task 4.9: Validate Command (2 hours)

**Deliverable**: `src/cli/validate.rs` — Validate workflow

**Acceptance Criteria**:
- [ ] Call transpiler `validate` mode (per IN-15)
- [ ] Display validation errors with paths
- [ ] Exit code reflects validation status
- [ ] Integration tests

---

### Task 4.10: Queue-to-Transpiler Integration (6 hours)

**Deliverable**: Wire queue engine to transpiler for actual execution

**Acceptance Criteria**:
- [ ] Queue engine calls transpiler when run is LEASED
- [ ] Run transitions: LEASED → RUNNING → DONE/FAILED
- [ ] Heartbeat manager keeps lease alive during execution
- [ ] Environment variables resolved and passed
- [ ] Transpiler result parsed and artifacts metadata stored
- [ ] Error classification maps to retry/DLQ
- [ ] Integration tests with mock transpiler

---

### Task 4.11: Error Propagation (3 hours)

**Deliverable**: Unified error handling across CLI → queue → transpiler

**Acceptance Criteria**:
- [ ] Transpiler errors classified per IN-13
- [ ] Retryable errors → FAILED with retry schedule
- [ ] Non-retryable errors → DLQ
- [ ] Fatal errors → abort with clear message
- [ ] User-facing error messages (no stack traces)
- [ ] Structured error logs (full context)
- [ ] Integration tests

---

### Task 4.12: Cron Scheduler (6 hours)

**Deliverable**: `src/queue/cron.rs` — Cron-based scheduling

**Acceptance Criteria**:
- [ ] Parse cron expressions with timezone support
- [ ] Scheduler tick checks due scheduled runs
- [ ] Auto-create new run from cron workflow template
- [ ] Handle missed schedules (grace period)
- [ ] Integration tests

---

### Task 4.13: Daemon Mode (4 hours)

**Deliverable**: `src/cli/daemon.rs` — Long-running queue process

**Acceptance Criteria**:
- [ ] Tokio runtime with graceful shutdown (Ctrl+C)
- [ ] Scheduler tick loop
- [ ] Configurable worker count and max concurrent
- [ ] Signal handling (SIGTERM, SIGINT)
- [ ] Health check endpoint (optional)
- [ ] Integration tests

---

## Testing Strategy

### Unit Tests

| Test Area | Tests | Focus |
|-----------|--------|-------|
| CLI arg parsing | 5 | All subcommands, defaults, validation |
| Error formatting | 3 | Display, JSON, log formats |
| Cron parsing | 4 | Valid expressions, timezone, edge cases |
| Idempotency enforcement | 2 | Duplicate key rejection, race condition |
| Audit log entries | 4 | All state changes logged correctly |
| **Total** | **18** | |

### Integration Tests

| Test Case | Focus | Expected Outcome |
|-----------|--------|-----------------|
| Enqueue → execute → done | Full flow | Run reaches DONE, artifacts stored |
| Enqueue → fail → retry | Error flow | Run retries, eventually DLQ |
| Cancel queued run | Cancellation | Run transitions to CANCELED |
| Cancel running run | Force cancel | Run transitions to CANCELED |
| Schedule cron job | Cron | Run created at scheduled time |
| Validate invalid YAML | Validation | Errors displayed, non-zero exit |
| Validate valid YAML | Validation | Success message, zero exit |
| Drain queue | Drain | All runs canceled |
| Daemon startup/shutdown | Daemon | Graceful startup and shutdown |
| Idempotency key | Dedup | Second enqueue returns same run |
| Retry from DLQ | Recovery | Run requeued with reset attempts |
| List with filters | Query | Filtered results match criteria |
| Inspect with steps | Detail | Step summaries from transpiler result |
| Env var resolution | Configuration | Missing vars cause clear error |
| Backpressure | Admission | Queue rejects when full |
| Concurrent enqueue | Race | Exactly one enqueue succeeds |
| Error classification | Error mapping | Transpiler errors mapped correctly |
| Artifact collection | Artifacts | Metadata stored after execution |
| **Total** | **18** | |

---

## Success Criteria

**Phase complete when:**

- [ ] All 13 tasks complete and code reviewed
- [ ] All CLI commands functional
- [ ] Full enqueue → execute → done flow works
- [ ] Error handling end-to-end
- [ ] Cron scheduling works
- [ ] Daemon mode runs and shuts down cleanly

---

## Dependencies

This phase depends on:
- ✅ Phase 1: Foundation (32h) — Entities, state machine, persistence
- ✅ Phase 2: Queue Engine (36h) — Scheduling, leases, retry logic
- ✅ Phase 3: Transpiler Integration (27h) — CLI invocation, heartbeat, results

Enables:
- ✅ Phase 5: Testing + Verification (36h) — E2E tests need full CLI

---

## Effort Summary

| Task | Hours | Cumulative |
|------|--------|------------|
| CLI Framework | 3h | 3h |
| Enqueue Command | 4h | 7h |
| List Command | 4h | 11h |
| Inspect Command | 4h | 15h |
| Cancel Command | 2h | 17h |
| Retry Command | 3h | 20h |
| Schedule Command | 3h | 23h |
| Drain Command | 2h | 25h |
| Validate Command | 2h | 27h |
| Queue-to-Transpiler Integration | 6h | 33h |
| Error Propagation | 3h | 36h |
| Cron Scheduler | 6h | 42h |
| Daemon Mode | 4h | 46h |
| **Phase 4 Total** | **46h** | **141h** (Phase 1+2+3+4) |

---

**Next**: [Phase 5: Testing + Polish](../phases/05-testing-polish.md)
