# Edge cases and failure modes

Each case states the required behaviour. IDs are referenced from assertions and tests.

## Manifest and signature

| ID | Case | Behaviour |
| --- | --- | --- |
| E-M1 | Manifest present, signature missing | `fail`, `OG-INT-002` |
| E-M2 | Signature present, manifest missing | `fail`, `OG-INT-001` |
| E-M3 | Manifest has a UTF-8 BOM or CRLF line endings | Not canonical. The signature is checked first, so a signed but non-canonical manifest gives `OG-INT-006` |
| E-M4 | Manifest larger than 16 MiB | `fail`, `OG-INT-005`, read without unbounded allocation |
| E-M5 | Manifest `format` is `oracle-guard-manifest/v2` | `fail`, `OG-INT-005` (unknown major) |
| E-M6 | Two keys in the set; the manifest is signed by the rotated-out key, which is no longer listed | `fail`, `OG-INT-003` |
| E-M7 | Manifest has a non-empty `test_config`, but the gate does not declare `config-guard` | `verify` does not judge `test_config`. Its summary notes `test_config not checked by verify`. Gate configuration must declare `config-guard` from L2 |
| E-M8 | `--expect-digest` is malformed (not `sha256:` plus 64 hex characters) | `error`, `OG-INP-007` |
| E-M9 | The manifest lists the same path twice | `fail`, `OG-INT-005` |
| E-M10 | An expected test outside every locked prefix | `fail`, `OG-INT-005` |

## Workspace and file system

| ID | Case | Behaviour |
| --- | --- | --- |
| E-W1 | A locked path's parent directory is a symlink | Treated as non-regular: `OG-INT-010`. Path resolution never leaves the root |
| E-W2 | Case-insensitive file system: `Foo_tests.rs` vs `foo_tests.rs` | Paths are compared byte by byte. A lookup that succeeds only through case folding is detected by comparing the sorted walk names, and reported as `OG-INT-008`. Gate sandboxes are Linux (case-sensitive); this is documented as unsupported elsewhere |
| E-W3 | Non-UTF-8 file name under `tests/locked/` | `fail`, `OG-INT-011`, with the path replaced by `tests/locked/<invalid path>`, which is not used as a location |
| E-W4 | A locked file is unreadable (permissions) | `error`, `OG-INP-001`. The workspace cannot be judged |
| E-W5 | A locked file changes during hashing | Not detected in a single run. The gate sandbox is quiescent. Documented residual (T-09) |
| E-W6 | A very large locked file (for example a 2 GiB snapshot) | Streamed hashing, so memory stays bounded. No size limit on locked files beyond the file system |
| E-W7 | `.git` is a file (worktree) or is absent | `.git` at the root is skipped either way |

## Test list and test report

| ID | Case | Behaviour |
| --- | --- | --- |
| E-T1 | The same test name exists in two binaries | Distinct test IDs. Both are handled independently |
| E-T2 | Workspace with several crates, so the binary IDs are `crate_a`, `crate_b::it` | Supported. Locked prefixes name the binary |
| E-T3 | Report has an expected test with both `<failure>` and `<skipped>` | `OG-INV-003` and `OG-INV-002` both reported |
| E-T4 | Report contains `<system-out>` with megabytes of output | Ignored (skipped without being kept in memory). It counts towards the size limit |
| E-T5 | Report from a nextest run with retries enabled | Any flaky or rerun elements give `OG-INV-006`. Gate profiles should set `retries = 0` |
| E-T6 | Report contains tests from several nextest runs concatenated | Duplicates give `OG-INV-005`. One report per run is expected |
| E-T7 | Report contains holdout tests, but no holdout expectations were passed | They are unexpected only if they match a locked prefix. Holdout binaries and modules must be outside the locked prefixes (`__holdout` module, separate `tests/` files) |
| E-T8 | Test list produced with `--run-ignored all` | Ignored tests are still marked `ignored: true`, giving `OG-LCK-005` |
| E-T9 | Test list from a different checkout | Undetectable by oracle-guard. The oracle stage runs both commands in one sandbox. Residual (T-10) |
| E-T10 | proptest or rstest-generated test names | Treated as ordinary test names. Generated case names must be stable across runs, or the oracle role must lock the parent prefix |
| E-T11 | Test report is empty because compilation failed | No testcases, so `OG-INP-003` (empty `<testsuites>`) and status `error`. The caller's own build failure is also reported by gate-warden's test check |

## Result emission

| ID | Case | Behaviour |
| --- | --- | --- |
| E-R1 | `GATE_RESULT_FILE` points to an existing file | Exit 2, file untouched (`OG-INP-008`) |
| E-R2 | `GATE_RESULT_FILE` directory not writable | Exit 2. gate-warden records `CheckOutputInvalid` |
| E-R3 | SIGTERM or timeout during a run | No partial result is written: the result is written to a temporary file in the same directory and renamed. gate-warden records `CheckTimeout` |
| E-R4 | A path in a finding is not schema-valid | The `location` is dropped and the message keeps a placeholder |
| E-R5 | Both a violation and an input error occur | Status `error`. All findings are listed |

## L2 and L3 (outline)

| ID | Case | Behaviour |
| --- | --- | --- |
| E-C1 | `Cargo.toml` uses `[target.'cfg(unix)'.dev-dependencies]` | The section path quotes the key. Canonical selection is defined at L2 |
| E-C2 | Locked section moved to a workspace-inherited form (`features.workspace = true`) | Hash differs, giving `OG-CFG-004`. The oracle role must lock the workspace root `Cargo.toml` section too |
| E-S1 | Macro-generated code (`macro_rules!` emitting `#[cfg(test)]`) | Detected only in the macro definition's tokens. Proc-macro output is not visible. Residual |
| E-S2 | `#[cfg(any(test, feature = "x"))]` | Counts as `cfg(test)` usage, giving `OG-SNF-001` |
| E-H1 | Holdout placeholder file exists but the `mod __holdout;` declaration was deleted | Holdout tests do not run, giving `OG-INV-001` with holdout visibility. `holdout-inject` also gives `OG-HLD-004` |
| E-H2 | Holdout set larger than the limits | `OG-HLD-001` |
