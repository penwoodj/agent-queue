# Phase 5: Testing + Polish (Week 5-6)

**36 hours (1 week)**

## Overview

This phase focuses on comprehensive testing, verification against all 48 MVP requirements, performance tuning, bug fixes, and documentation. The goal is to prove the system works end-to-end and is ready for use.

## Goals

1. **100% requirement coverage**: Every MVP requirement has passing tests
2. **E2E verification**: Complete user journeys tested
3. **Error scenarios**: Failure modes tested and handled
4. **Performance validation**: System meets performance targets
5. **Documentation**: All components documented

## Prerequisites

- ✅ Phase 1 complete: Entities, state machine, persistence
- ✅ Phase 2 complete: Queue engine, scheduling, retry logic
- ✅ Phase 3 complete: Transpiler integration
- ✅ Phase 4 complete: CLI + integration wiring

---

## Task Breakdown

### Task 5.1: Unit Tests — State Machine & Entities (4 hours)

**Deliverable**: Complete unit test coverage for core types

**Acceptance Criteria**:
- [ ] All RunState transitions tested (valid + invalid)
- [ ] Terminal state behavior tested
- [ ] QueueCategory and Priority serialization tested
- [ ] Entity creation and validation tested
- [ ] Error type classification tested
- [ ] All 48 unit tests passing

**Test File**: `src/state/tests/`

```rust
#[cfg(test)]
mod state_machine_tests {
    // Test every valid and invalid transition
    // Test terminal state immutability
    // Test transition side effects (audit log)
}

#[cfg(test)]
mod entity_tests {
    // Test entity creation
    // Test serialization/deserialization
    // Test validation rules
}

#[cfg(test)]
mod error_tests {
    // Test error classification (retryable vs non-retryable)
    // Test error message sanitization (no secrets in logs)
    // Test transpiler error mapping (per IN-13)
}
```

---

### Task 5.2: Integration Tests — Queue Engine (6 hours)

**Deliverable**: Integration tests for queue engine with mock transpiler

**Acceptance Criteria**:
- [ ] Enqueue → schedule → lease → execute → done flow
- [ ] Retry logic with exponential backoff
- [ ] DLQ routing after exhausted retries
- [ ] Lease expiry and reclamation
- [ ] Backpressure admission control
- [ ] Queue priority ordering (critical > high > normal > low)
- [ ] Cron scheduling
- [ ] Idempotency enforcement
- [ ] Audit log entries for all state transitions
- [ ] All 48 integration tests passing

**Test File**: `tests/integration/`

```rust
#[tokio::test]
async fn test_enqueue_execute_done() {
    // 1. Create in-memory SQLite
    // 2. Enqueue workflow with mock transpiler
    // 3. Start queue engine
    // 4. Verify run transitions: NEW → QUEUED → LEASED → RUNNING → DONE
    // 5. Verify artifacts metadata stored
    // 6. Verify audit log entries
}

#[tokio::test]
async fn test_retry_then_dlq() {
    // 1. Configure mock transpiler to always fail
    // 2. Enqueue with max_attempts=3
    // 3. Verify 3 retries with backoff
    // 4. Verify final state is DLQ
}

#[tokio::test]
async fn test_lease_expiry_reclaim() {
    // 1. Lease a task
    // 2. Don't send heartbeat
    // 3. Wait for lease expiry
    // 4. Verify task reclaimed to QUEUED
}

#[tokio::test]
async fn test_backpressure_rejects() {
    // 1. Set max_concurrent=2
    // 2. Enqueue 5 tasks
    // 3. Verify only 2 running
    // 4. Verify others remain QUEUED
}
```

---

### Task 5.3: E2E Tests — CLI User Journeys (4 hours)

**Deliverable**: End-to-end tests via CLI commands

**Acceptance Criteria**:
- [ ] Full workflow: enqueue → list → inspect → done
- [ ] Failure workflow: enqueue → fail → retry → done
- [ ] DLQ workflow: enqueue → fail repeatedly → inspect DLQ → requeue
- [ ] Cancellation: enqueue → cancel → verify state
- [ ] Scheduled: enqueue with cron → verify scheduled state
- [ ] Validation: validate valid and invalid YAML
- [ ] Drain: enqueue multiple → drain → verify empty
- [ ] All 48 E2E tests passing

**Test File**: `tests/e2e/`

```rust
#[tokio::test]
async fn test_e2e_happy_path() {
    // 1. agent-queue validate workflow.yaml → success
    // 2. agent-queue enqueue --workflow workflow.yaml → prints run_id
    // 3. agent-queue list → shows run in queued/running
    // 4. Wait for completion
    // 5. agent-queue inspect <run_id> → shows done state
    // 6. Verify artifacts exist
}

#[tokio::test]
async fn test_e2e_failure_recovery() {
    // 1. Enqueue workflow with invalid config
    // 2. agent-queue list --state failed → shows run
    // 3. agent-queue retry <run_id> → resets to queued
    // 4. agent-queue inspect <run_id> → shows retry state
}
```

---

### Task 5.4: Error Scenario Tests (4 hours)

**Deliverable**: Dedicated tests for failure modes

**Acceptance Criteria**:
- [ ] Transpiler process crash → run marked FAILED
- [ ] Transpiler timeout → run marked FAILED (retryable)
- [ ] Transpiler invalid output → run marked FAILED (non-retryable)
- [ ] Database connection lost → graceful error, no data corruption
- [ ] Invalid YAML → clear error message, no enqueue
- [ ] Missing env vars → clear error before execution
- [ ] Duplicate idempotency key → idempotent response
- [ ] Concurrent enqueue race → exactly one succeeds
- [ ] Process kill during execution → lease expires, run reclaimable

**Test File**: `tests/error_scenarios/`

---

### Task 5.5: Manual Verification (4 hours)

**Deliverable**: Manual test execution against success criteria

**Acceptance Criteria**:
- [ ] Execute all 10 must-have success criteria from `verification/success-criteria.md`
- [ ] Execute all 5 should-have criteria
- [ ] Document results with screenshots/output
- [ ] File any issues found

**Verification Checklist** (from success-criteria.md):

1. [ ] Enqueue YAML workflow and it executes to completion
2. [ ] ASAP tasks run before Whenever tasks
3. [ ] Failed tasks retry automatically and reach DLQ
4. [ ] DLQ tasks can be inspected and requeued
5. [ ] Cron workflow runs on schedule
6. [ ] System survives process restart (data persists)
7. [ ] Structured logs trace every state transition
8. [ ] Artifact output is persisted and accessible
9. [ ] CLI validate catches invalid YAML
10. [ ] Cancel stops a queued or running task

---

### Task 5.6: Documentation (4 hours)

**Deliverable**: Complete project documentation

**Acceptance Criteria**:
- [ ] README.md with quick start, architecture overview, CLI reference
- [ ] CONTRIBUTING.md with development setup
- [ ] Inline code documentation (/// doc comments on public API)
- [ ] Example workflow YAML files
- [ ] Configuration reference

---

### Task 5.7: Performance Tuning (4 hours)

**Deliverable**: System meets performance targets

**Acceptance Criteria**:
- [ ] Enqueue latency < 100ms
- [ ] Schedule latency < 1s
- [ ] State transition time < 50ms
- [ ] Heartbeat overhead < 10ms
- [ ] No memory leaks in daemon mode (run for 1h, check)
- [ ] SQLite WAL performance acceptable (batch inserts)

**Benchmarking approach**:
```bash
# Enqueue benchmark
time agent-queue enqueue --workflow test.yaml

# Throughput test
for i in $(seq 1 100); do agent-queue enqueue --workflow test.yaml --idempotency_key $i; done
time agent-queue list --limit 100

# Daemon stability test
agent-queue daemon --workers 1 --max-concurrent 3 &
# Run 50 workflows, monitor memory with /usr/bin/time -v
```

---

### Task 5.8: Bug Fixes and Polish (6 hours)

**Deliverable**: All known issues resolved

**Acceptance Criteria**:
- [ ] All bugs found during testing fixed
- [ ] Error messages user-friendly (no stack traces in CLI output)
- [ ] Log output clean and parsable
- [ ] CLI help text accurate
- [ ] Edge cases handled (empty queue, missing workflow, etc.)
- [ ] Code passes `cargo clippy` with no warnings
- [ ] Code formatted with `cargo fmt`

---

## Testing Strategy

### Test Types Summary

| Type | Location | Count | Runtime |
|------|----------|-------|---------|
| Unit | `src/*/tests/` | 48 | < 30s total |
| Integration | `tests/integration/` | 48 | < 5 min total |
| E2E | `tests/e2e/` | 48 | < 10 min total |
| Error Scenarios | `tests/error_scenarios/` | 12 | < 3 min total |
| **Total** | | **156** | |

### Coverage Targets

| Metric | Target | Tool |
|--------|--------|------|
| Line coverage | > 80% | `cargo tarpaulin` |
| Requirement coverage | 100% (48/48) | Traceability matrix |
| Success criteria | 10/10 must-have | Manual verification |

---

## Success Criteria

**Phase complete when:**

- [ ] All 156+ tests passing
- [ ] All 48 MVP requirements verified
- [ ] 10/10 must-have criteria pass
- [ ] Performance targets met
- [ ] No clippy warnings
- [ ] Documentation complete
- [ ] System runs stable for 1+ hour daemon test

---

## Dependencies

This phase depends on:
- ✅ Phase 1: Foundation (32h)
- ✅ Phase 2: Queue Engine (36h)
- ✅ Phase 3: Transpiler Integration (27h)
- ✅ Phase 4: CLI + Integration (46h)

---

## Effort Summary

| Task | Hours | Cumulative |
|------|--------|------------|
| Unit Tests | 4h | 4h |
| Integration Tests | 6h | 10h |
| E2E Tests | 4h | 14h |
| Error Scenario Tests | 4h | 18h |
| Manual Verification | 4h | 22h |
| Documentation | 4h | 26h |
| Performance Tuning | 4h | 30h |
| Bug Fixes & Polish | 6h | 36h |
| **Phase 5 Total** | **36h** | **177h** (Phase 1+2+3+4+5) |

---

**Previous**: [Phase 4: CLI + Integration](../phases/04-cli-integration.md)
