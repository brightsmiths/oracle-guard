# Rule catalogue

Every finding oracle-guard emits, and every `lock` diagnostic, has a rule ID from this page.

- **Format:** `OG-<AREA>-<NNN>`.
- IDs are never reused or renumbered. A retired rule is marked *retired*.
- Messages are fixed templates. Placeholders are only `{path}` (a repository path), `{test}` (a
  test ID in display form), `{rule}`, `{n}` and `{digest}`
  ([result-format.md](result-format.md#finding-mapping)).
- **Status:** **F** gives `fail`, **E** gives `error`, **N** is a notice and does not change the
  status, and **L** is a `lock` diagnostic (exit 2, no gate result).
- **Fixture** names a fixture in [testing.md](testing.md#fixture-catalogue).

## Input errors: `OG-INP` (all commands)

| ID | Lvl | Status | Message template | Trigger |
| --- | --- | --- | --- | --- |
| OG-INP-001 | L1 | E | `input not readable: {name}` | A required input file is missing or unreadable. `{name}` is the flag name, never the path content |
| OG-INP-002 | L1 | E | `input exceeds size limit: {name}` | Over the limit for that input ([constraints.md](constraints.md#resource-limits)) |
| OG-INP-003 | L1 | E | `test report is not a valid nextest JUnit report` | XML not well formed, DTD present, unexpected root, empty, or conflicting binary IDs |
| OG-INP-004 | L1 | E | `test list is not valid nextest list output` | Not JSON, or missing `rust-suites` |
| OG-INP-005 | L1 | E | `trusted input must be outside the workspace: {name}` | Resolves inside `--root` |
| OG-INP-006 | L1 | E | `trusted key set is invalid` | Schema, version, duplicate key ID, bad key bytes |
| OG-INP-007 | L1 | E | `invalid arguments` | Unknown flag, missing flag, bad `--check-id`, bad `--expect-digest` syntax |
| OG-INP-008 | L1 | E | *(no result can be written)* | Result destination exists or is not writable. Exit 2 only |
| OG-INP-009 | L1 | E | `internal error` | A caught panic or broken invariant |
| OG-INP-010 | L1 | N | `{n} further findings omitted` | Truncation at 1 000 findings |
| OG-INP-011 | L2 | E | `previous manifest is invalid` | `red-check --previous-manifest` fails loading |

## Lock diagnostics: `OG-LCK` (`lock` only)

| ID | Lvl | Status | Message template | Trigger |
| --- | --- | --- | --- | --- |
| OG-LCK-001 | L1 | L | `lock spec is invalid` | Missing, not strict YAML, unknown or unsupported version, unsupported language, L2 fields used in L1 |
| OG-LCK-002 | L1 | L | `file pattern matched no files: {pattern}` | Glob with zero matches |
| OG-LCK-003 | L1 | L | `locked path is not a regular file or not a valid repository path: {path}` | Symlink, directory, device, or invalid path |
| OG-LCK-004 | L1 | L | `locked prefix matched no tests: {binary} {prefix}` | Prefix selects nothing in the test list |
| OG-LCK-005 | L1 | L | `locked test is ignored: {test}` | Selected test has `ignored: true` |
| OG-LCK-006 | L1 | L | `locked test excluded by filter: {test}` | Selected test does not match the filter (the list was run with a filter) |
| OG-LCK-007 | L1 | L | `requirement mapping is ambiguous or unused: {req_id}` | Tie between `req_id` values, or an entry with no match |
| OG-LCK-008 | L1 | L | `no expected tests` | Empty selection |

## Integrity: `OG-INT`

`OG-INT-001` to `007` are emitted by **every** command that consumes a manifest. `OG-INT-008`
to `011` are emitted by `verify` only.

| ID | Lvl | Status | Message template | Trigger | Fixture |
| --- | --- | --- | --- | --- | --- |
| OG-INT-001 | L1 | F | `manifest is missing` | No `tests/locked/MANIFEST` | `int-missing-manifest` |
| OG-INT-002 | L1 | F | `manifest signature is missing or malformed` | No `.sig`, bad schema, bad base64, wrong length, algorithm not `ed25519` | `int-bad-sig-format` |
| OG-INT-003 | L1 | F | `manifest signature does not verify` | Unknown key ID, key lacks the `manifest` purpose, or the signature is invalid | `int-tampered-manifest`, `int-unknown-key` |
| OG-INT-004 | L1 | F | `manifest digest does not match the pinned digest {digest}` | `--expect-digest` mismatch (replay or downgrade) | `int-replayed-manifest` |
| OG-INT-005 | L1 | F | `manifest is invalid` | Signed, but not strict YAML, schema violation, unknown version, policy not the v1 constants, or an expected test outside every locked prefix | `int-bad-policy` |
| OG-INT-006 | L1 | F | `manifest is not in canonical form` | Re-serialisation differs from the bytes | `int-noncanonical` |
| OG-INT-007 | L1 | F | `manifest contains an invalid repository path` | Path fails the repository path rules, or lists `MANIFEST` or `MANIFEST.sig` | `int-path-traversal` |
| OG-INT-008 | L1 | F | `locked file is missing: {path}` | Listed file absent | `int-deleted-locked-file` |
| OG-INT-009 | L1 | F | `locked file content does not match the manifest hash: {path}` | SHA-256 differs | `int-edited-fixture`, `int-edited-test`, `int-edited-snapshot` |
| OG-INT-010 | L1 | F | `locked path is not a regular file: {path}` | Now a symlink, directory or other non-regular file | `int-symlinked-locked-file` |
| OG-INT-011 | L1 | F | `file under tests/locked is not in the manifest: {path}` | Unlisted file in the oracle-owned directory | `int-extra-locked-dir-file` |

## Inventory: `OG-INV`

Holdout tests (L3) use the same rules with `visibility: holdout` and the fixed message
`holdout test did not pass`.

| ID | Lvl | Status | Message template | Trigger | Fixtures (faking vector) |
| --- | --- | --- | --- | --- | --- |
| OG-INV-001 | L1 | F | `expected locked test did not run: {test}` | Expected test absent from the report | `inv-deleted-mod-decl` (deleted `mod tests;`), `inv-excluded-test-target` (`test = false` or `[[test]]` removed), `inv-nextest-filter` (default-filter exclusion), `inv-renamed-test`, `inv-cfg-gated-mod` |
| OG-INV-002 | L1 | F | `expected locked test was skipped: {test}` | `<skipped>` | `inv-ignore-attr` (`#[ignore]` with `--run-ignored` absent appears as skipped or absent: either way it is caught by 001 or 002) |
| OG-INV-003 | L1 | F | `expected locked test failed: {test}` | `<failure>` or `<error>` | `inv-failing-test` |
| OG-INV-004 | L1 | F | `unexpected test in locked scope: {test}` | Report test matches a locked prefix but is not expected | `inv-extra-locked-test` |
| OG-INV-005 | L1 | F | `test reported more than once: {test}` | Duplicate test ID in the report | `inv-duplicate-testcase` |
| OG-INV-006 | L1 | F | `expected locked test was flaky or retried: {test}` | Flaky or rerun elements present | `inv-flaky-retry` |

## Config guard: `OG-CFG` (L2, reserved)

| ID | Lvl | Status | Message template | Trigger |
| --- | --- | --- | --- | --- |
| OG-CFG-001 | L2 | F | `test config file does not match the manifest hash: {path}` | Whole-file entry differs |
| OG-CFG-002 | L2 | F | `test config file is missing: {path}` | Listed file absent |
| OG-CFG-003 | L2 | F | `test config section is missing: {path} [{section}]` | Section absent |
| OG-CFG-004 | L2 | F | `test config section does not match the manifest hash: {path} [{section}]` | Canonical section hash differs (for example a new feature or dev-dependency) |
| OG-CFG-005 | L2 | F | `test config file cannot be parsed: {path}` | Invalid TOML where a section is required |
| OG-CFG-006 | L2 | F | `test-affecting config file is not in the manifest: {path}` | For example a new `build.rs`, `.config/nextest.toml` or `.cargo/config.toml` |

## Red-check: `OG-RED` (L2, reserved)

| ID | Lvl | Status | Message template | Trigger |
| --- | --- | --- | --- | --- |
| OG-RED-001 | L2 | F | `oracle test passed against stubs: {test}` | The test asserts nothing that the stubs violate |
| OG-RED-002 | L2 | F | `oracle test did not run against stubs: {test}` | Absent from the stub run |
| OG-RED-003 | L2 | F | `oracle test was skipped against stubs: {test}` | `<skipped>` |

## Sniff: `OG-SNF` (L2, reserved)

The rules apply to non-locked `.rs` files under each crate's `src/`. Every finding has a
location with a line and column.

| ID | Lvl | Status | Message template | Detects |
| --- | --- | --- | --- | --- |
| OG-SNF-001 | L2 | F | `cfg(test) outside a locked test module declaration` | `#[cfg(test)]` or `cfg_attr(test, …)` on anything except a body-less `mod x;` declaration whose resolved file is a locked file |
| OG-SNF-002 | L2 | F | `cfg!(test) in production code` | The `cfg!(test)` macro, including nested in `any`/`all`/`not` |
| OG-SNF-003 | L2 | F | `test-only feature used in production code: {feature}` | `cfg(feature = "…")` naming a feature that is only enabled by dev-dependencies or test targets, or whose name matches `test`, `testing`, `mock`, `fake` or `stub` (exact list fixed at L2) |
| OG-SNF-004 | L2 | F | `test or CI environment detection in production code` | `env::var` / `var_os` / `option_env!` / `env!` of `CI`, `GITHUB_ACTIONS`, `NEXTEST*`, `RUST_TEST_*`, `CARGO_TARGET_TMPDIR`, `GATE_*` (exact list fixed at L2) |
| OG-SNF-005 | L2 | F | `production code references a locked test identity` | A string literal or path equal to an expected test name, its last segment (if at least 8 characters), or a locked module path |
| OG-SNF-006 | L2 | F | `production code inspects the test harness` | `thread::current().name()`, `std::env::args()` compared with test-harness flags, `#[cfg(debug_assertions)]` branches *(debug_assertions: decided at L2)* |
| OG-SNF-007 | L2 | F | *(reserved)* | Reserved for build-script detection of test builds (`build.rs` reading `PROFILE`) |
| OG-SNF-008 | L2 | F | `production source file cannot be parsed: {path}` | Parse failure. Fails closed, so unparseable code cannot hide behaviour |
| OG-SNF-009 | L2 | N | `sniff allowlist entry is unused: {rule} {path}` | Stale allowlist entry |

## Holdout injection: `OG-HLD` (L3, reserved)

Every `OG-HLD` finding has `visibility: holdout`, no location, and the fixed message
`holdout injection failed` (or the `req_id` where one exists).

| ID | Lvl | Status | Trigger |
| --- | --- | --- | --- |
| OG-HLD-001 | L3 | E | The holdout set signature is missing or invalid, or the set is malformed |
| OG-HLD-002 | L3 | E | A holdout target path is invalid or escapes `tests/` |
| OG-HLD-003 | L3 | F | A target file already exists in the workspace and is not the holdout placeholder |
| OG-HLD-004 | L3 | F | The placeholder declaration `#[cfg(test)] mod __holdout;` or its locked file is missing |
| OG-HLD-005 | L3 | E | A holdout test ID collides with an expected test or another holdout test |
