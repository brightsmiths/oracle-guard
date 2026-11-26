# Assumptions

Each row is an assumption that was challenged, with its resolution. **Decided** means the
product owner confirmed it. **Assumed** means it is the architect's default and can be revisited.

| ID | Assumption | Challenged because | Resolution | Status |
| --- | --- | --- | --- | --- |
| AS-01 | oracle-guard defines its own result format | gate-warden already defines `gate-result/v1` and its feedback rules | Conform to gate-warden's format; define only the mapping ([result-format.md](result-format.md), [ADR-0006](../adr/ADR-0006-gate-result-conformance-and-fail-closed.md)) | **Decided** |
| AS-02 | `lock` takes a hand-written test list | Hand lists drift and can silently omit tests | Input is `cargo nextest list --message-format json`, filtered by `locked_prefixes` ([ADR-0004](../adr/ADR-0004-test-identity-and-lock-input.md)) | **Decided** |
| AS-03 | The manifest may be any YAML | Parser differentials, alias bombs, multiple encodings of one content | Strict YAML: no anchors or aliases, unknown fields rejected, deterministic canonical output ([ADR-0002](../adr/ADR-0002-manifest-format-strict-yaml.md)) | **Decided** |
| AS-04 | red-check judges every expected test | Old tests may have no stubs any more | Only tests added since `--previous-manifest` when given, otherwise all expected tests ([ADR-0007](../adr/ADR-0007-oracle-policy-checks-red-check-and-sniff.md)) | **Decided** |
| AS-05 | Sniff findings can be suppressed inline | The implementer would self-certify | No inline suppressions. Allowlist in the signed manifest only, from L2 ([ADR-0007](../adr/ADR-0007-oracle-policy-checks-red-check-and-sniff.md)) | **Decided** |
| AS-06 | oracle-guard can find the trusted public key itself | Any key in the workspace is forgeable, and the tool has no network access | Explicit `--trusted-keys`, outside the workspace, supplied by the harness | Assumed |
| AS-07 | The signature alone prevents manifest substitution | An older valid manifest can be replayed | `--expect-digest` pinned by the gate caller. Unpinned runs are disclosed in the summary | Assumed. The gate definition must pin the digest (security T-03) |
| AS-08 | Only cargo-nextest JUnit and list output must be supported | `cargo test` has no stable JUnit output | nextest only. Its formats are pinned per oracle-guard release | Assumed |
| AS-09 | Test IDs from JUnit can be mapped to locked files | JUnit has no file paths, and `#[path]` breaks name inference | Locked prefixes in the manifest declare the locked test namespace | Assumed |
| AS-10 | The gate checkout is byte-identical to the oracle-stage checkout | `autocrlf` or `.gitattributes` filters change bytes | Required of gate-warden and sandbox-runner. A hash mismatch shows any deviation | Assumed ([operations.md](operations.md)) |
| AS-11 | Gate sandboxes are Linux on case-sensitive file systems | Case folding can alias paths | Yes. Other platforms are best effort | Assumed |
| AS-12 | Holdouts are fetched by the gate caller | Fetching needs credentials and network, which ADR-0001 forbids | `holdout-inject` reads a local directory outside the workspace | Assumed |
| AS-13 | The holdout set is signed | Otherwise anyone who can write the holdout directory can change hidden tests | Signed with a key whose purposes include `holdout-set`. Format to be fixed at L3 | Assumed |
| AS-14 | `holdout-inject` emits a gate result like the checks | gate-warden's documents define checks, but not preparation steps that mutate the workspace before the test run | Emit gate-result/v1 for now. Revisit when gate-warden L3 defines preparation steps | **Open, L3**. Not blocking L1 |
| AS-15 | The signing message format is acceptable to sandbox-runner and the key service | Some key services sign only digests (pre-hashed Ed25519ph) | Pure Ed25519 over the domain-prefixed bytes. If the key service needs Ed25519ph, a `/v2` signature format is added | Assumed, to be confirmed with sandbox-runner |
| AS-16 | Policy values can vary per manifest | The brief lists them as fixed | Constants in v1. Changing them needs v2 | Assumed |
| AS-17 | Performance matters | Gate latency adds up across checks | Targets in [constraints.md](constraints.md#performance-targets): seconds for 10k files and 50k tests | Assumed |
| AS-18 | Implementer tests never share binaries or modules with locked prefixes | Shared modules would produce `OG-INV-004` false positives | Implementer tests live in `tests/own/` (separate binaries). Locked unit tests sit in dedicated `tests` modules | Assumed, enforced by OG-INV-004 |
