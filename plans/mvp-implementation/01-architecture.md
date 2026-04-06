# Architecture Design with Stubbed Components

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    agent-queue (single process)                │
│                                                               │
│  ┌──────────┐    ┌──────────────┐    ┌──────────────────┐  │
│  │   CLI    │───▶│ Queue Engine  │───▶│ Stubbed Agent    │  │
│  │  (Clap)  │◀───│  (Scheduler)  │◀───│   Executor       │  │
│  └──────────┘    └──────┬───────┘    └────────┬─────────┘  │
│                         │                      │              │
│                    ┌────▼────┐         ┌───────▼───────┐    │
│                    │ SQLite  │         │  Mock LLM      │    │
│                    │  (WAL)  │         │  (Sleep sim)   │    │
│                    │         │         │  + failures    │    │
│                    └─────────┘         └───────────────┘    │
│                                              │              │
│                                       ┌──────┴───────┐     │
│                                       │ Mock Tools     │     │
│                                       │ (Latency sim)  │     │
│                                       └───────────────┘     │
│                                                               │
│  ┌──────────┐    ┌──────────────┐    ┌──────────────────┐  │
│  │  Logs    │    │  Artifacts   │    │  Audit Trail    │  │
│  │ (tracing)│    │  (files)     │    │  (append-only)  │  │
│  └──────────┘    └──────────────┘    └──────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

## Component Responsibilities

### CLI (src/cli/)
- Parse command-line arguments with Clap
- Enqueue workflows to queue
- List, inspect, cancel, retry tasks
- Validate workflows without execution
- Provide user feedback

### Queue Engine (src/queue/)
- **engine.rs**: Core queue engine with scheduler loop
- **scheduler.rs**: Meta-scheduler and task selection
- **categories.rs**: Queue category definitions (ASAP, Whenever, Scheduled, Cron)
- **lease.rs**: Lease management, heartbeat, expiry, reclaim
- **retry.rs**: Retry logic with exponential backoff
- **routing.rs**: Queue routing based on YAML metadata
- **backpressure.rs**: Admission control and queue depth limits

### State Machine (src/state/)
- **mod.rs**: Entity type definitions
- **machine.rs**: State machine with compile-time transitions
- **transitions.rs**: Valid transition table
- **store.rs**: SQLite persistence layer

### Stubbed Agent Executor (src/agent/mock/)
- **executor.rs**: Orchestrates step execution (STUBBED)
- **step_runner.rs**: Sequential step execution (STUBBED)
- **heartbeat.rs**: Lease heartbeat during execution (STUBBED)
- **context.rs**: Context window management (STUBBED)

### Mock Components (src/mock/)
- **llm_provider.rs**: Mock LLM with sleep simulation
  - Generates fake responses after 5-30s delay
  - Simulates 10% failure rate
  - Simulates context window exhaustion (1% chance)
- **tools.rs**: Mock tools with latency
  - shell: 1-3s delay
  - file_read: 0.5-1s delay
  - file_write: 1-2s delay

### Real Components (src/agent/)
These would connect to actual workflow engine (not needed for MVP):

```rust
// Placeholder for real executor
pub struct RealExecutor {
    // Would connect to transpiler workflow engine
}

// Stub executor for MVP
pub struct StubExecutor {
    // Uses sleep-based mocks
}
```

## Data Flow

### Enqueue Flow

```
CLI enqueue command
    │
    ├─▶ Parse and validate YAML
    ├─▶ Create workflow entity
    ├─▶ Create run entity
    ├─▶ Route to appropriate queue
    └─▶ Queue Engine: task → NEW → QUEUED
        │
        └─▶ Scheduler: selects task based on priority
            │
            └─▶ Queue Engine: task → LEASED
                │
                └─▶ Stubbed Agent Executor
                    │
                    ├─▶ Load workflow YAML
                    ├─▶ For each step:
                    │   ├─▶ Mock LLM call (5-30s)
                    │   ├─▶ Mock tool calls (1-5s)
                    │   ├─▶ Send heartbeat (every 15s)
                    │   └─▶ Collect artifacts
                    │
                    └─▶ Report completion
                        │
                        └─▶ Queue Engine: task → DONE or FAILED
                            │
                            └─▶ If FAILED: retry → DLQ
```

## Key Design Decisions

### 1. Single Process Architecture
- All components run in one Tokio runtime
- No inter-process communication
- Simplifies MVP development

### 2. Stubbed Execution Engine
- Simulates real execution with sleep-based delays
- Allows testing queue engine without real LLM
- Focus on queue semantics, not agent behavior

### 3. SQLite with WAL Mode
- Single-writer pattern
- ACID transactions
- Survives process restart

### 4. Structured Logging
- All logs as JSON with tracing
- Correlation IDs for all operations
- Enables post-mortem analysis

### 5. Trait-Based Abstractions
- LlmProvider trait for future real providers
- Tool trait for extensibility
- Easy to swap real implementation later

## Error Handling Strategy

### Errors are Classifiable

```rust
pub enum ErrorClass {
    Retryable,      // Transient: retry with backoff
    NonRetryable,   // Permanent: go to DLQ
    Fatal,          // System error: abort
}
```

### Error Recovery

1. **Retryable Errors**:
   - LLM timeout
   - Network timeout (if network tools existed)
   - Temporary resource exhaustion

2. **Non-Retryable Errors**:
   - Invalid YAML
   - Permission denied
   - Context window exceeded

3. **Fatal Errors**:
   - Database corruption
   - Out of memory
   - System crash

## Testing Strategy

### Unit Tests
- Test individual components in isolation
- Use mocks for external dependencies
- Fast execution (< 1s per test)

### Integration Tests
- Test component interactions
- Use in-memory SQLite
- Simulate real workflows

### End-to-End Tests
- Test complete user journeys
- Use real CLI commands
- Verify all state transitions

### Property-Based Tests
- Test invariants with random inputs
- Use proptest
- Find edge cases

## Performance Targets

| Metric | Target | Measurement |
|--------|--------|-------------|
| Enqueue latency | < 100ms | Time from CLI to QUEUED |
| Schedule latency | < 1s | Time from due to LEASED |
| Heartbeat overhead | < 10ms | Time to send heartbeat |
| State transition time | < 50ms | Time to write to SQLite |
| Log volume | < 1MB/hour | Structured log size |

## Monitoring and Observability

### Metrics to Track

1. **Queue Metrics**:
   - Queue depth per category
   - Tasks per second
   - Average wait time
   - Priority distribution

2. **Execution Metrics**:
   - Active tasks
   - Completion rate
   - Failure rate
   - Retry distribution

3. **System Metrics**:
   - CPU usage
   - Memory usage
   - Disk I/O
   - Database locks

### Logs to Emit

Every operation emits:
- `trace_id`: Correlation ID for entire workflow
- `run_id`: Run identifier
- `task_id`: Task identifier (for internal tasks)
- `step_id`: Step identifier
- `action`: What happened
- `state`: Before/after state
- `duration_ms`: How long it took
- `error`: Error details (if applicable)

## Security Considerations

### MVP Security

1. **Tool Allowlist**:
   - Default deny
   - Explicit allow per workflow
   - No network tools in MVP

2. **Path Restrictions**:
   - Tools limited to workspace
   - No absolute paths
   - No symlink traversal

3. **Secrets Management**:
   - Environment variables only
   - No secrets in YAML
   - No secrets in logs

### Future Security (Post-MVP)

- RBAC for multi-tenancy
- Rate limiting per user
- Quotas for resource usage
- Sandbox enforcement
- Audit log access control
