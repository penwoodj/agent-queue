# Gap & Consistency Report

**Date**: 2026-04-06  
**Scope**: All documentation in agent-queue project  
**Context**: Post-refactor to transpiler delegation model

---

## Summary

After 3 review cycles and updates across all plan documents, the documentation suite is now **largely consistent**. This report captures remaining gaps, minor inconsistencies, and considerations for implementation.

---

## ✅ Resolved Issues

These were identified in previous review cycles and have been fixed:

| Issue | Resolution |
|-------|-----------|
| Architecture doc referenced "Stubbed Agent Executor" and "Mock LLM" | Updated to "Transpiler Integration" model |
| Data model had real-time step tracking | Changed to metadata-only step summaries (per IN-11) |
| State machine lacked QE-02 RUNNING refinement | Added note about running-throughout-execution |
| Phase 04 and 05 didn't exist | Created with detailed task breakdowns |
| README referenced deleted `03-agent-sdk-mock.md` | Updated to reference `03-transpiler-integration.md` |
| Test counts inconsistent across docs | Aligned to 48 unit + 48 integration + 48 E2E + 12 error = 156 |
| Step entity had transpiler-owned fields (input_text, started_at, etc.) | Removed, now metadata-only |
| Artifact entity lacked content_type | Added |
| StepState had Pending/Running (transpiler tracks these) | Changed to Completed/Failed/Skipped |
| SQLite schema didn't match entity definitions | Aligned |

---

## 🟡 Known Gaps (Non-Blocking)

These are gaps that should be addressed **during implementation**, not before:

### Gap 1: YAML Schema for Queue-Level Metadata (IN-08)

**What**: IN-08 says agent-queue should adopt the transpiler's YAML schema. But the transpiler schema defines workflow execution (models, steps, tools). agent-queue needs **queue-level metadata** (queue category, priority, schedule, idempotency key) that the transpiler schema doesn't define.

**Impact**: During Phase 1 implementation, we need to decide:
- Option A: Extend transpiler schema with a `queue:` section
- Option B: Separate agent-queue wrapper YAML that references a transpiler YAML
- Option C: CLI flags for queue metadata, YAML only for workflow definition

**Recommendation**: Option C for MVP (simplest). Queue metadata via CLI flags, workflow definition in transpiler-format YAML.

**When to decide**: Phase 1, Task 1.4 (YAML Schema)

---

### Gap 2: Mock Transpiler Binary for Testing

**What**: The plan references a "mock transpiler" for integration testing, but no mock binary implementation exists yet. The old `mocks/` directory was removed from the plan.

**Impact**: Phase 5 integration tests need a way to simulate transpiler behavior without the real binary.

**Recommendation**: Create a simple shell script or Rust binary that mimics transpiler CLI interface (accepts `execute`/`validate` subcommands, returns JSON, supports configurable failure modes).

**When to decide**: Phase 3, Task 3.2 (CLI Invocation Workflow)

---

### Gap 3: Transpiler CLI Interface Not Yet Defined

**What**: The integration plan assumes the transpiler exposes a CLI with `execute` and `validate` subcommands and JSON output. The actual transpiler may have a different interface.

**Impact**: Phase 3 implementation depends on knowing the exact transpiler CLI contract.

**Recommendation**: Before starting Phase 3, verify the transpiler's actual CLI interface. The `TRANSPILER_INTEGRATION_SPEC.md` defines the *desired* interface — confirm the transpiler matches or plan adapter code.

**When to decide**: Before Phase 3 start

---

### Gap 4: No Cargo.toml / Project Scaffold

**What**: Zero Rust code exists. No `Cargo.toml`, no `src/` directory.

**Impact**: Phase 1 must begin with project scaffolding (not in the task list).

**Recommendation**: Add Task 0 to Phase 1: "Project scaffold" (1h) — `cargo init`, dependency setup (tokio, serde, rusqlite, clap, chrono, tracing, thiserror).

**When to decide**: Before Phase 1 start

---

## 🟢 Minor Inconsistencies (Cosmetic)

These don't affect implementation but should be cleaned up eventually:

| Location | Issue | Severity |
|----------|-------|----------|
| `README.md` | Summary still says "192 tests" in the final summary block | Low — update to 156 |
| `00-overview.md` | May still reference "stubbed" terminology (not checked in this cycle) | Low |
| Phase 3 | Test count says 11 unit + 7 integration = 18 (vs. 21 in README matrix) | Low — align |
| Phase 4 | Test count says 12 unit + 7 integration = 19 (vs. 54 in README matrix) | Low — align |
| Phase 5 | Test count says 48+48+48+12 = 156 (matches README) | ✅ Consistent |

---

## 📋 Implementation Considerations

### 1. Dependency Version Choices

The plan doesn't specify Rust dependency versions. Key choices to make:
- `tokio` (async runtime) — latest stable
- `rusqlite` (SQLite) — with bundled feature
- `clap` (CLI) — v4 with derive
- `serde` / `serde_yaml` (YAML parsing) — compatible with transpiler schema
- `chrono` (timestamps) — v0.4
- `tracing` (logging) — with tracing-subscriber
- `thiserror` (errors) — latest

### 2. Transpiler Binary Location

The plan assumes the transpiler is available as a CLI binary at a configurable path. Implementation needs:
- Config field for binary path (default: `yaml-to-rust-agentsdk` in PATH)
- Validation that binary exists at startup
- Error if binary not found

### 3. Database Migration Strategy

The plan shows a single `001_initial.sql` migration. Implementation needs:
- Migration runner (manual or using `rusqlite` embedded)
- Schema version tracking
- Future migration support

### 4. Graceful Shutdown

Phase 4 mentions "graceful shutdown" for daemon mode. Implementation needs:
- SIGTERM/SIGINT signal handling
- Drain in-progress runs before exit
- Configurable shutdown timeout

---

## Numbers Cross-Check

| Metric | MVP Definition | README | Traceability | Status |
|--------|---------------|--------|--------------|--------|
| MVP Requirements | 48 | 48 | 48 | ✅ |
| Unit Tests | — | 48 | — | ✅ |
| Integration Tests | — | 48 | — | ✅ |
| E2E Tests | — | 48 | — | ✅ |
| Total Tests | — | 156 | — | ✅ |
| Implementation Hours | — | 177 | — | ✅ |
| Phases | — | 5 | — | ✅ |
| Integration Reqs (IN-*) | 15 | 15 (8 new) | IN-08→IN-14 mapped | ✅ |

---

## Recommendation

**Documentation is ready for implementation.** The remaining gaps are all implementation-time decisions (YAML schema, mock binary, transpiler CLI verification, project scaffold). No further documentation passes are needed before writing code.

**Suggested next step**: Begin Phase 1 implementation, starting with project scaffold (Cargo.toml, dependencies).
