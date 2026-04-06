# Phase 2: Queue Engine (Week 2)

## Goal

Implement the queue scheduling system including queue categories, priority ordering, lease management, retry logic with backoff, and backpressure control. This phase enables tasks to be scheduled, executed, and recovered from failures.

## Deliverables

| Task | File | Description |
|------|------|-------------|
| Queue categories | `src/queue/categories.rs` | Queue category definitions |
| Scheduler | `src/queue/scheduler.rs` | Task selection logic |
| Lease management | `src/queue/lease.rs` | Lease claiming, heartbeat, expiry |
| Retry logic | `src/queue/retry.rs` | Backoff + DLQ |
| Queue routing | `src/queue/routing.rs` | YAML to queue mapping |
| Backpressure | `src/queue/backpressure.rs` | Admission control |
| Audit log | `src/audit/log.rs` | Append-only audit trail |
| Engine core | `src/queue/engine.rs` | Scheduler tick loop |

---

## Task 2.1: Queue Categories (4h)

### File: `src/queue/categories.rs`

```rust
use crate::state::{Priority, QueueCategory};
use chrono::{DateTime, Utc};
use std::collections::{BinaryHeap, HashMap};

#[derive(Debug, Clone)]
pub struct TaskQueue {
    category: QueueCategory,
    tasks: BinaryHeap<QueueItem>,
    tasks_by_id: HashMap<String, QueueItem>,
}

#[derive(Debug, Clone, Eq, PartialEq)]
struct QueueItem {
    id: String,
    priority: Priority,
    enqueued_at: DateTime<Utc>,
    due_ts: Option<DateTime<Utc>>,
}

impl Ord for QueueItem {
    fn cmp(&self, other: &Self) -> std::cmp::Ordering {
        // Higher priority first
        match self.priority.cmp(&other.priority) {
            std::cmp::Ordering::Equal => {}
            ord => return ord.reverse(),
        }

        // If equal priority, older tasks first
        self.enqueued_at.cmp(&other.enqueued_at)
    }
}

impl PartialOrd for QueueItem {
    fn partial_cmp(&self, other: &Self) -> Option<std::cmp::Ordering> {
        Some(self.cmp(other))
    }
}

impl TaskQueue {
    pub fn new(category: QueueCategory) -> Self {
        Self {
            category,
            tasks: BinaryHeap::new(),
            tasks_by_id: HashMap::new(),
        }
    }

    pub fn enqueue(&mut self, item: QueueItem) {
        self.tasks_by_id.insert(item.id.clone(), item.clone());
        self.tasks.push(item);
    }

    pub fn dequeue(&mut self) -> Option<QueueItem> {
        if let Some(item) = self.tasks.pop() {
            self.tasks_by_id.remove(&item.id);
            Some(item)
        } else {
            None
        }
    }

    pub fn peek(&self) -> Option<&QueueItem> {
        self.tasks.peek()
    }

    pub fn len(&self) -> usize {
        self.tasks.len()
    }

    pub fn is_empty(&self) -> bool {
        self.tasks.is_empty()
    }
}

#[derive(Debug, Clone)]
pub struct TaskCandidate {
    pub id: String,
    pub category: QueueCategory,
    pub priority: Priority,
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_queue_ordering() {
        let mut queue = TaskQueue::new(QueueCategory::Asap);

        // Enqueue tasks with different priorities
        queue.enqueue(QueueItem {
            id: "task-3".to_string(),
            priority: Priority::Low,
            enqueued_at: Utc::now(),
            due_ts: None,
        });

        queue.enqueue(QueueItem {
            id: "task-1".to_string(),
            priority: Priority::Critical,
            enqueued_at: Utc::now(),
            due_ts: None,
        });

        queue.enqueue(QueueItem {
            id: "task-2".to_string(),
            priority: Priority::High,
            enqueued_at: Utc::now(),
            due_ts: None,
        });

        // Critical should come first
        let first = queue.dequeue().unwrap();
        assert_eq!(first.id, "task-1");
        assert_eq!(first.priority, Priority::Critical);

        // Then High
        let second = queue.dequeue().unwrap();
        assert_eq!(second.id, "task-2");
        assert_eq!(second.priority, Priority::High);

        // Then Low
        let third = queue.dequeue().unwrap();
        assert_eq!(third.id, "task-3");
        assert_eq!(third.priority, Priority::Low);
    }
}
```

### Acceptance Criteria

- [ ] Tasks ordered by priority (descending)
- [ ] FIFO tie-breaking for same priority
- [ ] Enqueue/dequeue operations are O(log n)
- [ ] Peek returns next task without removing

---

## Task 2.2: Scheduler (8h)

### File: `src/queue/scheduler.rs`

```rust
use crate::queue::categories::{TaskCandidate, TaskQueue};
use crate::state::{Priority, QueueCategory, Run, RunState};
use chrono::Utc;
use std::collections::HashMap;
use std::time::Duration;
use tracing::{debug, info, warn};

pub struct Scheduler {
    queues: HashMap<QueueCategory, TaskQueue>,
    max_in_flight: usize,
    current_in_flight: usize,
}

impl Scheduler {
    pub fn new(max_in_flight: usize) -> Self {
        let mut queues = HashMap::new();
        queues.insert(QueueCategory::Asap, TaskQueue::new(QueueCategory::Asap));
        queues.insert(QueueCategory::Whenever, TaskQueue::new(QueueCategory::Whenever));
        queues.insert(QueueCategory::Scheduled, TaskQueue::new(QueueCategory::Scheduled));
        queues.insert(QueueCategory::Cron, TaskQueue::new(QueueCategory::Cron));

        Self {
            queues,
            max_in_flight,
            current_in_flight: 0,
        }
    }

    pub fn select_next_task(&mut self) -> Option<TaskCandidate> {
        if self.current_in_flight >= self.max_in_flight {
            debug!("Backpressure: max in-flight limit reached");
            return None;
        }

        // Meta-scheduling: priority-weighted queue selection
        // Priority order: ASAP > Scheduled > Cron > Whenever
        let categories = vec![
            QueueCategory::Asap,
            QueueCategory::Scheduled,
            QueueCategory::Cron,
            QueueCategory::Whenever,
        ];

        for category in categories {
            if let Some(queue) = self.queues.get_mut(&category) {
                if let Some(item) = queue.dequeue() {
                    self.current_in_flight += 1;
                    info!(
                        category = %category,
                        task_id = %item.id,
                        priority = %item.priority,
                        "Task selected for execution"
                    );

                    return Some(TaskCandidate {
                        id: item.id,
                        category,
                        priority: item.priority,
                    });
                }
            }
        }

        debug!("No tasks available for scheduling");
        None
    }

    pub fn enqueue_task(&mut self, category: QueueCategory, run: &Run) {
        let queue = self.queues.get_mut(&category).unwrap();
        queue.enqueue(crate::queue::categories::QueueItem {
            id: run.id.clone(),
            priority: run.priority,
            enqueued_at: run.enqueued_at,
            due_ts: run.due_ts,
        });

        info!(
            category = %category,
            run_id = %run.id,
            priority = %run.priority,
            queue_depth = queue.len(),
            "Task enqueued"
        );
    }

    pub fn task_completed(&mut self) {
        if self.current_in_flight > 0 {
            self.current_in_flight -= 1;
            debug!(in_flight = self.current_in_flight, "Task completed");
        }
    }

    pub fn get_queue_depth(&self, category: QueueCategory) -> usize {
        self.queues.get(&category).map_or(0, |q| q.len())
    }
}

#[cfg(test)]
mod tests {
    use super::*;
    use chrono::Utc;

    #[test]
    fn test_scheduler_prioritizes_asap() {
        let mut scheduler = Scheduler::new(10);

        // Enqueue tasks in different categories
        let asap_run = Run::new("asap-1".to_string(), "wf-1".to_string(), QueueCategory::Asap, Priority::Normal);
        let whenever_run = Run::new("whenever-1".to_string(), "wf-1".to_string(), QueueCategory::Whenever, Priority::High);
        let scheduled_run = Run::new("scheduled-1".to_string(), "wf-1".to_string(), QueueCategory::Scheduled, Priority::Critical);

        scheduler.enqueue_task(QueueCategory::Whenever, &whenever_run);
        scheduler.enqueue_task(QueueCategory::Scheduled, &scheduled_run);
        scheduler.enqueue_task(QueueCategory::Asap, &asap_run);

        // ASAP should be selected first even with lower priority
        let selected = scheduler.select_next_task().unwrap();
        assert_eq!(selected.id, "asap-1");
        assert_eq!(selected.category, QueueCategory::Asap);
    }

    #[test]
    fn test_backpressure() {
        let mut scheduler = Scheduler::new(1);

        let run = Run::new("task-1".to_string(), "wf-1".to_string(), QueueCategory::Asap, Priority::High);
        scheduler.enqueue_task(QueueCategory::Asap, &run);

        // First task should be selected
        let first = scheduler.select_next_task().unwrap();
        assert_eq!(first.id, "task-1");

        // Second task should be blocked by backpressure
        let second = scheduler.select_next_task();
        assert!(second.is_none());
    }
}
```

### Acceptance Criteria

- [ ] ASAP tasks prioritized over others
- [ ] Backpressure prevents exceeding max_in_flight
- [ ] FIFO ordering within same category
- [ ] In-flight tracking is accurate

---

## Task 2.3: Lease Management (6h)

### File: `src/queue/lease.rs`

```rust
use crate::state::{Run, RunState};
use chrono::{DateTime, Duration as ChronoDuration, Utc};
use std::time::Duration;
use tracing::{debug, info, warn};

#[derive(Debug, Clone)]
pub struct LeaseConfig {
    pub lease_ttl: Duration,
    pub heartbeat_interval: Duration,
    pub max_missed_heartbeats: u32,
    pub reclaim_delay: Duration,
}

impl Default for LeaseConfig {
    fn default() -> Self {
        Self {
            lease_ttl: Duration::from_secs(60),
            heartbeat_interval: Duration::from_secs(15),
            max_missed_heartbeats: 2,
            reclaim_delay: Duration::from_secs(5),
        }
    }
}

pub struct LeaseManager {
    config: LeaseConfig,
}

impl LeaseManager {
    pub fn new(config: LeaseConfig) -> Self {
        Self { config }
    }

    pub fn claim_lease(&self, run_id: &str, agent_id: &str, run: &mut Run) -> Result<(), LeaseError> {
        if run.state != RunState::Queued {
            return Err(LeaseError::NotLeaseable(run.state));
        }

        let lease_expiry = Utc::now() + ChronoDuration::from_std(self.config.lease_ttl)?;

        run.state = RunState::Leased;
        run.lease_agent_id = Some(agent_id.to_string());
        run.lease_expiry = Some(lease_expiry);
        run.last_heartbeat = Some(Utc::now());

        info!(
            run_id = %run_id,
            agent_id = %agent_id,
            lease_expiry = %lease_expiry.to_rfc3339(),
            "Lease claimed"
        );

        Ok(())
    }

    pub fn update_heartbeat(&self, run_id: &str, run: &mut Run) -> Result<(), LeaseError> {
        if !matches!(run.state, RunState::Leased | RunState::Running) {
            return Err(LeaseError::NotHeartbeatEligible(run.state));
        }

        let new_lease_expiry = Utc::now() + ChronoDuration::from_std(self.config.lease_ttl)?;

        run.last_heartbeat = Some(Utc::now());
        run.lease_expiry = Some(new_lease_expiry);

        debug!(
            run_id = %run_id,
            new_lease_expiry = %new_lease_expiry.to_rfc3339(),
            "Heartbeat updated"
        );

        Ok(())
    }

    pub fn check_expiry(&self, run: &Run) -> bool {
        if let (Some(lease_expiry), Some(last_heartbeat)) = (run.lease_expiry, run.last_heartbeat) {
            let now = Utc::now();
            let heartbeat_age = now - last_heartbeat;
            let lease_age = now - lease_expiry;

            // Consider expired if:
            // 1. Lease time has passed
            // 2. Heartbeat is too old (max_missed_heartbeats * interval)
            let heartbeat_deadline = ChronoDuration::from_std(self.config.heartbeat_interval)
                .unwrap()
                .mul(self.config.max_missed_heartbeats);

            if lease_age > ChronoDuration::seconds(0) || heartbeat_age > heartbeat_deadline {
                warn!(
                    run_id = %run.id,
                    lease_age = %lease_age.num_seconds(),
                    heartbeat_age = %heartbeat_age.num_seconds(),
                    "Lease expired"
                );
                return true;
            }
        }
        false
    }

    pub fn reclaim_lease(&self, run_id: &str, run: &mut Run) {
        info!(
            run_id = %run_id,
            previous_agent = %run.lease_agent_id.as_deref().unwrap_or("none"),
            "Reclaiming expired lease"
        );

        run.state = RunState::Queued;
        run.lease_agent_id = None;
        run.lease_expiry = None;
        run.last_heartbeat = None;
    }
}

#[derive(Debug, thiserror::Error)]
pub enum LeaseError {
    #[error("run not leaseable: state is {0}")]
    NotLeaseable(RunState),

    #[error("heartbeat not eligible: state is {0}")]
    NotHeartbeatEligible(RunState),

    #[error("lease already expired")]
    AlreadyExpired,
}
```

### Acceptance Criteria

- [ ] Lease claimed only from QUEUED state
- [ ] Heartbeat extends lease expiry
- [ ] Expired leases are detected
- [ ] Expired leases are reclaimed to QUEUED

---

## Task 2.4: Retry Logic + DLQ (4h)

### File: `src/queue/retry.rs`

```rust
use crate::state::{Run, RunState};
use chrono::Utc;
use rand::Rng;
use std::time::Duration;
use tracing::{debug, info, warn};

#[derive(Debug, Clone)]
pub struct RetryConfig {
    pub max_attempts: u32,
    pub base_delay_ms: u64,
    pub max_delay_ms: u64,
    pub backoff_multiplier: f32,
    pub jitter_factor: f32,
}

impl Default for RetryConfig {
    fn default() -> Self {
        Self {
            max_attempts: 3,
            base_delay_ms: 1000,
            max_delay_ms: 30000,
            backoff_multiplier: 2.0,
            jitter_factor: 0.2,
        }
    }
}

pub struct RetryManager {
    config: RetryConfig,
}

impl RetryManager {
    pub fn new(config: RetryConfig) -> Self {
        Self { config }
    }

    pub fn should_retry(&self, run: &Run) -> bool {
        run.attempts < run.max_attempts
    }

    pub fn should_dlq(&self, run: &Run) -> bool {
        run.attempts >= run.max_attempts
    }

    pub fn calculate_retry_delay(&self, attempt: u32) -> Duration {
        let base_delay = Duration::from_millis(self.config.base_delay_ms);
        let max_delay = Duration::from_millis(self.config.max_delay_ms);

        // Exponential backoff
        let multiplier = self.config.backoff_multiplier.powi(attempt as i32 - 1);
        let delay_ms = base_delay.as_millis() as f64 * multiplier;

        // Cap at max delay
        let delay_ms = delay_ms.min(max_delay.as_millis() as f64);

        // Add jitter
        let jitter = delay_ms * self.config.jitter_factor;
        let delay_with_jitter = delay_ms + (rand::thread_rng().gen::<f64>() - 0.5) * 2.0 * jitter;

        Duration::from_millis(delay_with_jitter as u64)
    }

    pub fn handle_failure(&self, run_id: &str, run: &mut Run, error: &str) -> RetryAction {
        let new_attempts = run.attempts + 1;
        run.attempts = new_attempts;
        run.error_message = Some(error.to_string());

        if new_attempts >= run.max_attempts {
            warn!(
                run_id = %run_id,
                attempts = new_attempts,
                max_attempts = run.max_attempts,
                "Max retries exceeded, moving to DLQ"
            );

            run.state = RunState::Dlq;
            RetryAction::MoveToDlq
        } else {
            let delay = self.calculate_retry_delay(new_attempts);
            info!(
                run_id = %run_id,
                attempt = new_attempts,
                max_attempts = run.max_attempts,
                delay_ms = delay.as_millis(),
                error = %error,
                "Scheduling retry"
            );

            run.state = RunState::Failed;
            RetryAction::Retry(delay)
        }
    }
}

#[derive(Debug)]
pub enum RetryAction {
    Retry(Duration),
    MoveToDlq,
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_retry_delays() {
        let manager = RetryManager::new(RetryConfig {
            base_delay_ms: 1000,
            max_delay_ms: 30000,
            backoff_multiplier: 2.0,
            jitter_factor: 0.0,  // No jitter for testing
            max_attempts: 3,
        });

        let delay1 = manager.calculate_retry_delay(1);
        assert!(delay1.as_millis() >= 900 && delay1.as_millis() <= 1100);

        let delay2 = manager.calculate_retry_delay(2);
        assert!(delay2.as_millis() >= 1900 && delay2.as_millis() <= 2100);

        let delay3 = manager.calculate_retry_delay(3);
        assert!(delay3.as_millis() >= 3900 && delay3.as_millis() <= 4100);
    }

    #[test]
    fn test_dlq_decision() {
        let manager = RetryManager::new(RetryConfig::default());

        let mut run = Run::new(
            "test-run".to_string(),
            "test-wf".to_string(),
            crate::state::QueueCategory::Asap,
            crate::state::Priority::Normal,
        );
        run.attempts = 0;
        run.max_attempts = 3;

        // First failure should retry
        let action = manager.handle_failure("test-run", &mut run, "test error");
        assert!(matches!(action, RetryAction::Retry(_)));
        assert_eq!(run.state, RunState::Failed);

        // Exhausted retries should go to DLQ
        run.attempts = 3;
        let action = manager.handle_failure("test-run", &mut run, "test error");
        assert!(matches!(action, RetryAction::MoveToDlq));
        assert_eq!(run.state, RunState::Dlq);
    }
}
```

### Acceptance Criteria

- [ ] Retry delays increase exponentially
- [ ] Jitter is applied to prevent thundering herd
- [ ] Max retries respected
- [ ] Exhausted retries move to DLQ

---

## Task 2.5: Queue Routing (3h)

### File: `src/queue/routing.rs`

```rust
use crate::state::{Priority, QueueCategory};
use crate::workflow::schema::{QueueConfig, WorkflowSchema};
use tracing::debug;

pub struct Router;

impl Router {
    pub fn route_workflow(workflow: &WorkflowSchema) -> (QueueCategory, Priority) {
        let category = match workflow.queue.category.as_str() {
            "asap" => QueueCategory::Asap,
            "whenever" => QueueCategory::Whenever,
            "scheduled" => QueueCategory::Scheduled,
            "cron" => QueueCategory::Cron,
            _ => {
                debug!(
                    category = %workflow.queue.category,
                    "Unknown queue category, defaulting to whenever"
                );
                QueueCategory::Whenever
            }
        };

        let priority = match workflow.queue.priority.as_deref() {
            Some("critical") => Priority::Critical,
            Some("high") => Priority::High,
            Some("normal") => Priority::Normal,
            Some("low") => Priority::Low,
            Some(other) => {
                debug!(
                    priority = %other,
                    "Unknown priority, defaulting to normal"
                );
                Priority::Normal
            }
            None => Priority::Normal,
        };

        debug!(
            workflow = %workflow.name,
            category = %category,
            priority = %priority,
            "Routed workflow"
        );

        (category, priority)
    }
}

#[cfg(test)]
mod tests {
    use super::*;
    use crate::workflow::schema::ScheduleConfig;

    #[test]
    fn test_routing_asap() {
        let workflow = WorkflowSchema {
            schema_version: "0.1.0".to_string(),
            name: "test".to_string(),
            description: None,
            version: None,
            queue: QueueConfig {
                category: "asap".to_string(),
                priority: Some("critical".to_string()),
                schedule: None,
            },
            retry: None,
            model: crate::workflow::schema::ModelConfig {
                provider: "llama-cpp".to_string(),
                model_path: "/test/model.gguf".to_string(),
                temperature: None,
                max_tokens: None,
                context_budget: None,
            },
            tools: crate::workflow::schema::ToolsConfig {
                allowed: vec![],
                blocked: vec![],
            },
            env: None,
            steps: vec![],
        };

        let (category, priority) = Router::route_workflow(&workflow);
        assert_eq!(category, QueueCategory::Asap);
        assert_eq!(priority, Priority::Critical);
    }

    #[test]
    fn test_routing_defaults() {
        let workflow = WorkflowSchema {
            schema_version: "0.1.0".to_string(),
            name: "test".to_string(),
            description: None,
            version: None,
            queue: QueueConfig {
                category: "whenever".to_string(),
                priority: None,
                schedule: None,
            },
            retry: None,
            model: crate::workflow::schema::ModelConfig {
                provider: "llama-cpp".to_string(),
                model_path: "/test/model.gguf".to_string(),
                temperature: None,
                max_tokens: None,
                context_budget: None,
            },
            tools: crate::workflow::schema::ToolsConfig {
                allowed: vec![],
                blocked: vec![],
            },
            env: None,
            steps: vec![],
        };

        let (category, priority) = Router::route_workflow(&workflow);
        assert_eq!(category, QueueCategory::Whenever);
        assert_eq!(priority, Priority::Normal);
    }
}
```

### Acceptance Criteria

- [ ] YAML queue category maps correctly
- [ ] YAML priority maps correctly
- [ ] Unknown values default to safe values

---

## Task 2.6: Backpressure (3h)

### File: `src/queue/backpressure.rs`

```rust
use crate::queue::categories::TaskQueue;
use crate::state::{QueueCategory, Run};
use tracing::{debug, warn};

pub struct BackpressureConfig {
    pub max_queue_depth: usize,
    pub reject_when_full: bool,
}

impl Default for BackpressureConfig {
    fn default() -> Self {
        Self {
            max_queue_depth: 1000,
            reject_when_full: true,
        }
    }
}

pub struct Backpressure {
    config: BackpressureConfig,
}

impl Backpressure {
    pub fn new(config: BackpressureConfig) -> Self {
        Self { config }
    }

    pub fn check_admission(
        &self,
        category: QueueCategory,
        current_depth: usize,
    ) -> Result<(), BackpressureError> {
        if current_depth >= self.config.max_queue_depth {
            if self.config.reject_when_full {
                warn!(
                    category = %category,
                    depth = current_depth,
                    max = self.config.max_queue_depth,
                    "Queue full, rejecting task"
                );
                return Err(BackpressureError::QueueFull {
                    category,
                    current_depth,
                    max_depth: self.config.max_queue_depth,
                });
            } else {
                debug!(
                    category = %category,
                    depth = current_depth,
                    max = self.config.max_queue_depth,
                    "Queue full, but accept configured"
                );
            }
        }

        Ok(())
    }
}

#[derive(Debug, thiserror::Error)]
pub enum BackpressureError {
    #[error("queue {category} is full: {current_depth}/{max_depth}")]
    QueueFull {
        category: QueueCategory,
        current_depth: usize,
        max_depth: usize,
    },
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_admission_allowed() {
        let bp = Backpressure::new(BackpressureConfig {
            max_queue_depth: 10,
            reject_when_full: true,
        });

        assert!(bp.check_admission(QueueCategory::Asap, 5).is_ok());
    }

    #[test]
    fn test_admission_rejected() {
        let bp = Backpressure::new(BackpressureConfig {
            max_queue_depth: 10,
            reject_when_full: true,
        });

        let result = bp.check_admission(QueueCategory::Asap, 10);
        assert!(result.is_err());
        match result.unwrap_err() {
            BackpressureError::QueueFull { current_depth, .. } => {
                assert_eq!(current_depth, 10);
            }
        }
    }
}
```

### Acceptance Criteria

- [ ] Rejects tasks when queue is full
- [ ] Can be configured to accept when full
- [ ] Reports current queue depth

---

## Task 2.7: Audit Log (2h)

### File: `src/audit/log.rs`

```rust
use crate::state::AuditLog;
use crate::store::SqliteStore;
use tracing::debug;

pub struct AuditLogger {
    store: SqliteStore,
}

impl AuditLogger {
    pub fn new(store: SqliteStore) -> Self {
        Self { store }
    }

    pub async fn log(&self, entry: AuditLog) -> Result<(), AuditError> {
        debug!(
            actor = %entry.actor,
            action = %entry.action,
            entity_type = %entry.entity_type,
            entity_id = %entry.entity_id,
            "Logging audit entry"
        );

        // Use insert_audit_log from store
        // self.store.insert_audit_log(entry).await?;
        Ok(())
    }
}

#[derive(Debug, thiserror::Error)]
pub enum AuditError {
    #[error("store error: {0}")]
    Store(#[from] crate::store::StoreError),
}
```

### Acceptance Criteria

- [ ] All state changes are logged
- [ ] Audit log is append-only
- [ ] Actor is tracked

---

## Task 2.8: Engine Core (Scheduler Tick Loop) (4h)

### File: `src/queue/engine.rs`

```rust
use crate::audit::log::AuditLogger;
use crate::queue::backpressure::Backpressure;
use crate::queue::categories::TaskCandidate;
use crate::queue::lease::{LeaseConfig, LeaseManager};
use crate::queue::retry::RetryManager;
use crate::queue::routing::Router;
use crate::queue::scheduler::Scheduler;
use crate::state::{Run, RunState};
use crate::store::SqliteStore;
use crate::workflow::schema::WorkflowSchema;
use crate::workflow::parser::WorkflowParser;
use chrono::Utc;
use std::time::Duration;
use tracing::{debug, info, warn};

pub struct QueueEngine {
    store: SqliteStore,
    scheduler: Scheduler,
    lease_manager: LeaseManager,
    retry_manager: RetryManager,
    backpressure: Backpressure,
    audit_logger: AuditLogger,
    agent_id: String,
}

impl QueueEngine {
    pub fn new(store: SqliteStore, agent_id: String) -> Self {
        Self {
            scheduler: Scheduler::new(10),  // max_in_flight = 10
            lease_manager: LeaseManager::new(LeaseConfig::default()),
            retry_manager: RetryManager::new(RetryConfig::default()),
            backpressure: Backpressure::new(BackpressureConfig::default()),
            audit_logger: AuditLogger::new(store.clone()),
            store,
            agent_id,
        }
    }

    pub async fn enqueue_workflow<P: AsRef<std::path::Path>>(
        &self,
        workflow_path: P,
    ) -> Result<String, EngineError> {
        // Parse workflow
        let workflow = WorkflowParser::parse(workflow_path)?;

        // Route to queue
        let (category, priority) = Router::route_workflow(&workflow);

        // Check backpressure
        let depth = self.scheduler.get_queue_depth(category);
        self.backpressure
            .check_admission(category, depth)
            .map_err(|e| EngineError::Backpressure(e))?;

        // Create run
        let run_id = format!("run-{}", uuid::Uuid::new_v4());
        let mut run = Run::new(run_id.clone(), workflow.name.clone(), category, priority);

        // Store workflow
        // self.store.insert_workflow(&workflow).await?;

        // Store run
        // let tx = self.store.begin_transaction().await?;
        // tx.insert_run(&run).await?;
        // tx.commit().await?;

        info!(
            run_id = %run_id,
            workflow = %workflow.name,
            category = %category,
            priority = %priority,
            "Workflow enqueued"
        );

        Ok(run_id)
    }

    pub async fn scheduler_tick(&mut self) -> Result<(), EngineError> {
        debug!("Scheduler tick");

        // 1. Check scheduled tasks - promote if due_ts <= now
        let scheduled_runs = self
            .store
            .get_runs_by_state(RunState::Scheduled)
            .await?;
        let now = Utc::now();

        for mut run in scheduled_runs {
            if let Some(due_ts) = run.due_ts {
                if due_ts <= now {
                    info!(run_id = %run.id, "Promoting scheduled task to queued");
                    run.state = RunState::Queued;
                    run.due_ts = None;
                    // self.store.update_run(&run).await?;
                    self.scheduler.enqueue_task(RunState::Queued, &run);
                }
            }
        }

        // 2. Check expired leases - reclaim to QUEUED
        let leased_runs = self
            .store
            .get_runs_by_state(RunState::Leased)
            .await?;

        for mut run in leased_runs {
            if self.lease_manager.check_expiry(&run) {
                warn!(run_id = %run.id, "Reclaiming expired lease");
                self.lease_manager.reclaim_lease(&run.id, &mut run);
                // self.store.update_run(&run).await?;
                self.scheduler.enqueue_task(RunState::Queued, &run);
            }
        }

        // 3. Reap DLQ candidates - move FAILED tasks with attempts >= max
        let failed_runs = self
            .store
            .get_runs_by_state(RunState::Failed)
            .await?;

        for mut run in failed_runs {
            if self.retry_manager.should_dlq(&run) {
                warn!(
                    run_id = %run.id,
                    attempts = run.attempts,
                    "Moving to DLQ"
                );
                run.state = RunState::Dlq;
                // self.store.update_run(&run).await?;
            }
        }

        // 4. Select next task
        if let Some(candidate) = self.scheduler.select_next_task() {
            debug!(run_id = %candidate.id, "Selecting task for execution");

            // Claim lease
            let mut run = self.store.get_run(&candidate.id).await?.unwrap();
            self.lease_manager
                .claim_lease(&candidate.id, &self.agent_id, &mut run)?;
            // self.store.update_run(&run).await?;

            // Dispatch to agent (TODO: integrate with agent executor)
            info!(
                run_id = %candidate.id,
                "Task dispatched to agent"
            );
        }

        Ok(())
    }

    pub async fn run_scheduler_loop(&mut self) -> Result<(), EngineError> {
        let mut interval = tokio::time::interval(Duration::from_secs(1));

        loop {
            interval.tick().await;
            self.scheduler_tick().await?;
        }
    }
}

#[derive(Debug, thiserror::Error)]
pub enum EngineError {
    #[error("validation error: {0}")]
    Validation(#[from] crate::workflow::parser::ValidationError),

    #[error("backpressure error: {0}")]
    Backpressure(#[from] crate::queue::backpressure::BackpressureError),

    #[error("store error: {0}")]
    Store(#[from] crate::store::StoreError),

    #[error("lease error: {0}")]
    Lease(#[from] crate::queue::lease::LeaseError),
}
```

### Acceptance Criteria

- [ ] Scheduler tick runs every second
- [ ] Scheduled tasks promoted when due
- [ ] Expired leases reclaimed
- [ ] Failed tasks moved to DLQ
- [ ] Tasks dispatched to agent

---

## Phase 2 Completion Checklist

- [ ] Queue categories implemented
- [ ] Scheduler with meta-scheduling working
- [ ] Lease management functional
- [ ] Retry logic with backoff
- [ ] DLQ management
- [ ] Queue routing from YAML
- [ ] Backpressure control
- [ ] Audit logging
- [ ] Scheduler tick loop running
- [ ] All unit tests pass
- [ ] Integration tests pass

**Phase 2 Estimated Time**: 36 hours

**Phase 2 Deliverable**: Functional queue engine that schedules and manages task lifecycle
