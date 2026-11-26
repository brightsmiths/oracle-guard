# Behavioural assertions

Each assertion is testable and names the rule it proves. Every assertion has at least one test,
and every faking vector has a fixture ([testing.md](testing.md#fixture-catalogue)). L1
assertions are complete. L2 and L3 assertions are outlines to be refined when those levels
start.

"Valid setup" means:

- a workspace with a manifest produced by `lock`;
- a signature by key `k1`, where `k1` has the `manifest` purpose in the trusted key set;
- the key set lies outside the workspace;
- a test report in which every expected test passed exactly once.

## lock (L1)

**A-LCK-01 Deterministic manifest.**
Given the same workspace, lock spec and test list,
When `lock` runs twice, in any directory walk order and with the test list keys in any order,
Then both manifests are byte-identical and in canonical form,
And stdout holds exactly the manifest digest.

**A-LCK-02 Locked files are hashed by raw bytes.**
Given a lock spec matching `src/a_tests.rs` (CRLF line endings) and `tests/locked/fixtures/x.bin`,
When `lock` runs,
Then `files` lists both and `tests/locked/LOCK.yaml`, each with the SHA-256 of its raw bytes,
And `files` does not list `MANIFEST` or `MANIFEST.sig`.

**A-LCK-03 Expected tests come from the test list, filtered by locked prefixes.**
Given a test list with `billing invoice::tests::a`, `billing invoice::tests::b` and
`billing own::c`, and a locked prefix `{billing, "invoice::tests::"}`,
When `lock` runs,
Then `expected_tests` is exactly `a` and `b`, sorted.

**A-LCK-04 Prefix matching respects segment boundaries.**
Given a locked prefix `{billing, "invoice::tests::"}` and a test `billing invoice::tests_extra::z`,
When `lock` runs,
Then `z` is not expected.

**A-LCK-05 Ignored locked tests are refused.**
Given a test inside a locked prefix that the test list marks as `ignored: true`,
When `lock` runs,
Then it exits 2 with `OG-LCK-005`,
And no manifest file is created or changed.

**A-LCK-06 Empty globs and prefixes are refused.**
Given a lock spec glob that matches no file (or a locked prefix that matches no test),
When `lock` runs,
Then it exits 2 with `OG-LCK-002` (or `OG-LCK-004`),
And no manifest is written.

**A-LCK-07 Symlinks are never locked.**
Given a glob that matches a symlink,
When `lock` runs,
Then it exits 2 with `OG-LCK-003`.

**A-LCK-08 Requirement mapping.**
Given `requirements` entries `{REQ-1, billing, "invoice::"}` and
`{REQ-2, billing, "invoice::tests::a"}`,
When `lock` runs,
Then test `a` carries `req_id: REQ-2` (the longest prefix wins) and test `b` carries `REQ-1`.

**A-LCK-09 lock holds no keys and writes no signature.**
Given any input,
When `lock` runs,
Then `MANIFEST.sig` is not created or modified, and no key input is accepted.

**A-LCK-10 L2 fields are rejected at L1.**
Given a lock spec with a non-empty `test_config` or `sniff_allowlist`,
When an L1 `lock` runs,
Then it exits 2 with `OG-LCK-001`.

## Manifest loading (L1, all manifest-consuming commands)

**A-LOAD-01 Signature verified before parsing.**
Given a manifest whose bytes were changed after signing, including changes that do not alter
the parsed content (an extra comment),
When any check runs,
Then the status is `fail` with `OG-INT-003`,
And no other manifest-content finding is reported.

**A-LOAD-02 Unknown key.**
Given a signature with key ID `k9`, which is not in the trusted key set,
Then the status is `fail` with `OG-INT-003`.

**A-LOAD-03 Key purpose.**
Given `k1` whose purposes are only `holdout-set`,
Then the status is `fail` with `OG-INT-003`.

**A-LOAD-04 Missing signature or manifest.**
Given `MANIFEST.sig` missing (or `MANIFEST` missing),
Then the status is `fail` with `OG-INT-002` (or `OG-INT-001`).

**A-LOAD-05 Algorithm confusion is rejected.**
Given a signature file with `algorithm: "none"` or `"ed448"`,
Then the status is `fail` with `OG-INT-002`.

**A-LOAD-06 Replay detected with a pinned digest.**
Given an older manifest validly signed by `k1`, and `--expect-digest` set to the current
manifest's digest,
Then the status is `fail` with `OG-INT-004`.

**A-LOAD-07 Unpinned run is disclosed.**
Given a valid setup without `--expect-digest`,
When `verify` passes,
Then the summary states that the manifest digest was not pinned.

**A-LOAD-08 Non-canonical manifest.**
Given a manifest with valid content but non-canonical bytes (for example single quotes),
correctly signed,
Then the status is `fail` with `OG-INT-006`.

**A-LOAD-09 YAML features rejected.**
Given a signed manifest that uses an anchor and alias, a tag, a duplicate key or an unknown
field,
Then the status is `fail` with `OG-INT-005`.

**A-LOAD-10 Policy constants.**
Given a signed manifest with `skipped_allowed: true`,
Then the status is `fail` with `OG-INT-005`.

**A-LOAD-11 Path traversal in the manifest.**
Given a signed manifest listing `../outside` or `/etc/passwd` or `tests/locked/MANIFEST`,
Then the status is `fail` with `OG-INT-007`,
And no file outside the workspace is read.

**A-LOAD-12 Trusted key set inside the workspace.**
Given `--trusted-keys` resolving inside `--root` (directly or through a symlink),
Then the status is `error` with `OG-INP-005`, and the exit code is 2.

**A-LOAD-13 Invalid trusted key set.**
Given a key set with a 31-byte public key, a duplicate key ID or an unknown format,
Then the status is `error` with `OG-INP-006`.

## verify (L1)

**A-VER-01 Clean pass.**
Given a valid setup,
When `verify` runs,
Then the status is `pass`, the exit code is 0, there are no findings, and `locked.files` equals
the manifest file count.

**A-VER-02 Edited fixture (faking vector).**
Given one byte changed in `tests/locked/fixtures/invoice_negative.json`,
Then the status is `fail` with exactly one `OG-INT-009` finding located at that path.

**A-VER-03 Edited locked test or snapshot.**
Given a changed locked test file or snapshot file,
Then the status is `fail` with `OG-INT-009` at that path.

**A-VER-04 Deleted locked file.**
Given a listed file that was deleted,
Then the status is `fail` with `OG-INT-008`.

**A-VER-05 Replaced by symlink.**
Given a locked file replaced by a symlink to a file with identical content,
Then the status is `fail` with `OG-INT-010`,
And the symlink is not followed.

**A-VER-06 Extra file in the oracle directory.**
Given a new file `tests/locked/fixtures/extra.json` that is not in the manifest,
Then the status is `fail` with `OG-INT-011`.

**A-VER-07 All violations reported.**
Given three different violations,
Then all three findings are present, sorted deterministically.

## inventory (L1)

**A-INV-01 Clean pass.**
Given a valid setup and a report in which every expected test passed once, plus extra tests
outside the locked prefixes,
Then the status is `pass`, and `tests.passed` equals `tests.expected`.

**A-INV-02 Deleted `mod tests;` (faking vector).**
Given the `#[cfg(test)] #[path = "invoice_tests.rs"] mod tests;` line removed from
`src/billing/invoice.rs`, so the locked tests are absent from the report,
Then the status is `fail` with one `OG-INV-001` per missing expected test, carrying `req_id`
where the manifest has one.

**A-INV-03 `#[ignore]` (faking vector).**
Given `#[ignore]` added to a locked test, which the report shows as skipped or omits,
Then the status is `fail` with `OG-INV-002` or `OG-INV-001` for that test.

**A-INV-04 Excluded test target (faking vector).**
Given an integration test target removed or set to `test = false` in `Cargo.toml`, or excluded
by a nextest `default-filter`,
Then every expected test in that binary gives `OG-INV-001`.

**A-INV-05 Failure.**
Given an expected test with `<failure>` or `<error>`,
Then the status is `fail` with `OG-INV-003`.

**A-INV-06 Extra locked test.**
Given a report test `billing invoice::tests::sneaky`, which is inside a locked prefix but not
expected,
Then the status is `fail` with `OG-INV-004`.

**A-INV-07 Duplicate result.**
Given the same test ID twice in the report,
Then the status is `fail` with `OG-INV-005`.

**A-INV-08 Flaky or retried.**
Given an expected test with `<flakyFailure>` and a final pass,
Then the status is `fail` with `OG-INV-006`.

**A-INV-09 Malformed or hostile report.**
Given a report with a DOCTYPE or entity declaration, non-well-formed XML, an empty
`<testsuites>`, or conflicting `classname` and `testsuite` names,
Then the status is `error` with `OG-INP-003`,
And no entity is expanded and no external resource is read.

**A-INV-10 Oversized report.**
Given a report over the size limit,
Then the status is `error` with `OG-INP-002`.

**A-INV-11 Test names never carry content.**
Given a report whose test name contains whitespace, control characters or more than 512 bytes,
Then any finding about it shows `<invalid test id>` and no part of the raw name.

## Result emission (L1, all checks)

**A-RES-01 Schema conformance.**
Given any check outcome in the test suite,
Then the emitted document validates against the vendored gate-result/v1 schema.

**A-RES-02 Result destination.**
Given `GATE_RESULT_FILE` set to a non-existent path,
Then the result is written there and nothing is written to stdout.
Given the path already exists,
Then the exit code is 2 (`OG-INP-008`) and the file is not changed.

**A-RES-03 Check ID.**
Given `GATE_CHECK_ID=g6.oracle-verify` and no `--check-id`,
Then `check_id` is `g6.oracle-verify`.
Given `--check-id` with an uppercase letter,
Then the exit code is 2 (`OG-INP-007`).

**A-RES-04 Status and exit code agree.**
For every outcome: `pass` comes with exit 0, `fail` with exit 1, `error` with exit 2.

**A-RES-05 Truncation.**
Given 1 500 violations,
Then there are exactly 1 000 findings: 999 sorted violations plus `OG-INP-010` as a notice,
`findings.dropped` is 501, and the status is unchanged.

**A-RES-06 Message caps.**
Given any finding,
Then its message is at most 300 characters and the summary is at most 200 characters.

**A-RES-07 Panic is contained.**
Given an injected internal panic,
Then the status is `error` with `OG-INP-009`, and the exit code is 2.

**A-RES-08 Determinism.**
Given identical inputs,
Then the gate result bytes are identical across runs.

## config-guard (L2, outline)

- **A-CFG-01:** A feature added to `Cargo.toml` `[features]` gives `OG-CFG-004`.
- **A-CFG-02:** A reformatted `[features]` table with the same meaning gives `pass`.
- **A-CFG-03:** A new `.config/nextest.toml` or `build.rs` that is not in the manifest gives
  `OG-CFG-006`.
- **A-CFG-04:** A changed `[package] version`, which is outside the locked sections, gives
  `pass`.

## red-check (L2, outline)

- **A-RED-01:** A new test that passes against `todo!()` stubs gives `OG-RED-001`.
- **A-RED-02:** With `--previous-manifest`, tests already in the previous manifest are not
  judged.
- **A-RED-03:** No new tests gives `pass` with the summary `no new oracle tests`.
- **A-RED-04:** A previous manifest with an invalid signature gives `error` with `OG-INP-011`.

## sniff (L2, outline)

- **A-SNF-01:** An inline `#[cfg(test)] fn helper()` in `src/` gives `OG-SNF-001`. The
  declaration `#[cfg(test)] #[path = "x_tests.rs"] mod tests;` pointing at a locked file gives
  no finding.
- **A-SNF-02:** `if cfg!(test) { … }` gives `OG-SNF-002`.
- **A-SNF-03:** `std::env::var("CI")` gives `OG-SNF-004`.
- **A-SNF-04:** The literal `"rejects_negative_amounts"` in `src/` gives `OG-SNF-005`.
- **A-SNF-05:** An inline suppression comment has no effect. A matching `sniff_allowlist` entry
  suppresses the finding. An unused entry gives an `OG-SNF-009` notice and the status stays
  `pass`.
- **A-SNF-06:** An unparseable `src/` file gives `OG-SNF-008` with status `fail`.

## holdout-inject (L3, outline)

- **A-HLD-01:** A valid holdout set is copied to its targets, and the holdout expectations list
  every holdout test ID with its `req_id`.
- **A-HLD-02:** A holdout target `../src/lib.rs` gives `OG-HLD-002`, and nothing is written.
- **A-HLD-03:** A failing holdout test in `inventory` gives a finding with
  `visibility: holdout`, the fixed message, no location and the `req_id`.
- **A-HLD-04:** A holdout ID equal to an expected test ID gives `OG-HLD-005`.
- **A-HLD-05:** No output of any command contains a holdout file path, test name or content.
