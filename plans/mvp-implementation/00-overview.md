# MVP Implementation Plan Suite

## Overview

This plan suite implements the Agent Queue System MVP with a **transpiler integration** that uses delegates workflow execution to transpiler. This approach allows thorough verification of all queue engine requirements without needing the actual workflow execution engine.

## Assumptions

1. **Workflow Engine is Complete**: We assume the transpiler/ folder workflow engine is fully functional and can execute workflows
2. **Stubbed Execution**: The agent executor will be stubbed with sleep-based mocks that simulate:
   - LLM generation latency (5-30s per step)
   - Tool execution time (1-5s per tool)
   - Failures and retries (10% failure rate)
   - Context window exhaustion (rare, simulated)
3. **Extensive Logging**: Every state transition, tool call, and mock operation will be logged with correlation IDs
4. **Verification-First**: Tests will validate all MVP requirements with structured assertions

## Plan Structure

```
plans/mvp-implementation/
├── 00-overview.md              # This file
├── 01-architecture.md          # Architecture design with stubbed components
├── 02-data-model.md            # Complete data model with SQLite schema
├── 03-state-machine.md         # State machine implementation
├── phases/
│   ├── 01-foundation.md        # Phase 1: Entities, state, persistence
│   ├── 02-queue-engine.md     # Phase 2: Scheduling, leases, retries
│   ├── 03-transpiler-integration.md   # Phase 3: Transpler Integration
│   ├── 04-cli-integration.md  # Phase 4: CLI + integration
│   └── 05-testing-polish.md   # Phase 5: Testing + verification
├── tests/
│   ├── unit/                  # Unit test specifications
│   ├── integration/           # Integration test specifications
│   ├── e2e/                  # End-to-end test specifications
│   └── verification/          # Verification criteria per requirement
├── mocks/
│   ├── executor.rs            # Stubbed execution engine
│   ├── llm_provider.rs        # Mock LLM with sleep simulation
│   └── tools.rs              # Mock tools with latency
└── verification/
    ├── requirements-traceability.md  # Requirement to test mapping
    ├── success-criteria.md          # MVP success criteria checklist
    └── test-coverage.md            # Test coverage matrix
```

## Implementation Phases

### Phase 1: Foundation (Week 1)
- Define all entity types
- Implement state machine
- Create SQLite persistence layer
- YAML validation and parsing
- Error handling and logging setup

### Phase 2: Queue Engine (Week 2)
- Queue categories and priority
- Scheduler with meta-scheduling
- Lease management and heartbeat
- Retry logic with backoff
- DLQ and error handling
- Backpressure and admission control

### Phase 3: Agent SDK Mock (Week 3)
- Stubbed execution engine
- Mock LLM provider with sleep simulation
- Mock tools with latency
- Step runner with context management
- Artifact collection

### Phase 4: CLI + Integration (Week 4-5)
- CLI commands (enqueue, list, inspect, cancel, retry, schedule, drain)
- Queue to agent integration
- State transition reporting
- Error propagation

### Phase 5: Testing + Verification (Week 5-6)
- Unit tests for all components
- Integration tests for workflows
- End-to-end tests with real workflows
- Verification of all MVP requirements
- Performance and load testing

## Verification Strategy

Each MVP requirement will be verified through:

1. **Unit Tests**: Test individual components in isolation
2. **Integration Tests**: Test component interactions
3. **End-to-End Tests**: Test complete workflows
4. **Property-Based Tests**: Test invariants with random inputs
5. **Manual Verification**: CLI command execution and observation

## Success Criteria

The MVP is complete when:
- All 55 MVP requirements are implemented
- All 10 must-have success criteria pass
- Test coverage > 80%
- All end-to-end workflows execute successfully
- System survives process restart
- Logs trace all state transitions

## Next Steps

Start with Phase 1: Foundation
