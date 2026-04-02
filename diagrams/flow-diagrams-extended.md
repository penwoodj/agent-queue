# Additional Flow Diagrams for Agent Queue System

## 10 more system flow diagrams (11–20)

### 11) Cost-aware scheduling (min-cost subject to SLA/deadlines)

```mermaid
flowchart TD
  A[Task QUEUED] --> B[Gather candidates: agent pools + compute classes]
  B --> C[Fetch pricing + capacity signals<br/>spot/on-demand, node availability]
  C --> D[Compute constraints<br/>capabilities, sandbox, quotas, rate limits]
  D --> E{Deadline/SLA present?}
  E -->|yes| F[Compute latest_start_ts = deadline - worst_case_runtime]
  E -->|no| G[Set cost-only objective]
  F --> H{Now > latest_start_ts?}
  H -->|yes| I[Escalate: pick fastest class<br/>ignore cost preference]
  H -->|no| J[Score each option: cost + risk + sla_penalty]
  G --> J
  I --> K[Select best feasible option]
  J --> K
  K --> L[Route to queue/pool + lease when eligible]
  L --> M[Emit audit: placement_decision + cost_estimate]
```

### 12) Spillover queues (admission control + overflow routing)

```mermaid
flowchart TD
  A[Enqueue request] --> B[Check primary queue depth + caps]
  B --> C{Primary has capacity?}
  C -->|yes| D[Enqueue to primary queue]
  C -->|no| E{Spillover policy exists?}
  E -->|no| F[Reject or hold PENDING_ADMISSION]
  E -->|yes| G[Select spillover target<br/>Whenever/Overflow/Deferred]
  G --> H[Attach spillover metadata<br/>original_queue, reason, ts]
  H --> I[Enqueue to spillover queue]
  I --> J[Notify + metrics: spillover_count]
  D --> K[Normal processing]
  F --> L[Audit + notify]
```

### 13) Work stealing (utilization without breaking policy)

```mermaid
flowchart TD
  A[Agent idle] --> B[Check own pool queues]
  B --> C{Work found?}
  C -->|yes| D[Lease work normally]
  C -->|no| E{Stealing enabled?}
  E -->|no| F[Sleep/backoff]
  E -->|yes| G[Enumerate steal candidates<br/>allowed pools/queues only]
  G --> H[Apply constraints<br/>tenant isolation, sandbox, capabilities]
  H --> I[Pick victim queue by policy<br/>oldest_age or depth]
  I --> J[Attempt lease with steal_tag=true]
  J --> K{Lease success?}
  K -->|no| F
  K -->|yes| L[Execute + emit steal metrics]
```

### 14) Batch formation (batch_key + window + size)

```mermaid
flowchart TD
  A[Task arrives with batch_key] --> B[Lookup batch buffer by key]
  B --> C[Append task to buffer]
  C --> D{Buffer size >= batch_size?}
  D -->|yes| E[Seal batch immediately]
  D -->|no| F{Batch window expired?}
  F -->|yes| G[Seal batch on timeout]
  F -->|no| H[Wait for more tasks]
  E --> I[Create BatchTask with members list]
  G --> I
  I --> J[Enqueue BatchTask -> queue]
  J --> K[On execution: process members deterministically]
  K --> L[Per-member results stored + mapped back]
```

### 15) Maintenance windows (hard no-run outside window)

```mermaid
flowchart TD
  A[Task routed to Maintenance queue] --> B[Validate maintenance window rules]
  B --> C[Set eligible_window: start/end + timezone]
  C --> D{Now inside window?}
  D -->|no| E[Task remains QUEUED but ineligible]
  D -->|yes| F[Meta-scheduler includes Maintenance queue]
  F --> G[Agent leases task]
  G --> H[Execute with maintenance guardrails]
  H --> I{Window ends mid-run?}
  I -->|no| J[Complete normally]
  I -->|yes| K{Policy: allow_overrun?}
  K -->|yes| J
  K -->|no| L[Checkpoint + pause task -> QUEUED for next window]
```

### 16) Schema migration (workflow.yml and runtime state)

```mermaid
flowchart TD
  A[New schema_version released] --> B[Publish migration spec]
  B --> C[CI validates: forward/backward compatibility]
  C --> D[Registry marks schema_version as current]
  D --> E{Existing workflows old version?}
  E -->|no| F[No action]
  E -->|yes| G[Choose migration mode]
  G -->|lazy at load| H[On load: transform workflow to current]
  G -->|eager batch| I[Background migrate stored defs]
  H --> J[Validate transformed YAML]
  I --> J
  J --> K{Valid?}
  K -->|no| L[Quarantine + report errors]
  K -->|yes| M[Write migrated version + keep original]
  M --> N[Emit audit: migration_applied]
```

### 17) Exactly-once outcome (outbox/inbox pattern around side effects)

```mermaid
flowchart TD
  A[Task step wants side effect<br/>send email / write external API] --> B[Compute idempotency_key]
  B --> C[Write intent to durable Outbox<br/>state=PREPARED]
  C --> D[Commit task state_version]
  D --> E[Outbox dispatcher reads PREPARED]
  E --> F[Call external system with idempotency_key]
  F --> G{External ack?}
  G -->|yes| H[Mark outbox item SENT + store receipt]
  G -->|no| I[Retry outbox with backoff]
  H --> J[Task step observes SENT and continues]
  I --> E
  J --> K[Task completes; replay-safe because outbox is durable]
```

### 18) Cancellation propagation (run/task/step scopes)

```mermaid
flowchart TD
  A[Cancel requested] --> B[AuthZ + audit cancel intent]
  B --> C{Cancel scope}
  C -->|Run| D[Mark run.cancel=true]
  C -->|Task| E[Mark task.cancel=true]
  C -->|Step| F[Mark step.cancel=true]
  D --> G[Propagate to all non-terminal tasks]
  E --> H[If QUEUED -> remove from queue]
  E --> I[If LEASED/RUNNING -> send cancel signal]
  I --> J{Agent supports cooperative cancel?}
  J -->|yes| K[Agent checkpoints + exits]
  J -->|no| L[Force timeout -> reclaim + mark CANCELED]
  K --> M[Set terminal state CANCELED]
  H --> M
  F --> N[Stop further steps + mark task canceled]
  N --> M
```

### 19) Run-level pause/resume (freeze orchestration without losing ordering)

```mermaid
flowchart TD
  A[Operator/User pauses Run] --> B[AuthZ + audit]
  B --> C[Set run.paused=true]
  C --> D[Mark child tasks ineligible for lease]
  D --> E{Any tasks currently RUNNING?}
  E -->|yes| F[Let finish or checkpoint+pause (policy)]
  E -->|no| G[All queued tasks remain queued but blocked]
  F --> H[Update states accordingly]
  G --> I[Resume requested]
  I --> J[Set run.paused=false]
  J --> K[Recompute eligibility + preserve ordering keys]
  K --> L[Meta-scheduler sees tasks again]
```

### 20) Observability + alerting loop (metrics → alerts → automated mitigations)

```mermaid
flowchart TD
  A[Agents emit metrics/logs/traces] --> B[Collector/OTel pipeline]
  B --> C[Storage: TSDB + log store + trace store]
  C --> D[Rules engine evaluates SLOs<br/>latency, error rate, lease expiries, DLQ rate]
  D --> E{Breach detected?}
  E -->|no| F[Dashboards only]
  E -->|yes| G[Create Incident + notify on-call]
  G --> H{Auto-mitigation enabled?}
  H -->|no| I[Human triage]
  H -->|yes| J[Execute playbook<br/>scale pool, adjust weights, pause queue, shed load]
  J --> K[Audit mitigation action]
  K --> L[Re-evaluate rules]
  L --> D
```

---

## 10 more user-story flow diagrams (US11–US20)

### US11) "Minimize cost but still meet an SLA deadline"

```mermaid
flowchart TD
  A[User submits job with deadline_ts] --> B[Router tags: cost_aware=true + deadline]
  B --> C[Planner estimates runtime]
  C --> D[Scheduler computes latest_start_ts]
  D --> E{Capacity cheap enough before latest_start?}
  E -->|yes| F[Wait in low-cost pool until eligible]
  E -->|no| G[Escalate to faster pool]
  F --> H[Lease + run]
  G --> H
  H --> I[Finish before deadline]
  I --> J[Store cost_actual + audit placement]
```

### US12) "Primary queue full; spill to overflow; later rehydrate back"

```mermaid
flowchart TD
  A[User enqueues burst of tasks] --> B[Primary queue hits max_depth]
  B --> C[Spillover policy routes to Overflow queue]
  C --> D[Tasks execute best-effort]
  D --> E{Primary capacity returns?}
  E -->|yes| F[Rehydrate: move remaining overflow tasks back]
  E -->|no| G[Continue overflow processing]
  F --> H[Preserve original ordering keys]
  H --> I[Complete all tasks]
```

### US13) "Idle GPU agent steals compatible work from CPU pool"

```mermaid
flowchart TD
  A[GPU agent idle] --> B[Own queue empty]
  B --> C[Steal policy allows GPU->CPU for compatible tasks]
  C --> D[Find candidate tasks requiring no GPU]
  D --> E[Lease with steal_tag]
  E --> F[Execute task]
  F --> G[Emit metric: work_steal_success]
  G --> H[System utilization improves]
```

### US14) "Batch 1,000 small embedding tasks into batches of 50"

```mermaid
flowchart TD
  A[1000 tasks enqueued with batch_key=embed:modelX] --> B[Batch buffer accumulates]
  B --> C{buffer size==50 or window expired}
  C -->|size==50| D[Seal batch]
  C -->|window expired| E[Seal partial batch]
  D --> F[BatchTask runs once for 50 items]
  E --> F
  F --> G[Results mapped back to each original task]
  G --> H[All tasks marked DONE]
```

### US15) "Run database migration only during maintenance window"

```mermaid
flowchart TD
  A[Operator schedules migration workflow] --> B[Router selects Maintenance queue + window]
  B --> C[Before window: tasks ineligible]
  C --> D[Window opens]
  D --> E[Agent leases and runs]
  E --> F{Window closes mid-run?}
  F -->|no| G[Finish migration]
  F -->|yes| H[Checkpoint + defer]
  H --> I[Next window resumes]
  G --> J[Audit + publish migration report]
```

### US16) "Upgrade schema; old workflows migrate on load; invalid ones quarantined"

```mermaid
flowchart TD
  A[Team deploys schema v2] --> B[User loads old workflow v1]
  B --> C[Lazy migration transforms v1->v2]
  C --> D[Validate transformed workflow]
  D --> E{Valid?}
  E -->|yes| F[Run proceeds normally]
  E -->|no| G[Quarantine + show validation errors]
  G --> H[User fixes workflow and re-registers]
  F --> I[Audit: migration_applied]
```

### US17) "Exactly-once external API write despite retries"

```mermaid
flowchart TD
  A[Task step prepares external POST] --> B[Write Outbox PREPARED with idempotency_key]
  B --> C[Worker calls external API]
  C --> D{Network failure?}
  D -->|yes| E[Outbox retries]
  D -->|no| F[External ACK]
  E --> C
  F --> G[Mark Outbox SENT]
  G --> H[Task continues; replays observe SENT and skip duplicate]
  H --> I[Outcome exactly-once]
```

### US18) "Cancel a run; queued tasks removed; running tasks checkpoint and stop"

```mermaid
flowchart TD
  A[User presses Cancel Run] --> B[RBAC allows cancel]
  B --> C[Mark run.cancel=true]
  C --> D[Remove queued tasks from queues]
  C --> E[Signal running agents]
  E --> F{Agent cooperative cancel?}
  F -->|yes| G[Checkpoint + stop]
  F -->|no| H[Timeout -> reclaim -> mark canceled]
  G --> I[Run final state = CANCELED]
  H --> I
```

### US19) "Pause a run mid-flight; later resume without losing ordering"

```mermaid
flowchart TD
  A[Operator pauses run for investigation] --> B[Mark run.paused=true]
  B --> C[Queued tasks become ineligible]
  B --> D[Running task finishes or checkpoints (policy)]
  D --> E[System holds stable ordering keys]
  E --> F[Operator resumes run]
  F --> G[Mark run.paused=false]
  G --> H[Tasks regain eligibility]
  H --> I[Backlog continues deterministically]
```

### US20) "Alert on DLQ spike; auto-mitigate by pausing risky queue and scaling pool"

```mermaid
flowchart TD
  A[DLQ rate increases] --> B[Rules engine detects breach]
  B --> C[Incident created + on-call notified]
  C --> D[Auto-playbook enabled]
  D --> E[Pause risky queue]
  D --> F[Scale safer agent pool]
  F --> G[Reduce error rate]
  E --> H[Prevent more bad executions]
  H --> I[On-call reviews, applies fix, resumes queue]
  I --> J[Audit mitigation + resolution]
```

If you want, I can also output these as **one bundled "diagram pack" YAML** (each diagram as a named string field) so your coding agent can ingest them programmatically.
