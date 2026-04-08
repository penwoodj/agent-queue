# MVP Success Criteria

## Must-Have Criteria (System is useful ONLY if all pass)

### 1. Enqueue a YAML workflow and it executes to completion

**Verification**: `test_full_workflow`

**Manual Steps**:
```bash
# 1. Create a simple workflow
cat > simple-workflow.yaml << 'EOF'
schema_version: "0.1.0"
name: simple-test
queue:
  category: asap
  priority: high
model:
  provider: llama-cpp
  model_path: ./models/llama-3.2-3b-q4_k_m.gguf
  temperature: 0.7
  max_tokens: 1000
tools:
  allowed: [file_read, shell]
  blocked: []
steps:
  - id: step-1
    description: "First step"
    prompt:
      user: "Process this input"
    timeout_s: 30
EOF

# 2. Enqueue the workflow
agent-queue enqueue simple-workflow.yaml

# 3. Wait for completion
sleep 45

# 4. Verify completion
agent-queue list --state done

# Expected: Task shows as DONE with completed_at timestamp
```

**Success Indicators**:
- Task enqueued successfully
- Task transitions: NEW → QUEUED → LEASED → RUNNING → DONE
- No errors in logs
- Duration recorded

**Failure Indicators**:
- Task stuck in LEASED or RUNNING
- Error in logs
- Task moved to FAILED or DLQ

---

### 2. ASAP tasks run before Whenever tasks

**Verification**: `test_priority_preemption`

**Manual Steps**:
```bash
# 1. Enqueue Whenever tasks first
agent-queue enqueue bg-task-1.yaml --queue whenever
agent-queue enqueue bg-task-2.yaml --queue whenever

# 2. Enqueue ASAP task
agent-queue enqueue urgent-task.yaml --queue asap --priority critical

# 3. Monitor execution
watch -n 1 'agent-queue list --state running'

# Expected: ASAP task runs first, then Whenever tasks
```

**Success Indicators**:
- ASAP task selected for execution before Whenever tasks
- ASAP task completes first
- Scheduler logs show priority-based selection

**Failure Indicators**:
- Whenever tasks run before ASAP
- No priority-based ordering

---

### 3. Failed tasks retry automatically and reach DLQ

**Verification**: `test_failure_retry`

**Manual Steps**:
```bash
# 1. Create a workflow that will fail (simulated)
cat > failing-workflow.yaml << 'EOF'
schema_version: "0.1.0"
name: failing-test
queue:
  category: asap
model:
  provider: llama-cpp
  model_path: ./models/llama-3.2-3b-q4_k_m.gguf
steps:
  - id: step-1
    prompt:
      user: "This will fail (simulated)"
    timeout_s: 10
EOF

# 2. Enqueue the workflow
agent-queue enqueue failing-workflow.yaml

# 3. Monitor retries
watch -n 5 'agent-queue inspect $(agent-queue list --state running | head -1 | cut -d" " -f1)'

# Expected: 3 retries with increasing delays, then DLQ
```

**Success Indicators**:
- Task fails on first attempt
- Automatically retries 2 more times
- Retry delays: ~1s → ~2s → ~4s
- After 3rd retry, moves to DLQ
- Audit log shows retry attempts

**Failure Indicators**:
- No retry occurs
- Retry count < 3
- Task doesn't reach DLQ

---

### 4. DLQ tasks can be inspected and requeued

**Verification**: `test_retry_from_dlq`

**Manual Steps**:
```bash
# 1. List DLQ tasks
agent-queue list --state dlq

# 2. Inspect a DLQ task
agent-queue inspect TASK-XXX

# Expected output:
# State: DLQ
# Attempts: 3/3
# Error: <error message>
# Step: <step_id>

# 3. Retry the task
agent-queue retry TASK-XXX --priority high --reason "manual retry"

# 4. Monitor new task
agent-queue inspect TASK-YYY

# Expected: New task in QUEUED or RUNNING state
```

**Success Indicators**:
- DLQ tasks visible in list
- Inspect shows error details and retry count
- Retry creates new task with reset attempts
- New task executes successfully

**Failure Indicators**:
- Cannot inspect DLQ tasks
- Retry command fails
- New task has same ID (should be new)

---

### 5. Cron workflow runs on schedule

**Verification**: `test_cron_workflow`

**Manual Steps**:
```bash
# 1. Create a cron workflow
cat > cron-workflow.yaml << 'EOF'
schema_version: "0.1.0"
name: cron-test
queue:
  category: cron
  schedule:
    cron: "*/1 * * * *"  # Every minute
    timezone: "UTC"
model:
  provider: llama-cpp
  model_path: ./models/llama-3.2-3b-q4_k_m.gguf
steps:
  - id: step-1
    prompt:
      user: "Cron task execution"
    timeout_s: 30
EOF

# 2. Register the cron workflow
agent-queue schedule cron-workflow.yaml

# 3. Watch for executions
watch -n 30 'agent-queue list --queue cron --state done'

# Expected: New task every minute
```

**Success Indicators**:
- Cron scheduled correctly
- Tasks appear in cron queue
- Tasks execute every minute
- Due time gates respected

**Failure Indicators**:
- No scheduled tasks created
- Tasks execute at wrong times
- Duplicate executions

---

### 6. System survives process restart

**Verification**: `test_restart_recovery`

**Manual Steps**:
```bash
# 1. Start queue engine
agent-queue run &
ENGINE_PID=$!

# 2. Enqueue multiple tasks
for i in {1..5}; do
  agent-queue enqueue test-$i.yaml --queue asap
done

# 3. Check queued tasks
agent-queue list --state queued
# Expected: 5 tasks queued

# 4. Kill the engine
kill $ENGINE_PID

# 5. Restart the engine
agent-queue run &
ENGINE_PID=$!

# 6. Wait and check queued tasks
sleep 5
agent-queue list --state queued
# Expected: Same 5 tasks still queued (or some running)
```

**Success Indicators**:
- All queued tasks persist in database
- Tasks resume execution after restart
- No duplicate tasks created
- State machine integrity maintained

**Failure Indicators**:
- Tasks lost after restart
- Database corrupted
- Duplicate tasks

---

### 7. Structured logs trace every state transition

**Verification**: `test_log_traces`

**Manual Steps**:
```bash
# 1. Start engine with logs
agent-queue run > /tmp/agent-queue.log 2>&1 &
ENGINE_PID=$!

# 2. Enqueue a task
agent-queue enqueue test.yaml

# 3. Wait for completion
sleep 45

# 4. Search logs for task lifecycle
grep "trace_id" /tmp/agent-queue.log | jq '.'

# Expected output:
# {
#   "trace_id": "xxx",
#   "run_id": "run-xxx",
#   "action": "state_transition",
#   "from": "queued",
#   "to": "leased",
#   "timestamp": "..."
# }
```

**Success Indicators**:
- All state transitions logged
- Each log has trace_id, run_id, action
- Logs are valid JSON
- Full lifecycle traceable by run_id

**Failure Indicators**:
- Missing state transitions
- Logs not JSON
- No correlation IDs

---

### 8. Artifact output is persisted and accessible

**Verification**: `test_artifact_files`

**Manual Steps**:
```bash
# 1. Create workflow with output
cat > artifact-workflow.yaml << 'EOF'
schema_version: "0.1.0"
name: artifact-test
queue:
  category: asap
tools:
  allowed: [file_write]
steps:
  - id: step-1
    prompt:
      user: "Generate output file"
    tools: [file_write]
EOF

# 2. Enqueue and wait
RUN_ID=$(agent-queue enqueue artifact-workflow.yaml | grep -o 'run-[^ ]*')
sleep 45

# 3. Check artifacts directory
ls -la ./artifacts/$RUN_ID/

# Expected: step-1/output.txt or similar
```

**Success Indicators**:
- Artifact directory created
- Output files present
- File content matches mock output
- Artifact metadata stored in database

**Failure Indicators**:
- No artifact directory
- Empty artifacts
- Corrupted files

---

### 9. CLI validate catches invalid YAML

**Verification**: `test_validation_cli`

**Manual Steps**:
```bash
# 1. Create invalid YAML
cat > invalid.yaml << 'EOF'
schema_version: "0.1.0"
name: invalid-test
queue:
  category: invalid-category  # Invalid
steps: []  # Missing steps
EOF

# 2. Try to validate
agent-queue validate invalid.yaml

# Expected: Non-zero exit code with error message
# "Error: Unknown queue category: invalid-category"
# "Error: Missing required field: steps"
```

**Success Indicators**:
- Invalid YAML detected
- Clear error message
- Non-zero exit code
- Valid YAML passes

**Failure Indicators**:
- Invalid YAML passes validation
- Vague error messages
- Exit code 0 for invalid YAML

---

### 10. Cancel stops a queued or running task

**Verification**: `test_cancel_cli`

**Manual Steps**:
```bash
# 1. Enqueue a task
RUN_ID=$(agent-queue enqueue test.yaml | grep -o 'run-[^ ]*')

# 2. Cancel the task
agent-queue cancel $RUN_ID --reason "user cancellation"

# 3. Check task state
agent-queue inspect $RUN_ID

# Expected output:
# State: CANCELED
# Reason: "user cancellation"
```

**Success Indicators**:
- Task transitions to CANCELED
- If running, stop signal sent
- Audit log records cancellation
- Task doesn't execute further

**Failure Indicators**:
- Cancel command fails
- Task continues running
- State not updated

---

## Should-Have Criteria (System is significantly better if these pass)

### 11. Lease expiry reclaims stuck tasks

**Verification**: `test_lease_recovery`

**Manual Steps**:
```bash
# 1. Start engine
agent-queue run &

# 2. Enqueue task and capture run_id
RUN_ID=$(agent-queue enqueue long-running.yaml | grep -o 'run-[^ ]*')

# 3. Wait for task to start
sleep 2

# 4. Find agent process and kill it (simulate crash)
# (This is tricky - we need to block heartbeat)
# For testing, we can manually set lease_expiry in DB)
sqlite3 agent-queue.db "UPDATE runs SET lease_expiry = datetime('now', '-60 seconds') WHERE id = '$RUN_ID'"

# 5. Wait for scheduler tick
sleep 5

# 6. Check task state
agent-queue inspect $RUN_ID

# Expected: State is QUEUED (reclaimed) or LEASED (re-leased)
```

**Success Indicators**:
- Expired lease detected
- Task reclaimed to QUEUED
- Available for re-scheduling
- Audit log shows reclaim

---

### 12. Backpressure rejects when queue is full

**Verification**: `test_backpressure_cli`

**Manual Steps**:
```bash
# 1. Configure small queue depth for testing
# (Edit config or use CLI flag)

# 2. Fill queue to limit
for i in {1..1000}; do
  agent-queue enqueue fill-$i.yaml --queue asap
done

# 3. Try to enqueue one more
agent-queue enqueue extra.yaml --queue asap

# Expected: Error "Queue full: 1000/1000"
```

**Success Indicators**:
- Rejection when queue at limit
- Clear error message
- Current depth reported
- Rejected task not enqueued

---

### 13. Idempotency key prevents duplicate enqueue

**Verification**: `test_idempotency_cli`

**Manual Steps**:
```bash
# 1. Enqueue with idempotency key
agent-queue enqueue test.yaml --idempotency-key "unique-key-123"

# 2. Try to enqueue again with same key
agent-queue enqueue test.yaml --idempotency-key "unique-key-123"

# Expected: Error "Idempotency key already exists: unique-key-123"
```

**Success Indicators**:
- First enqueue succeeds
- Second enqueue rejected
- Clear error about duplicate key
- Original task executes normally

---

### 14. Audit log records all state changes

**Verification**: `test_audit_query`

**Manual Steps**:
```bash
# 1. Enqueue and run a task
RUN_ID=$(agent-queue enqueue test.yaml | grep -o 'run-[^ ]*')
sleep 45

# 2. Query audit log
sqlite3 agent-queue.db "SELECT * FROM audit_log WHERE entity_id = '$RUN_ID' ORDER BY ts ASC"

# Expected: Multiple entries for all state transitions:
# - enqueue (NEW → QUEUED)
# - lease (QUEUED → LEASED)
# - start (LEASED → RUNNING)
# - complete (RUNNING → DONE)
```

**Success Indicators**:
- All state transitions recorded
- Timestamps in order
- Actor tracked (system/agent/cli)
- Diff JSON captures changes

---

### 15. Step-level errors show which step failed

**Verification**: `test_inspect_steps`

**Manual Steps**:
```bash
# 1. Run a failing workflow
RUN_ID=$(agent-queue enqueue failing-workflow.yaml | grep -o 'run-[^ ]*')
sleep 45

# 2. Inspect with steps
agent-queue inspect $RUN_ID --steps

# Expected output:
# Steps:
#   step-1: FAILED
#     Error: <error message>
#     Attempt: 1/3
#     Duration: 30s
```

**Success Indicators**:
- Each step listed with state
- Failed step identified
- Error message shown
- Retry count per step
- Duration per step

---

## Acceptance Criteria Summary

### MVP Complete When

- **All 10 must-have criteria pass**
- **At least 13 of 15 should-have criteria pass** (87%)
- **Test coverage > 80%**
- **All 48 MVP requirements verified**

### Sign-Off Checklist

- [ ] Unit tests: 48/48 pass
- [ ] Integration tests: 48/48 pass
- [ ] E2E tests: 48/48 pass
- [ ] Must-have criteria: 10/10 pass
- [ ] Should-have criteria: XX/15 pass
- [ ] Code coverage: XX% (> 80% target)
- [ ] Documentation complete
- [ ] All requirements verified

### Final Decision

**[ ] MVP COMPLETE - Ready for deployment**
**[ ] MVP INCOMPLETE - Requires fixes**

**If incomplete**, list blocking issues:

1.
2.
3.
