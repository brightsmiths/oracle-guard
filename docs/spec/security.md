# Security

oracle-guard is a **security-boundary component**. It is treated as T2 or higher whoever builds
it:

- every PR gets a human review;
- gate configuration is read from the default branch.

Its whole purpose is to stop the build agent from weakening the oracle, so the main attacker is
that agent.

## Assets

1. The **verdict integrity** of every oracle-guard check that gate-warden runs.
2. **Holdout confidentiality**: names, files and content (L3).
3. The **oracle role's authority**: only manifests signed by a trusted key are accepted.
4. **Availability** of the gate. A hostile input must not hang or crash the gate in a way that
   counts as passing.

## Actors and trust

| Actor | Trust |
| --- | --- |
| Build agent (zone C) | Hostile. Controls `src/`, `tests/own/` and, through the code under test, everything the test binaries do at gate time |
| Workspace content | Untrusted. That includes MANIFEST, MANIFEST.sig, test reports (influenced by test code) and source |
| Oracle role (zone B) | Trusted for oracle content. Its key is never available to oracle-guard |
| Harness and default-branch trees | Trusted, human-owned (`GATE_HARNESS_DIR`, `GATE_DEFAULT_BRANCH_DIR`) |
| gate-warden | Trusted caller. Provides a clean sandbox and the trusted directories |

## Threats (STRIDE) and mitigations

| ID | Threat | STRIDE | Mitigation | Rules / ADR | Residual |
| --- | --- | --- | --- | --- | --- |
| T-01 | Agent edits a locked test, fixture or snapshot to match its code | Tampering | SHA-256 of every locked file in a signed manifest. agent-gate read-only mounts as defence in depth | OG-INT-009, ADR-0005 | None if `verify` runs |
| T-02 | Agent edits the MANIFEST to match its edits | Tampering | Ed25519 signature over the domain-prefixed bytes. Key held by the key service, never in the sandbox | OG-INT-003, ADR-0003 | Compromise of the oracle key or sandbox-runner |
| T-03 | Agent restores an older, validly signed manifest (fewer tests) | Tampering / replay | `--expect-digest` pinned by the gate caller from a trusted record | OG-INT-004 | **High if the gate does not pin the digest.** The summary discloses unpinned runs. Gate definitions must pin it |
| T-04 | Agent substitutes the trusted key set via the workspace | Spoofing | Trusted inputs must resolve outside `--root`. Key set paths come from the gate definition | OG-INP-005 | Misconfigured gate definition |
| T-05 | Agent makes locked tests silently not run: deleted `mod tests;`, `#[ignore]`, `test = false`, a nextest filter, a `cfg` gate on the module | Tampering | Inventory: every expected test must appear exactly once and pass. Test IDs are tied to locked prefixes | OG-INV-001…006 | None for the listed vectors |
| T-06 | Agent adds a locked-scope test or replaces a test body via a new file in a locked module | Tampering | Unexpected locked test detection. Unlisted file detection under `tests/locked/` | OG-INV-004, OG-INT-011 | New files elsewhere in `src/` that are mounted under a locked module path are caught by OG-INV-004 only if they add test IDs |
| T-07 | Test code (run at gate time) overwrites the JUnit report or the gate result file | Tampering | Out of oracle-guard's reach. gate-warden must write the report outside the paths the tests can write, and the result file is create-new. oracle-guard validates the report structure strictly | OG-INP-003 | **Owned by gate-warden** (sandbox layout) |
| T-08 | Production code detects the test environment (`cfg!(test)`, `CI` variables, thread names) | Tampering / elevation | `sniff` rules (L2). Holdouts run elsewhere (L3) | OG-SNF-001…006 | Proc-macro and runtime-obfuscated detection (see E-S1) |
| T-09 | Time-of-check/time-of-use: a file changes between `verify` and the test run | Tampering | gate-warden runs checks in a quiescent clean sandbox. Compiled-in test sources cannot be changed by test code, but files read at runtime (fixtures) can | — | Fixtures changed **by test code during the run**. Mitigated by read-only mounts at gate time (gate-warden / agent-gate) |
| T-10 | Oracle stage produces a test list from a different tree than the locked files | Tampering | `lock` runs in one sandbox. The manifest is reviewed with the oracle | — | Requires a compromised oracle stage |
| T-11 | Hostile XML (XXE, billion laughs, deep nesting) | DoS / information disclosure | DTD rejected, no entity expansion, depth and size limits, streaming parser | OG-INP-002/003 | — |
| T-12 | Hostile YAML (alias bombs, tags, duplicate keys, parser differentials) | DoS / tampering | Signature checked before parsing. Strict subset. Canonical round-trip | OG-INT-005/006, ADR-0002 | The signature file and key set are parsed before verification; they are tiny and size-capped |
| T-13 | Path traversal through manifest paths, symlinks or holdout targets | Tampering / information disclosure | Repository path rules. `lstat` without following symlinks. Root containment | OG-INT-007/010, OG-HLD-002 | — |
| T-14 | Holdout leakage through findings, summaries, metrics or stderr | Information disclosure | Fixed holdout messages, no location or name, counts only. gate-warden also reduces holdout findings to `req_id`. stderr never names holdouts | A-HLD-05 | Timing and count side channels (coarse) |
| T-15 | Finding messages carry test source, or are used for prompt injection into later agents | Information disclosure / injection | Template-only messages with validated placeholders. gate-warden escapes and caps them | ADR-0006 | Test names chosen by the oracle role appear verbatim (trusted author) |
| T-16 | Agent treats its own local oracle-guard run as authoritative ("tests verified") | Repudiation / spoofing | Only gate-warden's run counts. oracle-guard output has no signature and no authority | ADR-0001 | Process risk only |
| T-17 | Crash or hang that turns into a pass | DoS | Panics are caught and reported as `error`. Bounded inputs. Linear algorithms. gate-warden timeouts give `error` | OG-INP-009 | — |
| T-18 | Algorithm or key confusion (`alg: none`, wrong-purpose key, small-order key) | Spoofing | Single algorithm, purpose check, `verify_strict`, weak-key rejection | OG-INT-002/003, OG-INP-006 | — |
| T-19 | Allowlist or suppression used to hide test-only code | Elevation | No inline suppressions. Allowlist only in the signed manifest | ADR-0007 | Oracle role mistakes |
| T-20 | Supply chain: a compromised dependency in oracle-guard | Tampering | Minimal dependencies, `cargo-deny`, pinned lockfile, binary pinned by digest in the harness, reproducible build | [operations.md](operations.md) | — |

## Security rules for implementation

1. Never follow symlinks in the workspace. Use `lstat` semantics and root-contained path
   resolution.
2. Verify the manifest signature before parsing the manifest.
3. Never interpolate input content into messages, except validated test IDs and repository
   paths.
4. All reads are size-bounded. Recursion depth is bounded. No unbounded allocation driven by
   input.
5. No `unsafe` code, no network, no subprocesses, no dynamic loading.
6. Panics are caught at the top level and turned into `error`. Library code returns `Result`.
7. Constant-time comparison is **not** required for digests, because they are not secrets.
   Signatures are verified by the library.
8. Holdout data (L3) is never written to stderr, logs or any file other than the injected test
   files and the holdout expectations outside the workspace.

## Out of scope (owned elsewhere)

- File permissions in agent sandboxes (agent-gate mount plan).
- Signing and key custody (sandbox-runner, the key service, chain-chronicler).
- Placing reports and results where test code cannot write them (gate-warden sandbox layout).
- Publishing, redaction at publish time, and escaping (gate-warden feedback rules).
