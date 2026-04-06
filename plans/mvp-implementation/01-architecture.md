# Architecture Design with Transpiler Integration

## High-Level Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                      agent-queue (single process)                   │
│                                                                      │
│  ┌──────────┐    ┌──────────────┐    ┌───────────────────────────┐ │
│  │   CLI    │───▶│ Queue Engine  │───▶│ Transpiler Integration    │ │
│  │  (Clap)  │◀───│  (Scheduler)  │◀───│ (CLI invocation / API)    │ │
│  └──────────┘    └──────┬───────┘    └─────────────┬─────────────┘ │
│                         │                          │                │
│                    ┌────▼────┐               ┌──────▼────────────┐  │
│                    │ SQLite  │               │ yaml-to-rust-     │  │
│                    │  (WAL)  │               │ agentsdk          │  │
│                    │         │               │ (external process) │  │
│                    └─────────┘               └──────┬────────────┘  │
│                                                     │               │
│                                              ┌──────┴──────────┐   │
│                                              │ Artifacts        │   │
│                                              │ (metadata-only)  │   │
│                                              └─────────────────┘   │
│                                                                      │
│  ┌──────────┐    ┌──────────────┐    ┌──────────────────────────┐  │
│  │  Logs    │    │   Audit      │    │  Heartbeat Manager       │  │
│  │ (tracing)│    │  Trail       │    │  (lease keep-alive)      │  │
│  └──────────┘    │(append-only) │    └──────────────────────────┘  │
│                  └──────────────┘                                   │
└──────────────────────────────────────────────────────────────────────┘
```

> **Key Principle**: agent-queue handles **queue orchestration** (scheduling, state, retry, DLQ, audit). Workflow **execution** is fully delegated to `yaml-to-rust-agentsdk`. agent-queue never reimplements LLM calls, tool execution, or step orchestration.

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
- **retry.rs**: Retry logic with exponential backoff (queue-level only — per IN-09)
- **routing.rs**: Queue routing based on YAML metadata
- **backpressure.rs**: Admission control and queue depth limits

### State Machine (src/state/)
- **mod.rs**: Entity type definitions
- **machine.rs**: State machine with compile-time transitions
- **transitions.rs**: Valid transition table
- **store.rs**: SQLite persistence layer

### Transpiler Integration (src/transpiler/)
- **integration.rs**: CLI invocation interface to yaml-to-rust-agentsdk
- **heartbeat.rs**: Lease heartbeat during long-running transpiler executions
- **result.rs**: Result parsing, error classification, artifact extraction
- **env.rs**: Environment variable resolution and passthrough

> **Not in agent-queue**: No LLM provider trait, no tool execution, no step runner.
> These are all handled by yaml-to-rust-agentsdk. See
> [TRANSPILER_INTEGRATION_SPEC.md](../../docs/TRANSPILER_INTEGRATION_SPEC.md) for the full contract.

## Data Flow

### Enqueue Flow

```
CLI enqueue command
    │
    ├─▶ Parse and validate YAML
    ├─▶ Create workflow entity
    ├─▶ Create run entity
    ├─▶ Route to appropriate queue
    └─▶ Queue Engine: run → NEW → QUEUED
        │
        └─▶ Scheduler: selects run based on priority
            │
            └─▶ Queue Engine: run → LEASED
                │
                └─▶ Transpiler Integration
                    │
                    ├─▶ Invoke yaml-to-rust-agentsdk (CLI subprocess)
                    ├─▶ Send heartbeat (every 15s, maintains lease)
                    └─▶ Collect result + artifact metadata
                        │
                        └─▶ Report completion/failure
                            │
                            └─▶ Queue Engine: run → DONE or FAILED
                                │
                                └─▶ If FAILED: retry → DLQ
```

> **Note**: agent-queue does NOT step through individual workflow steps.
> The transpiler handles all step execution internally. agent-queue only sees
> the final result (success/failure, artifact list, error details).

## Key Design Decisions

### 1. Single Process Architecture
- All components run in one Tokio runtime
- Transpiler invoked as external CLI subprocess
- No inter-process communication needed beyond subprocess stdout/stderr

### 2. Transpiler Delegation (not stubbed execution)
- Workflow execution fully delegated to `yaml-to-rust-agentsdk`
- No mock LLM, no mock tools, no stubbed step runner in agent-queue
- Agent-queue is a queue orchestrator, not an agent runtime
- Per IN-08: adopts transpiler's YAML schema for workflow definitions
- Per IN-09: disables transpiler internal retry (queue handles retry)
- Per IN-10: tool execution is entirely within transpiler scope

### 3. SQLite with WAL Mode
- Single-writer pattern
- ACID transactions
- Survives process restart

### 4. Structured Logging
- All logs as JSON with tracing
- Correlation IDs for all operations
- Enables post-mortem analysis
- Per IN-14: log format compatible with transpiler's 9-level hierarchy

### 5. Trait-Based Transpiler Abstraction
- `TranspilerIntegration` trait for invoking the transpiler
- CLI invocation for MVP, library API for post-MVP
- Easy to swap invocation strategy or add caching

## Error Handling Strategy

### Errors are Classifiable

```rust
pub enum ErrorClass {
    Retryable,      // Transient: retry with backoff (transpiler timeout, LLM rate limit)
    NonRetryable,   // Permanent: go to DLQ (invalid YAML, auth failure)
    Fatal,          // System error: abort (database corruption, out of memory)
}
```

### Error Classification Protocol (per IN-13)

Errors from the transpiler are classified using a shared protocol:

1. **Transpiler reports error**: Includes `error_type`, `message`, `retryable` flag
2. **Queue classifies**: Maps transpiler error to queue error class
3. **Queue acts**: Retryable → retry with backoff, NonRetryable → DLQ, Fatal → abort

```rust
// Error mapping examples:
// Transpiler "LLMRateLimit"   → Queue Retryable
// Transpiler "Timeout"        → Queue Retryable
// Transpiler "InvalidYAML"    → Queue NonRetryable
// Transpiler "AuthFailed"     → Queue NonRetryable
// Transpiler "DiskFull"       → Queue Fatal
```

### Error Recovery

1. **Retryable Errors** (queue handles retry, per IN-09 transpiler retry is disabled):
   - Transpiler process timeout
   - LLM provider rate limiting (reported by transpiler)
   - Network connectivity (transpiler subprocess fails to start)
   - Temporary resource exhaustion

2. **NonRetryable Errors** (sent to DLQ):
   - Invalid workflow YAML (caught by transpiler validation)
   - Authentication failure (API keys missing/invalid)
   - Transpiler reports non-retryable execution error

3. **Fatal Errors** (abort):
   - Database corruption
   - Out of memory
   - System crash

## Testing Strategy

### Unit Tests
- Test individual components in isolation
- Mock transpiler CLI interface (not mock LLM — agent-queue never touches LLM)
- Fast execution (< 1s per test)

### Integration Tests
- Test component interactions
- Use mock transpiler binary or in-memory SQLite
- Simulate real transpiler responses (success, failure, timeout)

### End-to-End Tests
- Test complete user journeys
- Use real CLI commands
- Verify all state transitions
- May use real transpiler for smoke tests (optional)

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

1. **Subprocess Isolation**:
   - Transpiler runs as separate process
   - Environment variables passed explicitly (never inherited)
   - No secrets stored in database or logs

2. **Path Restrictions**:
   - Artifacts tracked in managed location
   - Working directory scoped per execution
   - No symlink traversal for artifact paths

3. **Secrets Management**:
   - Environment variables only
   - No secrets in YAML
   - No secrets in logs
   - Env var names stored, resolved values never persisted

### Future Security (Post-MVP)

- RBAC for multi-tenancy
- Rate limiting per user
- Quotas for resource usage
- Sandbox enforcement
- Audit log access control
