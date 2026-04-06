# State Machine Implementation

## State Transitions

### Complete Transition Graph

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

## Valid Transitions

### From NEW

```rust
impl RunState {
    pub fn from_new(self) -> Result<RunState, StateError> {
        match self {
            RunState::Queued => Ok(RunState::Queued),
            RunState::Canceled => Ok(RunState::Canceled),
            _ => Err(StateError::InvalidTransition {
                from: RunState::New,
                to: self,
            }),
        }
    }
}
```

**Valid**: NEW → QUEUED, NEW → CANCELED

### From QUEUED

```rust
impl RunState {
    pub fn from_queued(self) -> Result<RunState, StateError> {
        match self {
            RunState::Leased => Ok(RunState::Leased),
            RunState::Scheduled => Ok(RunState::Scheduled),
            RunState::Canceled => Ok(RunState::Canceled),
            _ => Err(StateError::InvalidTransition {
                from: RunState::Queued,
                to: self,
            }),
        }
    }
}
```

**Valid**: QUEUED → LEASED, QUEUED → SCHEDULED, QUEUED → CANCELED

### From SCHEDULED

```rust
impl RunState {
    pub fn from_scheduled(self) -> Result<RunState, StateError> {
        match self {
            RunState::Queued => Ok(RunState::Queued),  // Time gate passed
            RunState::Canceled => Ok(RunState::Canceled),
            _ => Err(StateError::InvalidTransition {
                from: RunState::Scheduled,
                to: self,
            }),
        }
    }
}
```

**Valid**: SCHEDULED → QUEUED, SCHEDULED → CANCELED

### From LEASED

```rust
impl RunState {
    pub fn from_leased(self) -> Result<RunState, StateError> {
        match self {
            RunState::Running => Ok(RunState::Running),
            RunState::Expired => Ok(RunState::Expired),
            RunState::Queued => Ok(RunState::Queued),  // Lease expired, reclaimed
            RunState::Canceled => Ok(RunState::Canceled),
            _ => Err(StateError::InvalidTransition {
                from: RunState::Leased,
                to: self,
            }),
        }
    }
}
```

**Valid**: LEASED → RUNNING, LEASED → EXPIRED, LEASED → QUEUED, LEASED → CANCELED

### From RUNNING

```rust
impl RunState {
    pub fn from_running(self) -> Result<RunState, StateError> {
        match self {
            RunState::Done => Ok(RunState::Done),
            RunState::Failed => Ok(RunState::Failed),
            RunState::Canceled => Ok(RunState::Canceled),
            RunState::Expired => Ok(RunState::Expired),  // Lease timeout
            _ => Err(StateError::InvalidTransition {
                from: RunState::Running,
                to: self,
            }),
        }
    }
}
```

**Valid**: RUNNING → DONE, RUNNING → FAILED, RUNNING → CANCELED, RUNNING → EXPIRED

### From FAILED

```rust
impl RunState {
    pub fn from_failed(self) -> Result<RunState, StateError> {
        match self {
            RunState::Leased => Ok(RunState::Leased),  // Retry
            RunState::Dlq => Ok(RunState::Dlq),      // Exhausted retries
            RunState::Queued => Ok(RunState::Queued),  // Manual requeue
            RunState::Canceled => Ok(RunState::Canceled),
            _ => Err(StateError::InvalidTransition {
                from: RunState::Failed,
                to: self,
            }),
        }
    }
}
```

**Valid**: FAILED → LEASED (retry), FAILED → DLQ, FAILED → QUEUED (manual), FAILED → CANCELED

### From EXPIRED

```rust
impl RunState {
    pub fn from_expired(self) -> Result<RunState, StateError> {
        match self {
            RunState::Queued => Ok(RunState::Queued),  // Reclaimed
            RunState::Leased => Ok(RunState::Leased),  // Re-leased
            RunState::Failed => Ok(RunState::Failed),  // Expired during run
            RunState::Canceled => Ok(RunState::Canceled),
            _ => Err(StateError::InvalidTransition {
                from: RunState::Expired,
                to: self,
            }),
        }
    }
}
```

**Valid**: EXPIRED → QUEUED, EXPIRED → LEASED, EXPIRED → FAILED, EXPIRED → CANCELED

### Terminal States

No valid transitions from:
- DONE
- DLQ
- CANCELED

## State Machine Implementation

```rust
use thiserror::Error;

#[derive(Debug, Error)]
pub enum StateError {
    #[error("invalid state transition: {from} → {to}")]
    InvalidTransition { from: RunState, to: RunState },

    #[error("state {0} is terminal, cannot transition")]
    TerminalState(RunState),
}

impl RunState {
    /// Check if state is terminal (no valid transitions)
    pub fn is_terminal(self) -> bool {
        matches!(self, RunState::Done | RunState::Dlq | RunState::Canceled)
    }

    /// Check if state is active (can transition)
    pub fn is_active(self) -> bool {
        !self.is_terminal()
    }

    /// Transition to new state
    pub fn transition_to(self, new_state: RunState) -> Result<RunState, StateError> {
        if self.is_terminal() {
            return Err(StateError::TerminalState(self));
        }

        match self {
            RunState::New => new_state.from_new(),
            RunState::Queued => new_state.from_queued(),
            RunState::Scheduled => new_state.from_scheduled(),
            RunState::Leased => new_state.from_leased(),
            RunState::Running => new_state.from_running(),
            RunState::Failed => new_state.from_failed(),
            RunState::Expired => new_state.from_expired(),
            RunState::Done | RunState::Dlq | RunState::Canceled => {
                Err(StateError::TerminalState(self))
            }
        }
    }
}
```

## State Machine Tests

### Unit Test Structure

```rust
#[cfg(test)]
mod tests {
    use super::*;

    fn assert_valid_transition(from: RunState, to: RunState) {
        assert!(from.transition_to(to).is_ok());
    }

    fn assert_invalid_transition(from: RunState, to: RunState) {
        assert!(from.transition_to(to).is_err());
    }

    #[test]
    fn test_new_transitions() {
        assert_valid_transition(RunState::New, RunState::Queued);
        assert_valid_transition(RunState::New, RunState::Canceled);

        assert_invalid_transition(RunState::New, RunState::Running);
        assert_invalid_transition(RunState::New, RunState::Done);
    }

    #[test]
    fn test_queued_transitions() {
        assert_valid_transition(RunState::Queued, RunState::Leased);
        assert_valid_transition(RunState::Queued, RunState::Scheduled);
        assert_valid_transition(RunState::Queued, RunState::Canceled);

        assert_invalid_transition(RunState::Queued, RunState::Running);
        assert_invalid_transition(RunState::Queued, RunState::Done);
    }

    #[test]
    fn test_leased_transitions() {
        assert_valid_transition(RunState::Leased, RunState::Running);
        assert_valid_transition(RunState::Leased, RunState::Expired);
        assert_valid_transition(RunState::Leased, RunState::Queued);
        assert_valid_transition(RunState::Leased, RunState::Canceled);

        assert_invalid_transition(RunState::Leased, RunState::Done);
        assert_invalid_transition(RunState::Leased, RunState::Failed);
    }

    #[test]
    fn test_running_transitions() {
        assert_valid_transition(RunState::Running, RunState::Done);
        assert_valid_transition(RunState::Running, RunState::Failed);
        assert_valid_transition(RunState::Running, RunState::Canceled);
        assert_valid_transition(RunState::Running, RunState::Expired);

        assert_invalid_transition(RunState::Running, RunState::Queued);
        assert_invalid_transition(RunState::Running, RunState::Leased);
    }

    #[test]
    fn test_failed_transitions() {
        assert_valid_transition(RunState::Failed, RunState::Leased);
        assert_valid_transition(RunState::Failed, RunState::Dlq);
        assert_valid_transition(RunState::Failed, RunState::Queued);
        assert_valid_transition(RunState::Failed, RunState::Canceled);

        assert_invalid_transition(RunState::Failed, RunState::Running);
        assert_invalid_transition(RunState::Failed, RunState::Done);
    }

    #[test]
    fn test_terminal_states() {
        assert!(RunState::Done.is_terminal());
        assert!(RunState::Dlq.is_terminal());
        assert!(RunState::Canceled.is_terminal());

        assert!(!RunState::New.is_terminal());
        assert!(!RunState::Queued.is_terminal());
        assert!(!RunState::Running.is_terminal());
    }

    #[test]
    fn test_no_transitions_from_terminal() {
        assert_invalid_transition(RunState::Done, RunState::Queued);
        assert_invalid_transition(RunState::Dlq, RunState::Leased);
        assert_invalid_transition(RunState::Canceled, RunState::Running);
    }
}
```

## Transition Side Effects

### Enqueue Side Effects

```rust
pub struct StateMachine {
    db: Arc<SqliteStore>,
}

impl StateMachine {
    pub async fn enqueue(&self, workflow_id: &str, run: &Run) -> Result<(), Error> {
        let tx = self.db.begin_transaction().await?;

        // 1. Validate workflow exists
        let workflow = tx.get_workflow(workflow_id).await?;
        if workflow.is_none() {
            return Err(Error::WorkflowNotFound(workflow_id.to_string()));
        }

        // 2. Create run in NEW state
        tx.insert_run(run).await?;

        // 3. Audit log
        tx.insert_audit_log(&AuditLog {
            id: 0,
            ts: Utc::now(),
            actor: "cli".to_string(),
            action: "enqueue".to_string(),
            entity_type: "run".to_string(),
            entity_id: run.id.clone(),
            prev_state: None,
            new_state: Some("new".to_string()),
            diff_json: None,
            reason: None,
        }).await?;

        // 4. Transition to QUEUED (validation successful)
        self.transition_to(&run.id, RunState::Queued, Some("validation_successful")).await?;

        tx.commit().await?;
        Ok(())
    }
}
```

### Lease Side Effects

```rust
impl StateMachine {
    pub async fn lease(&self, run_id: &str, agent_id: &str, ttl: Duration) -> Result<(), Error> {
        let tx = self.db.begin_transaction().await?;

        // 1. Get current state
        let run = tx.get_run(run_id).await?
            .ok_or_else(|| Error::RunNotFound(run_id.to_string()))?;

        // 2. Check leaseable
        if !matches!(run.state, RunState::Queued) {
            return Err(Error::NotLeaseable(run.state));
        }

        // 3. Update lease
        let lease_expiry = Utc::now() + chrono::Duration::from_std(ttl)?;
        tx.update_lease(run_id, agent_id, lease_expiry).await?;

        // 4. Transition to LEASED
        self.transition_to(run_id, RunState::Leased, Some(format!("leased by {}", agent_id))).await?;

        tx.commit().await?;
        Ok(())
    }
}
```

### Heartbeat Side Effects

```rust
impl StateMachine {
    pub async fn heartbeat(&self, run_id: &str) -> Result<(), Error> {
        let tx = self.db.begin_transaction().await?;

        // 1. Get current state
        let run = tx.get_run(run_id).await?
            .ok_or_else(|| Error::RunNotFound(run_id.to_string()))?;

        // 2. Check heartbeat-eligible
        if !matches!(run.state, RunState::Leased | RunState::Running) {
            return Err(Error::NotHeartbeatEligible(run.state));
        }

        // 3. Update heartbeat and extend lease
        let new_lease_expiry = Utc::now() + chrono::Duration::seconds(60);
        tx.update_heartbeat(run_id, new_lease_expiry).await?;

        // 4. No state transition needed
        //    Just audit log for tracking
        tx.insert_audit_log(&AuditLog {
            id: 0,
            ts: Utc::now(),
            actor: "agent".to_string(),
            action: "heartbeat".to_string(),
            entity_type: "run".to_string(),
            entity_id: run_id.to_string(),
            prev_state: None,
            new_state: None,
            diff_json: Some(json!({"new_expiry": new_lease_expiry}).to_string()),
            reason: None,
        }).await?;

        tx.commit().await?;
        Ok(())
    }
}
```

### Complete Side Effects

```rust
impl StateMachine {
    pub async fn complete(&self, run_id: &str) -> Result<(), Error> {
        let tx = self.db.begin_transaction().await?;

        // 1. Get current state
        let run = tx.get_run(run_id).await?
            .ok_or_else(|| Error::RunNotFound(run_id.to_string()))?;

        // 2. Check completable
        if !matches!(run.state, RunState::Running) {
            return Err(Error::NotCompletable(run.state));
        }

        // 3. Update completion timestamp
        tx.update_completion(run_id, Utc::now()).await?;

        // 4. Transition to DONE
        self.transition_to(run_id, RunState::Done, Some("execution_successful")).await?;

        tx.commit().await?;
        Ok(())
    }
}
```

### Fail Side Effects

```rust
impl StateMachine {
    pub async fn fail(&self, run_id: &str, error: &str, step_id: Option<&str>) -> Result<(), Error> {
        let tx = self.db.begin_transaction().await?;

        // 1. Get current state
        let run = tx.get_run(run_id).await?
            .ok_or_else(|| Error::RunNotFound(run_id.to_string()))?;

        // 2. Check failable
        if !matches!(run.state, RunState::Running) {
            return Err(Error::NotFailable(run.state));
        }

        // 3. Update error info
        tx.update_error(run_id, error, step_id).await?;

        // 4. Increment attempts
        let new_attempts = run.attempts + 1;
        tx.increment_attempts(run_id).await?;

        // 5. Check retry exhausted
        if new_attempts >= run.max_attempts {
            // Move to DLQ
            self.transition_to(run_id, RunState::Dlq, Some("exhausted_retries")).await?;
        } else {
            // Move to FAILED (will retry)
            self.transition_to(run_id, RunState::Failed, Some(error)).await?;

            // Schedule retry
            let delay = self.calculate_retry_delay(new_attempts);
            self.schedule_retry(run_id, delay).await?;
        }

        tx.commit().await?;
        Ok(())
    }

    fn calculate_retry_delay(&self, attempt: u32) -> Duration {
        let base_delay = Duration::from_secs(1);
        let max_delay = Duration::from_secs(30);
        let multiplier = 2_f32.powi(attempt as i32 - 1);
        let delay_ms = (base_delay.as_millis() as f64 * multiplier).min(max_delay.as_millis() as f64);

        // Add jitter
        let jitter = delay_ms * 0.2;  // 20% jitter
        let delay_with_jitter = delay_ms + (rand::random::<f64>() - 0.5) * 2.0 * jitter;

        Duration::from_millis(delay_with_jitter as u64)
    }
}
```

## State Transition Table (Quick Reference)

| From | To | Condition | Side Effects |
|------|-----|-----------|--------------|
| NEW | QUEUED | Validation success | Create run, audit log |
| NEW | CANCELED | User cancel | Audit log |
| QUEUED | LEASED | Scheduler claims | Set lease, agent_id |
| QUEUED | SCHEDULED | Has due_ts | Set due_ts |
| QUEUED | CANCELED | User cancel | Audit log |
| SCHEDULED | QUEUED | Time gate passed | Clear due_ts |
| SCHEDULED | CANCELED | User cancel | Audit log |
| LEASED | RUNNING | Agent starts | Set started_at |
| LEASED | EXPIRED | Lease timeout | Clear lease, agent_id |
| LEASED | QUEUED | Reclaimed | Clear lease, agent_id |
| LEASED | CANCELED | User cancel | Audit log |
| RUNNING | DONE | Success | Set completed_at |
| RUNNING | FAILED | Error | Set error, increment attempts |
| RUNNING | CANCELED | User cancel | Audit log |
| RUNNING | EXPIRED | Lease timeout | Clear lease, agent_id |
| FAILED | LEASED | Retry | Clear error, schedule retry |
| FAILED | DLQ | Exhausted retries | Audit log |
| FAILED | QUEUED | Manual requeue | Clear error, attempts |
| FAILED | CANCELED | User cancel | Audit log |
| EXPIRED | QUEUED | Reclaimed | Clear lease, agent_id |
| EXPIRED | LEASED | Re-leased | Set lease, agent_id |
| EXPIRED | FAILED | Failed during execution | Set error, increment attempts |
| EXPIRED | CANCELED | User cancel | Audit log |
| DONE | - | Terminal | - |
| DLQ | - | Terminal | - |
| CANCELED | - | Terminal | - |
