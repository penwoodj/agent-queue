# MVP Implementation Plan Suite

## Overview

This plan suite implements the Agent Queue System MVP with **transpiler integration** that delegates workflow execution to `yaml-to-rust-agentsdk`. Agent-queue is a queue orchestrator — it handles scheduling, state management, retry, DLQ, and audit. The transpiler handles all workflow execution (LLM calls, tools, steps).

## Assumptions

1. **Transpiler is Complete**: The workflow execution engine in `yaml-to-rust-agentsdk` is fully functional and can execute workflows independently
2. **Transpiler Delegation**: Agent-queue invokes the transpiler as an external CLI subprocess and processes its JSON output — it does NOT reimplement any execution logic
3. **Metadata-Only Tracking**: Agent-queue tracks orchestration metadata (run state, lease, retry count, artifact paths). Step-level execution details belong to the transpiler (per IN-11)
4. **Queue-Level Retry Only**: Per IN-09, transpiler internal retry is disabled. Agent-queue owns all retry decisions
5. **Extensive Logging**: Every state transition and queue operation will be logged with correlation IDs, compatible with transpiler's 9-level hierarchy (per IN-14)
6. **Verification-First**: Tests will validate all 48 MVP requirements with structured assertions

## Plan Structure

```
plans/mvp-implementation/
├── 00-overview.md              # This file
├── 01-architecture.md          # Architecture with transpiler integration
├── 02-data-model.md            # Data model (metadata-only steps/artifacts)
├── 03-state-machine.md         # State machine implementation
├── phases/
│   ├── 01-foundation.md        # Phase 1: Entities, state, persistence (32h)
│   ├── 02-queue-engine.md     # Phase 2: Scheduling, leases, retries (36h)
│   ├── 03-transpiler-integration.md   # Phase 3: Transpiler integration (27h)
│   ├── 04-cli-integration.md  # Phase 4: CLI + integration (46h)
│   └── 05-testing-polish.md   # Phase 5: Testing + verification (36h)
└── verification/
    ├── requirements-traceability.md  # 48 requirements → tests mapping
    └── success-criteria.md          # 15 success criteria checklist
```

## Implementation Phases

### Phase 1: Foundation (Week 1 — 32h)
- Project scaffold (Cargo.toml, dependencies)
- Define all entity types (Run, Workflow, Step, Artifact, AuditLog)
- Implement 10-state run lifecycle state machine
- Create SQLite persistence layer with WAL mode
- YAML workflow parsing (transpiler-format, per IN-08)
- Error handling and logging setup

### Phase 2: Queue Engine (Week 2 — 36h)
- Queue categories (ASAP, Whenever, Scheduled, Cron)
- Scheduler with meta-scheduling (EDF, WRR, priority-based)
- Lease management and heartbeat keep-alive
- Retry logic with exponential backoff + jitter
- DLQ routing and triage
- Backpressure and admission control
- Audit logging (append-only)

### Phase 3: Transpiler Integration (Week 3 — 27h)
- Transpiler CLI invocation interface
- Heartbeat management during long executions
- Result parsing and error classification (per IN-13)
- Artifact metadata collection (per IN-12)
- Environment variable resolution and passthrough
- Validation mode integration (per IN-15)
- Mock transpiler binary for testing

### Phase 4: CLI + Integration (Week 4-5 — 46h)
- CLI commands (enqueue, list, inspect, cancel, retry, schedule, drain, validate, cleanup, daemon)
- Queue-to-transpiler wiring
- Error propagation (transpiler → queue → user)
- Cron scheduling with timezone support
- Daemon mode with graceful shutdown

### Phase 5: Testing + Verification (Week 5-6 — 36h)
- 156 tests (48 unit + 48 integration + 48 E2E + 12 error scenarios)
- Manual verification against 15 success criteria
- Performance tuning and benchmarking
- Documentation and code polish

## Verification Strategy

Each MVP requirement will be verified through:

1. **Unit Tests** (48): Test individual components in isolation
2. **Integration Tests** (48): Test component interactions with mock transpiler
3. **End-to-End Tests** (48): Test complete user journeys via CLI
4. **Error Scenario Tests** (12): Test failure modes and edge cases
5. **Manual Verification**: CLI command execution against success criteria

## Success Criteria

The MVP is complete when:
- All 48 MVP requirements are implemented and verified
- All 10 must-have success criteria pass
- Test coverage > 80%
- All end-to-end workflows execute successfully
- System survives process restart
- Logs trace all state transitions

## Next Steps

Start with Phase 1: Foundation — read `phases/01-foundation.md`.
