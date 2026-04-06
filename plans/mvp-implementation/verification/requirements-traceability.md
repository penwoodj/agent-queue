# Requirements Traceability Matrix

This document maps every MVP requirement to its corresponding tests and verification criteria.

## Requirement to Test Mapping

### Queue Engine Requirements (30)

| ID | Requirement | Unit Test | Integration Test | E2E Test | Verification Criteria |
|-----|-------------|------------|------------------|-------------|----------------------|
| QE-01 | Entity definitions | `test_entity_types` | `test_workflow_lifecycle` | `test_full_workflow` | All entities created and persisted |
| QE-02 | State machine | `test_all_transitions` | `test_state_transitions` | `test_workflow_states` | All transitions valid |
| QE-03 | SQLite persistence | `test_crud_operations` | `test_transaction_rollbacks` | `test_restart_recovery` | Data survives restart |
| QE-04 | YAML validation | `test_valid_yaml` | `test_invalid_yaml` | `test_validation_cli` | Clear errors on invalid YAML |
| QE-05 | 3 queue categories | `test_queue_categories` | `test_category_routing` | `test_multi_category` | All categories work |
| QE-06 | Priority bands | `test_priority_ordering` | `test_priority_preemption` | `test_priority_execution` | Higher priority runs first |
| QE-07 | FIFO tie-breaking | `test_fifo_ordering` | `test_same_priority` | `test_fifo_execution` | Older tasks first |
| QE-08 | Lease + heartbeat | `test_lease_claim` | `test_heartbeat_expiry` | `test_lease_recovery` | Lease expires without heartbeat |
| QE-09 | Retry with backoff | `test_retry_delays` | `test_retry_logic` | `test_failure_retry` | Delays increase exponentially |
| QE-10 | Dead-letter queue | `test_dlq_movement` | `test_dlq_retrieval` | `test_exhausted_retries` | Exhausted retries go to DLQ |
| QE-11 | Idempotency key | `test_idempotency` | `test_duplicate_enqueue` | `test_idempotency_cli` | Duplicate keys rejected |
| QE-12 | Cancellation | `test_cancel_queued` | `test_cancel_running` | `test_cancel_cli` | Cancelled tasks don't run |
| QE-13 | Concurrency limits | `test_max_in_flight` | `test_backpressure` | `test_concurrency_limit` | Respects max_in_flight |
| QE-14 | Structured logging | `test_log_format` | `test_correlation_ids` | `test_log_traces` | All events logged with trace_id |
| QE-15 | CLI: enqueue | `test_enqueue_command` | `test_enqueue_validation` | `test_enqueue_workflow` | Workflows enqueued successfully |
| QE-16 | CLI: list | `test_list_command` | `test_list_filters` | `test_list_output` | Lists tasks with filters |
| QE-17 | CLI: inspect | `test_inspect_command` | `test_inspect_details` | `test_inspect_steps` | Shows task details |
| QE-18 | CLI: cancel | `test_cancel_command` | `test_cancel_state` | `test_cancel_cli` | Cancels running tasks |
| QE-19 | CLI: retry | `test_retry_command` | `test_retry_from_dlq` | `test_retry_cli` | Retries failed tasks |
| QE-20 | CLI: drain | `test_drain_command` | `test_drain_wait` | `test_drain_cli` | Waits for completion |
| QE-21 | Meta-scheduler | `test_scheduler_selection` | `test_meta_scheduling` | `test_scheduler_priority` | ASAP prioritized |
| QE-22 | Queue routing | `test_routing_logic` | `test_auto_routing` | `test_yaml_routing` | Routes based on YAML |
| QE-23 | Repeat-Cron | `test_cron_parsing` | `test_cron_execution` | `test_cron_workflow` | Cron schedules execute |
| QE-24 | Artifact storage | `test_artifact_create` | `test_artifact_retrieval` | `test_artifact_files` | Artifacts persisted to disk |
| QE-25 | Audit log | `test_audit_entries` | `test_audit_trail` | `test_audit_query` | All changes audited |
| QE-26 | Backpressure | `test_admission_control` | `test_rejection` | `test_backpressure_cli` | Rejects when full |
| QE-27 | Due-time gate | `test_due_time_gate` | `test_scheduled_promotion` | `test_scheduled_workflow` | Tasks wait until due |
| QE-28 | Admission control | `test_admission` | `test_rejection_reason` | `test_admission_cli` | Clear error on rejection |
| QE-29 | Step error classification | `test_retryable_errors` | `test_non_retryable` | `test_error_classification` | Errors classified correctly |
| QE-30 | Schema version header | `test_version_check` | `test_version_migration` | `test_version_cli` | Version validated |

### Transpiler Integration Requirements (3)

| ID | Requirement | Unit Test | Integration Test | E2E Test | Verification Criteria |
|-----|-------------|------------|------------------|-------------|----------------------|
| IN-EX-01 | LLM provider abstraction | `test_provider_trait` | `test_transpiler_cli` | `test_transpiler_integration` | Abstraction allows future providers |
| IN-EX-02 | Environment variable passthrough | `test_env_resolution` | `test_secret_handling` | `test_env_workflow` | Env vars resolved and passed |
| IN-EX-03 | Deterministic validation mode | `test_validation_cli` | `test_validate_workflow` | `test_dry_run` | Validates without execution |

### Transpiler Integration Requirements (7)

| ID | Requirement | Unit Test | Integration Test | E2E Test | Verification Criteria |
|-----|-------------|------------|------------------|-------------|----------------------|
| IN-01 | Queue dispatches to transpiler | `test_dispatch_logic` | `test_transpiler_dispatch` | `test_dispatch_workflow` | Queue triggers transpiler |
| IN-02 | Transpiler reports back | `test_reporting_logic` | `test_state_update` | `test_completion_workflow` | Transpiler updates state |
| IN-03 | Lease extension via heartbeat | `test_heartbeat_extend` | `test_heartbeat_during_run` | `test_heartbeat_workflow` | Heartbeat extends lease |
| IN-04 | Workflow input to transpiler | `test_input_passing` | `test_workflow_params` | `test_param_workflow` | Queue passes input |
| IN-05 | Artifact handoff | `test_artifact_transfer` | `test_transpiler_to_queue` | `test_artifact_workflow` | Artifacts flow to queue |
| IN-06 | Error propagation | `test_error_passing` | `test_retry_propagation` | `test_error_workflow` | Errors trigger retry |
| IN-07 | Single process architecture | `test_process_integration` | `test_shared_state` | `test_single_process` | All components in one process |

## Test Coverage Matrix

### Coverage by Feature Area

| Area | Total Requirements | Unit Tests | Integration Tests | E2E Tests | Coverage |
|------|-------------------|-------------|------------------|-------------|----------|
| Entities & State | 3 | 3 | 3 | 3 | 100% |
| Persistence | 1 | 1 | 1 | 1 | 100% |
| Validation | 1 | 1 | 1 | 1 | 100% |
| Queue Categories | 2 | 2 | 2 | 2 | 100% |
| Priority | 2 | 2 | 2 | 2 | 100% |
| Leases | 1 | 1 | 1 | 1 | 100% |
| Retry & DLQ | 4 | 4 | 4 | 4 | 100% |
| Cancellation | 1 | 1 | 1 | 1 | 100% |
| Concurrency | 1 | 1 | 1 | 1 | 100% |
| Observability | 2 | 2 | 2 | 2 | 100% |
| CLI | 6 | 6 | 6 | 6 | 100% |
| Scheduling | 3 | 3 | 3 | 3 | 100% |
| Artifacts | 1 | 1 | 1 | 1 | 100% |
| Backpressure | 1 | 1 | 1 | 1 | 100% |
| Admission | 1 | 1 | 1 | 1 | 100% |
| Transpiler Integration | 3 | 3 | 3 | 3 | 100% |
| Integration | 7 | 7 | 7 | 7 | 100% |
| **Total** | **40** | **40** | **40** | **40** | **100%** |

## Verification Checklist

### Must-Have Criteria (10)

- [ ] 1. Enqueue a YAML workflow and it executes to completion
  - Test: `test_full_workflow`
  - Command: `agent-queue enqueue test.yaml && agent-queue list --state done`

- [ ] 2. ASAP tasks run before Whenever tasks
  - Test: `test_priority_preemption`
  - Enqueue both, verify ASAP completes first

- [ ] 3. Failed tasks retry automatically and reach DLQ
  - Test: `test_failure_retry`
  - Force a failure, observe 3 retries → DLQ

- [ ] 4. DLQ tasks can be inspected and requeued
  - Test: `test_retry_from_dlq`
  - Command: `agent-queue retry <id>` succeeds

- [ ] 5. Cron workflow runs on schedule
  - Test: `test_cron_workflow`
  - Schedule a 1-minute cron, verify execution

- [ ] 6. System survives process restart
  - Test: `test_restart_recovery`
  - Kill and restart, queued tasks resume

- [ ] 7. Structured logs trace every state transition
  - Test: `test_log_traces`
  - Grep logs for task_id, see full lifecycle

- [ ] 8. Artifact output is persisted and accessible
  - Test: `test_artifact_files`
  - Check `./artifacts/<run_id>/`

- [ ] 9. CLI validate catches invalid YAML
  - Test: `test_validation_cli`
  - Command: `agent-queue validate bad.yaml` returns non-zero

- [ ] 10. Cancel stops a queued task
  - Test: `test_cancel_cli`
  - Command: `agent-queue cancel <id>` sets state to canceled

### Should-Have Criteria (5)

- [ ] 11. Lease expiry reclaims stuck tasks
  - Test: `test_lease_recovery`
  - Block heartbeat, observe reclaim

- [ ] 12. Backpressure rejects when queue is full
  - Test: `test_backpressure_cli`
  - Fill queue, verify rejection with clear error

- [ ] 13. Idempotency key prevents duplicate enqueue
  - Test: `test_idempotency_cli`
  - Enqueue same key twice, second is rejected

- [ ] 14. Audit log records all state changes
  - Test: `test_audit_query`
  - Query audit_log table

- [ ] 15. Step-level errors show which step failed
  - Test: `test_inspect_steps`
  - Command: `agent-queue inspect <id> --steps`

## Test Execution Strategy

### Phase 1: Unit Tests (Week 5, Days 1-2)

Run all unit tests:
```bash
cargo test --lib
```

Expected: All 55 unit tests pass

### Phase 2: Integration Tests (Week 5, Days 3-4)

Run all integration tests:
```bash
cargo test --test '*'
```

Expected: All 55 integration tests pass

### Phase 3: End-to-End Tests (Week 5, Days 5)

Run all E2E tests with real CLI:
```bash
cargo test --test e2e -- --test-threads=1
```

Expected: All 55 E2E tests pass

### Phase 4: Manual Verification (Week 6)

Manually verify all 15 success criteria:
1. Create test workflows
2. Execute CLI commands
3. Verify results
4. Check logs
5. Inspect database

Expected: All 15 criteria pass

## Coverage Metrics

### Code Coverage

Run coverage report:
```bash
cargo tarpaulin --out Html
```

Target: > 80% coverage

### Requirement Coverage

Current: 55/55 = 100%

### Test Coverage

- Unit tests: 55
- Integration tests: 55
- E2E tests: 55
- Total: 165 tests

## Verification Report Template

```markdown
# MVP Verification Report

## Date: YYYY-MM-DD

## Test Results

### Unit Tests
- Total: 55
- Passed: XX
- Failed: XX
- Coverage: XX%

### Integration Tests
- Total: 55
- Passed: XX
- Failed: XX
- Coverage: XX%

### E2E Tests
- Total: 55
- Passed: XX
- Failed: XX
- Coverage: XX%

## Success Criteria

### Must-Have (10/10)
- [ ] 1. Enqueue and execute workflow
- [ ] 2. ASAP before Whenever
- [ ] 3. Retry and DLQ
- [ ] 4. DLQ inspection and retry
- [ ] 5. Cron scheduling
- [ ] 6. Process restart survival
- [ ] 7. Structured logging
- [ ] 8. Artifact persistence
- [ ] 9. YAML validation
- [ ] 10. Task cancellation

### Should-Have (5/5)
- [ ] 11. Lease expiry reclaim
- [ ] 12. Backpressure rejection
- [ ] 13. Idempotency enforcement
- [ ] 14. Audit log
- [ ] 15. Step error details

## Requirements Coverage

- Total MVP Requirements: 55
- Verified: XX
- Failed: XX
- Coverage: XX%

## Issues

List any failing tests or verification criteria with details.

## Conclusion

MVP is [COMPLETE | INCOMPLETE]
```
