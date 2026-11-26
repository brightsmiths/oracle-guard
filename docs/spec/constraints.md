# Implementation constraints

These rules bind the interface designer, the coder and the reviewers.

## Type system rules

- Use newtypes for every domain value:
  - RepositoryPath, Sha256Digest, ManifestDigest
  - BinaryId, TestName, TestId, RequirementId, KeyId, CheckId, RuleId
- Constructors validate, so invalid values cannot exist.
- `Policy` has exactly one v1 value. Invalid policies cannot be represented.
- `Finding` for holdouts is a separate variant with no location and no test name fields.
- Status is computed from findings. There is no setter.
- Rule IDs form a closed enum generated from, or checked against,
  [rule-catalogue.md](rule-catalogue.md). Message templates belong to the rule, not to the
  call site.
- No stringly typed paths in the core: no `String` or `PathBuf` for repository paths.

## Error handling

- Expected failures are values (`Result<T, E>` with typed, `thiserror`-style enums per area).
  Panics are bugs.
- Workspace violations are **findings** (data), not errors. Input problems are typed errors that
  become `OG-INP` findings at the shell.
- No `unwrap`, `expect` or indexing that can panic on input-derived data. Enforced by clippy:
  `unwrap_used`, `expect_used`, `indexing_slicing` deny in non-test code.
- The top-level handler catches panics and emits `OG-INP-009` (exit 2).

## Boundary rules

- The core does not import infrastructure crates (`clap`, XML, YAML, `syn`, `std::fs`,
  `std::env`, `std::process`, `std::net`).
- Only the command-line adapter reads environment variables, and only `GATE_RESULT_FILE`,
  `GATE_CHECK_ID` and `GATE_WORKSPACE`.
- The binary never spawns processes or opens sockets. A CI check (for example a `cargo-deny`
  ban list, or a grep for `std::process` and `std::net`) enforces this.
- `#![forbid(unsafe_code)]` in every crate.
- No `#[cfg(test)]` in production code except the sibling `mod tests;` declarations. This
  repository follows its own sniff rules.

## Resource limits

| Input | Limit | On excess |
| --- | --- | --- |
| Manifest | 16 MiB | OG-INT-005 |
| Lock spec | 1 MiB | OG-LCK-001 |
| Signature file, trusted key set | 64 KiB | OG-INT-002 / OG-INP-006 |
| Test list | 256 MiB | OG-INP-002 |
| Test report | 256 MiB, XML depth 32 | OG-INP-002 / OG-INP-003 |
| Single test ID part | 512 bytes | Treated as invalid |
| Manifest entries | 100 000 per list | OG-INT-005 |
| Source file for sniff (L2) | 8 MiB | OG-SNF-008 |
| Gate result | 1 000 findings, 1 MiB | Truncation (OG-INP-010) |

## Performance targets

On a gate sandbox with 2 vCPUs and an SSD, measured in benchmarks:

- `verify`: under 2 s for 10 000 locked files and 200 MiB in total. Hashing is streamed, and
  parallel hashing is allowed if the output stays deterministic.
- `inventory`: under 2 s for a 50 000-testcase report.
- `lock`: under 3 s for 10 000 files and a 50 000-test list.
- Peak memory under 256 MiB for all of the above (excluding the OS file cache).
- No algorithm worse than O(n log n) in the input size.

## Testing requirements

- Line coverage ≥ 90% overall and 100% of rule IDs exercised. Every rule has a fixture that
  triggers it.
- Mutation testing (`cargo-mutants`) on the core: no surviving mutants in the manifest loading
  pipeline, inventory evaluation or status derivation.
- Fuzzing for the strict YAML reader, the manifest codec, the JUnit reader and the nextest list
  reader.
- Test doubles only at the ports. The core is tested with plain values.
- Details: [testing.md](testing.md).

## Determinism rules

- Sorted directory walks. `BTreeMap` or sorted `Vec` in every output path. No `HashMap`
  iteration in outputs.
- No timestamps, hostnames, absolute paths or random values in any output.

## Documentation rules

- Specs and comments never reference temporary or ignored folders. Cite the handbook or
  gate-warden repositories instead.
- Each public item references the rule IDs or assertions it implements.
