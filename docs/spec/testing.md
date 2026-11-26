# Testing strategy

Targets and tool rules are in [constraints.md](constraints.md#testing-requirements). This page
defines what is tested and how.

## Layers

| Layer | Scope | Doubles |
| --- | --- | --- |
| Unit (core) | Newtype validation, prefix matching, the canonical codec, the loading pipeline steps, evaluators, result building, truncation | None. Plain values |
| Property | Codec round-trip (`parse(serialise(m)) == m`, and `serialise` is canonical); lock determinism under permuted input order; status derivation is monotonic (adding an error finding never yields `pass`); truncation keeps the status | proptest |
| Adapter | JUnit reader, nextest list reader, strict YAML reader, Ed25519 verifier, workspace reader (symlinks, non-UTF-8, permissions) | Temporary directories |
| End-to-end (CLI) | Run the built binary against each fixture repository and compare the gate result JSON, the exit code and stderr rule IDs with golden files | None |
| Fuzz | Strict YAML reader, manifest codec, signature file, key set, JUnit reader, nextest list reader. No panics; bounded memory | cargo-fuzz |
| Mutation | Core crates | cargo-mutants |

## Fixture catalogue

Fixtures live under the repository's own `tests/fixtures/`. Each fixture is a miniature
workspace containing:

- a manifest, a signature, a lock spec and locked files;
- a pre-generated nextest test list and/or JUnit report;
- the expected gate result (golden JSON) and exit code.

Fixtures are signed with a **test-only key** whose private key is checked in under
`tests/fixtures/keys/`. It is clearly marked as never trusted anywhere else. The fixtures'
trusted key sets are stored outside each fixture workspace, to honour `OG-INP-005`.

The real Cargo projects used to generate the test list and JUnit inputs are kept under
`tests/fixtures/projects/`. A regeneration script records the cargo-nextest version used.
Generated inputs are committed, so tests do not need nextest at run time.

### Faking vectors (required by the brief)

| Fixture | Faking vector | Expected |
| --- | --- | --- |
| `inv-deleted-mod-decl` | `#[cfg(test)] #[path = "invoice_tests.rs"] mod tests;` removed | `fail`, OG-INV-001 |
| `inv-ignore-attr` | `#[ignore]` added to a locked test | `fail`, OG-INV-002 or OG-INV-001 (both report shapes covered) |
| `inv-excluded-test-target` | Integration test target removed / `test = false` | `fail`, OG-INV-001 |
| `int-edited-fixture` | One byte changed in a locked fixture | `fail`, OG-INT-009 |

### Further L1 fixtures

| Fixture | Vector / case | Expected |
| --- | --- | --- |
| `inv-nextest-filter` | `.config/nextest.toml` `default-filter` excludes locked tests | `fail`, OG-INV-001 |
| `inv-cfg-gated-mod` | `mod tests;` gated behind `cfg(feature = "x")` that is not enabled | `fail`, OG-INV-001 |
| `inv-renamed-test` | Locked test renamed | `fail`, OG-INV-001 |
| `inv-extra-locked-test` | Extra test in a locked module | `fail`, OG-INV-004 |
| `inv-failing-test` | Locked test fails | `fail`, OG-INV-003 |
| `inv-duplicate-testcase` | Same test ID twice | `fail`, OG-INV-005 |
| `inv-flaky-retry` | Retried test | `fail`, OG-INV-006 |
| `inv-hostile-xml` | DOCTYPE with entities | `error`, OG-INP-003 |
| `int-edited-test` / `int-edited-snapshot` | Locked test or snapshot changed | `fail`, OG-INT-009 |
| `int-deleted-locked-file` | Locked file deleted | `fail`, OG-INT-008 |
| `int-symlinked-locked-file` | Locked file replaced by a symlink | `fail`, OG-INT-010 |
| `int-extra-locked-dir-file` | Unlisted file in `tests/locked/` | `fail`, OG-INT-011 |
| `int-tampered-manifest` | Manifest edited after signing | `fail`, OG-INT-003 |
| `int-unknown-key` | Signed by an untrusted key | `fail`, OG-INT-003 |
| `int-bad-sig-format` | `algorithm: none`, bad base64 | `fail`, OG-INT-002 |
| `int-replayed-manifest` | Older signed manifest with `--expect-digest` set to the newer one | `fail`, OG-INT-004 |
| `int-noncanonical` | Signed non-canonical manifest | `fail`, OG-INT-006 |
| `int-bad-policy` | Signed manifest with `skipped_allowed: true` | `fail`, OG-INT-005 |
| `int-path-traversal` | Signed manifest listing `../x` | `fail`, OG-INT-007 |
| `int-missing-manifest` | No manifest | `fail`, OG-INT-001 |
| `inp-keys-in-workspace` | `--trusted-keys` inside the root | `error`, OG-INP-005 |
| `lck-*` | One per OG-LCK rule | exit 2 with that rule |
| `clean-*` | Clean workspaces (single crate, multi-crate workspace, integration-only) | `pass` |

L2 and L3 fixtures follow the same pattern, one per `OG-CFG`, `OG-RED`, `OG-SNF` and `OG-HLD`
rule, and are added at those levels.

## Gate result conformance

- The gate-warden schema `gate-result/v1.schema.json` is vendored under `tests/schemas/`, with
  a header recording the gate-warden commit it came from.
- Every golden result and every result produced in end-to-end tests is validated against it
  with a JSON Schema 2020-12 validator.
- A test asserts that oracle-guard emits no field missing from the vendored schema
  (`additionalProperties: false` covers this).

## Cross-checks

- **Rule coverage:** a test parses [rule-catalogue.md](rule-catalogue.md) and fails if any rule
  ID lacks a fixture, or if the code emits an ID that is not in the catalogue.
- **Signing contract:** a test signs `"oracle-guard/manifest-signature/v1\n" || bytes` with the
  test key using an independent implementation (for example `openssl pkeyutl` output stored as a
  fixture). This shows that sandbox-runner's signing recipe interoperates.

## Not tested here

Running nextest live, GitHub interaction, key-service signing and sandbox permissions all belong
to other services.
