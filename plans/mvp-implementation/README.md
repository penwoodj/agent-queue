# MVP Implementation Plan Suite - Summary

## Overview

This plan suite implements the **Agent Queue System MVP** with **transpiler integration** that delegates workflow execution to `yaml-to-rust-agentsdk`. This approach focuses on queue orchestration requirements without reimplementing the workflow execution engine.

## What You Get

### Complete Implementation Plan

✅ **Architecture Design** - Full component design with transpiler integration
✅ **Data Model** - Complete SQLite schema and entity definitions
✅ **State Machine** - All state transitions with validation
✅ **5 Implementation Phases** - Detailed task breakdown with effort estimates
✅ **Transpiler Integration** - CLI/API interface to yaml-to-rust-agentsdk
✅ **Comprehensive Testing Strategy** - 156 tests across 3 test types + error scenarios (48 queue requirements)
✅ **Verification Matrix** - All 48 MVP requirements mapped to tests
✅ **Success Criteria** - 10 must-have + 5 should-have criteria

## Directory Structure

```
plans/mvp-implementation/
├── 00-overview.md              # This summary
├── 01-architecture.md          # Architecture with transpiler integration
├── 02-data-model.md            # Complete SQLite schema (metadata-only steps/artifacts)
├── 03-state-machine.md         # State machine implementation
├── phases/
│   ├── 01-foundation.md        # Week 1: Entities, state, persistence (32h)
│   ├── 02-queue-engine.md     # Week 2: Scheduling, leases, retries (36h)
│   ├── 03-transpiler-integration.md   # Week 3: Transpiler integration (27h)
│   ├── 04-cli-integration.md  # Week 4-5: CLI + integration (46h)
│   └── 05-testing-polish.md   # Week 5-6: Testing + verification (36h)
├── verification/
│   ├── requirements-traceability.md  # 48 requirements → tests mapping
│   └── success-criteria.md          # 15 success criteria checklist
└── tests/                                    # Test specifications
```

## Implementation Timeline

### Phase 1: Foundation (Week 1 - 32 hours)
**Deliverable**: Working data model and state machine

Tasks:
1. Entity definitions (4h)
2. State machine implementation (6h)
3. SQLite persistence (8h)
4. YAML schema (4h)
5. YAML parser + validation (6h)
6. Error types (2h)
7. Logging setup (2h)

**Outcome**: Can enqueue workflows to queue with full state tracking

---

### Phase 2: Queue Engine (Week 2 - 36 hours)
**Deliverable**: Functional queue scheduling system

Tasks:
1. Queue categories (4h)
2. Scheduler with meta-scheduling (8h)
3. Lease management (6h)
4. Retry logic + DLQ (4h)
5. Queue routing (3h)
6. Backpressure (3h)
7. Audit logging (2h)
8. Engine core (4h)

**Outcome**: Tasks scheduled, executed, and recovered from failures

---

### Phase 3: Transpiler Integration (Week 3 - 27 hours)
**Deliverable**: Transpiler integration for workflow execution

Tasks:
1. Transpiler CLI integration interface (4h)
2. CLI invocation workflow (6h)
3. Heartbeat management during transpiler execution (4h)
4. Result parsing and error handling (4h)
5. Artifact collection from transpiler output (4h)
6. Environment variable passthrough (2h)
7. Validation mode integration (3h)

**Outcome**: Queue orchestrates transpiler for workflow execution

---

### Phase 4: CLI + Integration (Week 4-5 - 46 hours)
**Deliverable**: User interface and full integration

Tasks:
1. CLI framework (3h)
2. enqueue command (4h)
3. list command (4h)
4. inspect command (4h)
5. cancel command (2h)
6. retry command (3h)
7. schedule command (3h)
8. drain command (2h)
9. validate command (2h)
10. Queue to transpiler integration (6h)
11. Transpiler to state transitions (4h)
12. Error propagation (3h)
13. Cron scheduler (6h)

**Outcome**: Complete CLI with all commands working

---

### Phase 5: Testing + Polish (Week 5-6 - 36 hours)
**Deliverable**: Verified, tested, documented system

Tasks:
1. Unit tests (4h) - 48 tests
2. Integration tests (6h) - 48 tests
3. E2E tests (4h) - 48 tests
4. Error scenario tests (4h)
5. Manual verification (4h)
6. Documentation (4h)
7. Performance tuning (4h)
8. Bug fixes (6h)

**Outcome**: All requirements verified with >80% test coverage

---

## Key Design Decisions

### 1. Transpiler Delegation

**Why**: Workflow execution is already fully implemented in `yaml-to-rust-agentsdk`. Reimplementing it would be wasted effort. agent-queue focuses on what the transpiler doesn't do: queue orchestration, scheduling, state management, retry, DLQ, audit.

**How**: Transpiler invoked as CLI subprocess. agent-queue manages:
- Queue lifecycle (enqueue → schedule → lease → run → done/failed)
- Retry with exponential backoff (queue-level, per IN-09)
- Heartbeat keep-alive during long transpiler executions
- Error classification and DLQ routing
- Artifact metadata tracking

**Result**: agent-queue is a queue orchestrator, not an agent runtime.

---

### 2. Single Process Architecture

**Why**: Simplicity for MVP

**Components in one Tokio runtime**:
- Queue engine
- Transpiler integration (subprocess invocation)
- SQLite store
- CLI commands

**Post-MVP**: Can split into multiple processes if needed

---

### 3. Extensive Logging

**Every operation logs**:
- trace_id: Correlation ID for entire workflow
- run_id: Run identifier
- task_id: Internal task identifier
- step_id: Step identifier
- action: What happened
- state: Before/after state
- duration_ms: How long it took
- error: Error details (if applicable)

**Format**: Structured JSON for easy parsing

---

### 4. Verification-First Approach

**Every requirement has corresponding tests**:
- Unit test: Component isolation
- Integration test: Component interaction
- E2E test: Complete workflow

**Coverage**: 100% of MVP requirements

---

## Test Strategy

### Test Types

#### Unit Tests (48 tests)
Test individual components in isolation
- Fast execution (< 1s per test)
- Mock transpiler interface (not mock LLM)
- Location: `src/*/tests/`

#### Integration Tests (48 tests)
Test component interactions
- Use in-memory SQLite
- Mock transpiler for execution simulation
- Location: `tests/integration/`

#### End-to-End Tests (48 tests)
Test complete user journeys
- Use real CLI commands
- Verify all state transitions
- Location: `tests/e2e/`

**Total**: 144 tests + 12 error scenario tests = 156 tests

---

### Test Coverage Matrix

| Area | Unit | Integration | E2E | Total | Coverage |
|------|-------|-------------|------|-------|----------|
| Entities & State | 3 | 3 | 3 | 9 | 100% |
| Queue Engine | 20 | 20 | 20 | 60 | 100% |
| Transpiler Integration | 7 | 7 | 7 | 21 | 100% |
| CLI & Integration | 18 | 18 | 18 | 54 | 100% |
| **Total** | **48** | **48** | **48** | **144** | **100%** |

---

## Verification Criteria

### Must-Have (10 criteria)

System is **useful ONLY** if all pass:

1. ✅ Enqueue YAML workflow and it executes to completion
2. ✅ ASAP tasks run before Whenever tasks
3. ✅ Failed tasks retry automatically and reach DLQ
4. ✅ DLQ tasks can be inspected and requeued
5. ✅ Cron workflow runs on schedule
6. ✅ System survives process restart
7. ✅ Structured logs trace every state transition
8. ✅ Artifact output is persisted and accessible
9. ✅ CLI validate catches invalid YAML
10. ✅ Cancel stops a queued or running task

### Should-Have (5 criteria)

System is **significantly better** if these pass:

11. 🔄 Lease expiry reclaims stuck tasks
12. 🔄 Backpressure rejects when queue is full
13. 🔄 Idempotency key prevents duplicate enqueue
14. 🔄 Audit log records all state changes
15. 🔄 Step-level errors show which step failed

---

## Success Metrics

### Code Coverage
- Target: > 80%
- Tool: `cargo tarpaulin`

### Requirement Coverage
- Target: 100% of 48 MVP requirements
- Current: Planned 100%

### Test Pass Rate
- Target: 100% of 156 tests pass

### Performance Targets
- Enqueue latency: < 100ms
- Schedule latency: < 1s
- State transition: < 50ms

---

## Getting Started

### Prerequisites

```bash
# Rust toolchain
rustup install stable
cargo install cargo-tarpaulin

# Dependencies
cargo build
```

### Implementation Order

1. **Start with Phase 1**: Foundation
   - Read `phases/01-foundation.md`
   - Implement entity types
   - Implement state machine
   - Create SQLite schema
   - Write unit tests

2. **Continue to Phase 2**: Queue Engine
   - Read `phases/02-queue-engine.md`
   - Implement queue categories
   - Implement scheduler
   - Implement lease management
   - Write integration tests

3. **Phase 3**: Transpiler Integration
   - Read `phases/03-transpiler-integration.md`
   - Implement CLI invocation interface
   - Implement heartbeat management
   - Implement result parsing and error classification
   - Test with queue engine

4. **Phase 4**: CLI + Integration
   - Read `phases/04-cli-integration.md`
   - Implement all CLI commands
   - Integrate queue and agent
   - Test end-to-end

5. **Phase 5**: Testing + Verification
   - Read `phases/05-testing-polish.md`
   - Run all test suites
   - Verify success criteria
   - Generate coverage report
   - Document results

---

## Verification Process

### Step 1: Run All Tests

```bash
# Unit tests
cargo test --lib

# Integration tests
cargo test --test '*'

# E2E tests
cargo test --test e2e -- --test-threads=1
```

### Step 2: Generate Coverage

```bash
cargo tarpaulin --out Html
open tarpaulin-report.html
```

### Step 3: Manual Verification

```bash
# Follow success-criteria.md
# Execute each criterion manually
# Document results
```

### Step 4: Generate Report

```bash
# Use verification/success-criteria.md template
# Fill in test results
# Mark completion
```

---

## What Happens Next

### MVP Complete ✅

When all criteria pass:
1. ✅ Queue engine fully functional
2. ✅ Stubbed execution engine verified
3. ✅ All requirements tested
4. ✅ Documentation complete
5. ✅ Ready for deployment

**Post-MVP Evolution** (from MVP definition):
- Phase 6: Robustness (+2 weeks)
  - DAG dependencies
  - Checkpointing
  - Priority aging
- Phase 7: Multi-User (+4 weeks)
  - Multi-tenancy
  - RBAC
  - Rate limiting
- Phase 8: Scale (+6 weeks)
  - Agent pools
  - Work stealing
  - Batching

### MVP Incomplete ❌

If criteria fail:
1. Review failed tests
2. Fix issues
3. Re-run verification
4. Iterate until complete

---

## Key Documents

- **`00-overview.md`**: This summary
- **`01-architecture.md`**: Full architecture design
- **`02-data-model.md`**: Complete schema
- **`03-state-machine.md`**: State transitions
- **`phases/*.md`**: Detailed implementation plans
- **`verification/requirements-traceability.md`**: Requirements → tests mapping
- **`verification/success-criteria.md`**: Success checklist

---

## Questions?

Refer to individual phase documents for detailed implementation guidance.

Each phase includes:
- ✅ Task breakdown
- ✅ Effort estimates
- ✅ Code examples
- ✅ Acceptance criteria
- ✅ Test specifications

---

## Summary

This plan suite provides:

✅ **Complete architecture** with transpiler integration
✅ **5 implementation phases** with detailed tasks (178 hours)
✅ **156 tests** across 3 test types + error scenarios (100% requirement coverage for 48 queue requirements)
✅ **15 success criteria** for verification
✅ **8 integration requirements** (IN-08 through IN-15) for robust transpiler coupling
✅ **Comprehensive documentation** for each component

**Result**: A production-ready queue orchestration MVP that delegates workflow execution to yaml-to-rust-agentsdk.
