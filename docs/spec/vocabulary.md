# Vocabulary

Every spec file, ADR, type name and finding message uses these terms with these meanings. When
code needs a new domain term, add it here first.

Notation: **Identifier** is how the concept is identified or written. **Lifespan** is how long it
exists.

## Roles and actors

### Oracle role

The identity (trust zone B) that writes locked tests and owns the manifest. It signs the
manifest through sandbox-runner and the key service. oracle-guard never holds its key.

### Build agent

An agent in trust zone C that writes the implementation. It may run oracle-guard for feedback.
Its results are never authoritative.

### Gate caller

The process that runs an oracle-guard check and whose result counts: gate-warden, in a clean
sandbox. Humans stand in for gate-warden at M1.

## Locked oracle

### Locked file

A file whose content is part of the oracle: a locked test file, a fixture, a snapshot or the
lock spec.

- **Identifier:** repository-relative path (see [Repository path](#repository-path)).
- **Constraints:** a regular file (never a symlink, directory or device). Listed in the manifest
  with its SHA-256.
- **Lifespan:** from `lock` until the next `lock` by the oracle role.

### Locked test

A test function inside a locked file. For Rust, a test in a sibling file mounted with
`#[cfg(test)] #[path = "foo_tests.rs"] mod tests;`, or a test in a public-API integration test
file under `tests/`.

### Lock spec

The oracle role's input to `lock`, at `tests/locked/LOCK.yaml`. It holds the file globs, the
[locked prefixes](#locked-prefix) and the optional requirement mapping. It is itself a locked
file. See [manifest-schema.md](manifest-schema.md#lock-spec-testslockedlockyaml).

### Manifest

The signed statement of what the oracle is, at `tests/locked/MANIFEST`. It contains the locked
files, test config entries, locked prefixes, expected tests, the sniff allowlist and the
policy.

- **Identifier:** its [manifest digest](#manifest-digest).
- **Format:** `oracle-guard-manifest/v1`, strict canonical YAML
  ([manifest-schema.md](manifest-schema.md)).
- **Constraints:** byte-for-byte canonical. Signed by the oracle role. Written only by `lock`.
- **Lifespan:** one locked oracle version. A new `lock` produces a new manifest and digest.

### Manifest digest

`sha256:` followed by 64 lowercase hex characters: the SHA-256 of the exact manifest bytes. It
is used to pin a manifest against replay.

### Manifest signature

The detached signature at `tests/locked/MANIFEST.sig` (format `oracle-guard-sig/v1`).

- **Contents:** algorithm (`ed25519`), key ID and the base64 signature over the
  [signing message](manifest-schema.md#signing-message).

### Trusted key set

A file of public keys that are allowed to sign manifests (and, from L3, holdout sets). Each
entry is a key ID, an algorithm and a public key. It is a [trusted input](#trusted-input).

### Key ID

A string that names one key in the trusted key set: 1–128 characters from `[A-Za-z0-9._:-]`.

### Policy

The manifest's fixed rules: `all_expected_must_run: true`, `skipped_allowed: false`,
`extra_locked_tests_allowed: false`. In v1 these values are constants. A manifest with any other
value is invalid.

## Tests and results

### Binary ID

cargo-nextest's identifier for a test binary, for example `billing` (library unit tests) or
`billing::invoice_api` (the integration test `tests/invoice_api.rs`).

### Test name

The test's path inside its binary, as nextest reports it, for example
`invoice::tests::rejects_negative_amounts`.

### Test ID

The pair (binary ID, test name). It uniquely identifies a test in a workspace.

- **Display form:** `<binary-id> <test-name>`, separated by one space.
- **Constraints:** both parts are non-empty, at most 512 bytes, UTF-8, and contain no whitespace
  or control characters.

### Locked prefix

A test selector `{binary, name_prefix}` that declares which test IDs belong to the locked oracle.
A test ID matches when:

- the binary IDs are equal, and
- `name_prefix` is empty, or equals the test name, or ends in `::` and is a prefix of the test
  name.

Every test that matches a locked prefix is a **locked test** for the inventory.

### Expected test

A test ID listed in the manifest. It must run and pass. It may carry a requirement ID.

### Requirement ID

The upstream requirement a test verifies, for example `REQ-017`. Pattern:
`^[A-Za-z][A-Za-z0-9_-]{0,63}$`, the gate result `req_id` rule. It is the only thing published
about a failing holdout test.

### Test list

The JSON output of `cargo nextest list --message-format json`. It is the input to `lock`.

### Test report

The JUnit XML report written by cargo-nextest for one test run. It is the input to `inventory`
and `red-check`.

### Test outcome

The result of one test ID in a test report:

| Outcome | Meaning |
| --- | --- |
| `passed` | Ran once and succeeded |
| `failed` | Has a `<failure>` or `<error>` element |
| `skipped` | Has a `<skipped>` element |
| `flaky` | Has `<flakyFailure>`, `<flakyError>`, `<rerunFailure>` or `<rerunError>` elements, whatever the final state |
| `not-run` | Absent from the report |

### Inventory

The comparison of expected tests (plus holdout tests from L3) with the test report, under the
policy.

### Stub run *(L2)*

A test run of the locked tests against the implementation stubs (for Rust, bodies of `todo!()`
or `unimplemented!()`), done by the caller before implementation starts.

### Red-check target set *(L2)*

The expected tests that `red-check` judges: the tests added since `--previous-manifest` when
that flag is given, otherwise all expected tests.

## Configuration and production code

### Test config entry *(L2)*

A manifest entry `{path, section?, sha256}` for a file that affects which tests run or how they
behave. With no `section`, the whole file is hashed. With a `section`, only that part is hashed.

### Config section *(L2)*

A dotted TOML table path inside a config file, for example `features` or `profile.test`. It is
hashed in a canonical form ([ADR-0005](../adr/ADR-0005-hashing.md)).

### Sniff rule *(L2)*

A static rule that finds test-only behaviour in production code. Rule IDs are `OG-SNF-NNN`
([rule-catalogue.md](rule-catalogue.md#sniff-og-snf-l2-reserved)).

### Sniff allowlist entry *(L2)*

A manifest entry `{rule, path, reason}` that exempts one rule in one file. Allowlist entries are
the only way to make an exception: there are no inline suppressions.

## Holdouts (L3)

### Holdout set

A signed bundle of hidden tests from the private holdouts repository. It contains the test
files, their target paths, and the holdout test IDs, each with a requirement ID. The gate caller
fetches it; oracle-guard only reads a local directory.

### Holdout test

A test ID from the holdout set. Inventory treats it as expected. Any finding about it has
`visibility: holdout` and carries only its requirement ID.

### Holdout placeholder

The locked file mounted with `#[cfg(test)] mod __holdout;`. `holdout-inject` replaces its
content with holdout tests.

## Checks and results

### Check

One oracle-guard command run that judges a workspace and writes a gate result: `verify`,
`inventory`, `config-guard`, `red-check` or `sniff`. `lock` and `holdout-inject` are not checks.

### Check ID

The `check_id` in the gate result. It must equal `GATE_CHECK_ID` when gate-warden runs the
check. Pattern: `^[a-z0-9][a-z0-9._-]{0,63}$`. The default is `oracle-guard.<command>`.

### Gate result

One JSON document in gate-warden's `gate-result/v1` format
([result-format.md](result-format.md)).

### Status

The gate result status:

- **`pass`:** the workspace is acceptable.
- **`fail`:** the workspace content is unacceptable.
- **`error`:** oracle-guard could not judge.

There is no skip.

### Finding

One reason for a status. It has a [rule ID](#rule-id), severity, a coarse message, an optional
location, an optional requirement ID and a visibility.

### Rule ID

A stable identifier `OG-<AREA>-<NNN>`, defined in [rule-catalogue.md](rule-catalogue.md). Rule
IDs are never reused.

### Visibility

`visible` or `holdout`, as in gate-result/v1. Holdout findings carry only a requirement ID and a
fixed message.

## Inputs and locations

### Workspace

The checkout under judgement (`--root`, or `GATE_WORKSPACE`). Its content is untrusted.

### Trusted input

An input that decides how the workspace is judged:

- the trusted key set;
- the expected manifest digest;
- the previous manifest;
- the holdout set.

Trusted inputs must come from outside the workspace, such as `GATE_HARNESS_DIR` or
`GATE_DEFAULT_BRANCH_DIR`. oracle-guard rejects a trusted input path that resolves inside the
workspace.

### Repository path

A path relative to the workspace root:

- forward slashes;
- no leading `/`;
- no `.` or `..` segments, and no empty segments;
- no backslashes, NUL or control characters;
- at most 1024 bytes;
- valid UTF-8.

These rules are a strict subset of the gate-result/v1 location path rule.

### Language convention

The language-specific knowledge oracle-guard needs:

- how locked tests are mounted;
- how test IDs are formed;
- which files are test-affecting config;
- which sniff rules apply;
- how holdouts are injected.

Only `rust` exists until after the factory role is complete
([ADR-0008](../adr/ADR-0008-language-conventions-extension-point.md)).

## Error concepts

| Concept | Status | Meaning | Security note |
| --- | --- | --- | --- |
| Workspace violation | `fail` | The workspace content breaks the oracle (`OG-INT`, `OG-INV`, `OG-CFG`, `OG-RED`, `OG-SNF`) | Is the expected result of a faking attempt |
| Input error | `error` | A caller-supplied or trusted input is missing, malformed, too large or in the wrong place (`OG-INP`) | Never a pass. Messages never echo input content |
| Lock error | exit 2, no gate result | `lock` cannot build a valid manifest (`OG-LCK`) | No partial manifest is written |
| Holdout error *(L3)* | `error` or `fail` | `holdout-inject` cannot inject safely (`OG-HLD`) | Never names a holdout file or test |
| Internal error | `error` | A bug: oracle-guard caught a panic (`OG-INP-009`) | Fail closed |
