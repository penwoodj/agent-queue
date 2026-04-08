# Mock Transpiler Binary

**Purpose**: Simulate `yaml-to-rust-agentsdk` CLI for integration and E2E testing without requiring the actual transpiler installation.

**Location**: `tests/mock-transpiler/src/main.rs` (built as a separate binary for tests)

---

## CLI Interface Contract

The mock must match the interface defined in `docs/TRANSPILER_INTEGRATION_SPEC.md`:

### `execute` Subcommand

```bash
mock-transpiler execute \
  --workflow <path-to-yaml> \
  --format json \
  [ENV_VARS...]
```

**Behavior**:
- Reads the YAML file (validates it exists)
- Simulates execution with configurable delay
- Writes JSON result to stdout
- Writes JSON error to stderr on failure
- Exit code 0 on success, 1 on failure, 124 on timeout

**Output Format** (stdout on success):
```json
{
  "run_id": "mock-001",
  "status": "completed",
  "duration_ms": 5000,
  "steps": [
    {
      "step_id": "step-1",
      "status": "completed",
      "duration_ms": 3000,
      "token_usage": {
        "prompt_tokens": 100,
        "completion_tokens": 50,
        "total_tokens": 150
      },
      "output": "Mock output from step 1"
    },
    {
      "step_id": "step-2",
      "status": "completed",
      "duration_ms": 2000,
      "token_usage": {
        "prompt_tokens": 200,
        "completion_tokens": 100,
        "total_tokens": 300
      },
      "output": "Mock output from step 2"
    }
  ],
  "artifacts": [
    {
      "step_id": "step-2",
      "path": "/tmp/mock-artifacts/output.md",
      "size_bytes": 1024
    }
  ],
  "error": null
}
```

**Output Format** (stderr on failure):
```json
{
  "error_type": "ContextWindowExceeded",
  "message": "Context window exceeded at step step-2",
  "step_id": "step-2",
  "retryable": true,
  "details": {}
}
```

### `validate` Subcommand

```bash
mock-transpiler validate \
  --workflow <path-to-yaml> \
  --format json
```

**Behavior**:
- Reads the YAML file
- Checks basic structure (has name, schema_version)
- Writes JSON result to stdout
- Exit code 0 on valid, 1 on invalid

**Output Format** (stdout on valid):
```json
{
  "valid": true,
  "errors": []
}
```

**Output Format** (stderr on invalid):
```json
{
  "valid": false,
  "errors": [
    {
      "path": "name",
      "message": "field is required",
      "severity": "error"
    }
  ]
}
```

---

## Configuration (via environment variables)

| Variable | Default | Description |
|----------|---------|-------------|
| `MOCK_TRANSPILER_DELAY_MS` | `5000` | Execution delay in milliseconds |
| `MOCK_TRANSPILER_STEPS` | `2` | Number of mock steps to generate |
| `MOCK_TRANSPILER_FAIL_RATE` | `0.0` | Failure rate (0.0-1.0). Failures are retryable. |
| `MOCK_TRANSPILER_NONRETRYABLE_RATE` | `0.0` | Non-retryable failure rate |
| `MOCK_TRANSPILER_TIMEOUT` | `0` | Simulate timeout (exit code 124) if set to 1 |
| `MOCK_TRANSPILER_CRASH` | `0` | Simulate crash (signal) if set to 1 |
| `MOCK_TRANSPILER_ARTIFACTS_DIR` | `/tmp/mock-artifacts` | Directory to write artifact files |

---

## Usage in Tests

### Unit/Integration Tests

```rust
// In test setup
fn mock_transpiler_path() -> PathBuf {
    // Build mock transpiler binary
    let status = std::process::Command::new("cargo")
        .args(["build", "--package", "mock-transpiler"])
        .status()
        .expect("Failed to build mock transpiler");

    assert!(status.success());
    PathBuf::from(env!("CARGO_MANIFEST_DIR"))
        .join("../target/debug/mock-transpiler")
}

fn mock_transpiler_config() -> TranspilerConfig {
    TranspilerConfig {
        binary_path: mock_transpiler_path(),
        working_dir: tempfile::tempdir().unwrap().into_path(),
        timeout: Duration::from_secs(30),
    }
}
```

### Configurable Failure Scenarios

```rust
#[tokio::test]
async fn test_retryable_failure() {
    let config = mock_transpiler_config();
    let mut cmd = Command::new(&config.binary_path);
    cmd.env("MOCK_TRANSPILER_FAIL_RATE", "1.0");  // Always fail

    // Queue should retry, then move to DLQ
}

#[tokio::test]
async fn test_nonretryable_failure() {
    let config = mock_transpiler_config();
    let mut cmd = Command::new(&config.binary_path);
    cmd.env("MOCK_TRANSPILER_NONRETRYABLE_RATE", "1.0");

    // Queue should send directly to DLQ
}

#[tokio::test]
async fn test_timeout() {
    let config = mock_transpiler_config();
    let mut cmd = Command::new(&config.binary_path);
    cmd.env("MOCK_TRANSPILER_TIMEOUT", "1");

    // Queue should treat as retryable failure
}

#[tokio::test]
async fn test_crash() {
    let config = mock_transpiler_config();
    let mut cmd = Command::new(&config.binary_path);
    cmd.env("MOCK_TRANSPILER_CRASH", "1");

    // Queue should detect process death and reclaim lease
}
```

---

## Implementation Notes

1. **Separate Cargo workspace member**: The mock transpiler should be a separate crate in the workspace (not a dependency of agent-queue itself) to avoid polluting production code.

2. **Artifact creation**: The mock should actually write files to `MOCK_TRANSPILER_ARTIFACTS_DIR` so artifact metadata tests work.

3. **Deterministic by default**: Without env vars, the mock should produce consistent results (same delay, same output, no failures).

4. **Build in CI**: Add `cargo build --package mock-transpiler` to CI test setup.

5. **Not a production dependency**: The mock is only used for testing. Production agent-queue invokes the real transpiler binary.

---

## Implementation Effort

| Task | Hours |
|------|--------|
| Cargo workspace setup | 0.5h |
| `execute` subcommand | 2h |
| `validate` subcommand | 1h |
| Configurable failure modes | 1.5h |
| Artifact file creation | 0.5h |
| Test helpers | 1h |
| **Total** | **6.5h** |

> This effort is absorbed into Phase 3 (Task 3.2) and Phase 5 (testing).
