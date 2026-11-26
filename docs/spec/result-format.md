# Result format: mapping onto gate-result/v1

oracle-guard does **not** define its own result format. Every check writes exactly one document
in gate-warden's standard gate result format, v1. The canonical sources are:

- Schema: [gate-result/v1.schema.json](https://github.com/brightsmiths/gate-warden/blob/doc/initial-spec/schemas/gate-result/v1.schema.json)
- Format and emission contract: [gate-result-format.md](https://github.com/brightsmiths/gate-warden/blob/doc/initial-spec/docs/spec/interfaces/gate-result-format.md)
- What build agents see: [feedback-format.md](https://github.com/brightsmiths/gate-warden/blob/doc/initial-spec/docs/spec/interfaces/feedback-format.md)
- GitHub side (gate-warden only): [github-integration.md](https://github.com/brightsmiths/gate-warden/blob/doc/initial-spec/docs/spec/interfaces/github-integration.md)

If this page and gate-warden's documents disagree, gate-warden wins. This page only fixes how
oracle-guard fills the fields. The decision is recorded in
[ADR-0006](../adr/ADR-0006-gate-result-conformance-and-fail-closed.md).

## Which commands emit a gate result

| Command | Emits gate result | Notes |
| --- | --- | --- |
| `verify`, `inventory`, `config-guard`, `red-check`, `sniff` | yes | These are the checks |
| `lock` | no | A producer. It writes the manifest and reports problems as `OG-LCK` diagnostics on stderr ([cli.md](cli.md#lock-l1)) |
| `holdout-inject` *(L3)* | yes | So that gate-warden can record a failed injection as a non-passing check. The final contract depends on how gate-warden sequences preparation steps ([assumptions.md](assumptions.md), AS-14) |

## Emission contract

1. **Destination.** The result goes to the first of these that is set:
   - `--result-file <path>`;
   - the `GATE_RESULT_FILE` environment variable;
   - otherwise stdout.

   With a result file, oracle-guard writes nothing else to that file and fails if the file
   already exists (`OG-INP-008`). With `--format text` and no result file, a human-readable
   rendering goes to stdout instead of JSON.
2. **`check_id`.** The first of these that is set:
   - `--check-id`;
   - `GATE_CHECK_ID`;
   - the default `oracle-guard.<command>`, for example `oracle-guard.inventory`.

   The value must match `^[a-z0-9][a-z0-9._-]{0,63}$` (`OG-INP-007`).
3. **Workspace.** `--root`, else `GATE_WORKSPACE`, else the current directory.
4. **Trusted inputs** (`--trusted-keys`, `--previous-manifest`, `--holdouts`) must resolve
   outside the workspace (`OG-INP-005`). This mirrors gate-warden's rule that checks read
   trusted inputs from `GATE_DEFAULT_BRANCH_DIR` or `GATE_HARNESS_DIR`, never from
   `GATE_WORKSPACE`.
5. **Size.** The document is UTF-8 JSON of at most 1 MiB (enforced by the
   [limits](#limits-and-truncation)).
6. **Logs.** Diagnostics go to stderr only. gate-warden never publishes logs, so nothing
   needed for a verdict may appear only on stderr.
7. **Environment.** oracle-guard reads only `GATE_RESULT_FILE`, `GATE_CHECK_ID` and
   `GATE_WORKSPACE`, and only in the command-line adapter. The other `GATE_*` variables are
   passed in explicitly as flag values by the gate definition.

## Field mapping

| gate-result/v1 field | oracle-guard value |
| --- | --- |
| `format` | `"gate-result/v1"` |
| `check_id` | As above |
| `status` | `pass`, `fail` or `error` ([status rules](#status-rules)) |
| `summary` | One coarse line, at most 200 characters (gate-warden publishes 200). For example: `verify: 42 locked files match manifest sha256:9f2c…` or `inventory: 3 of 120 expected tests did not pass` |
| `findings[]` | One per violation ([finding mapping](#finding-mapping)) |
| `metrics[]` | Per command ([metrics](#metrics)) |
| `tool` | `{"name": "oracle-guard", "version": "<semver>"}` |
| `seed` | Never set. oracle-guard is deterministic |

oracle-guard never emits any other field. The schema forbids unknown properties, and a new field
first needs a gate-warden minor revision of v1.

### Status rules

| Situation | `status` | Exit code |
| --- | --- | --- |
| No findings of severity `error` | `pass` | 0 |
| At least one workspace violation (`OG-INT`, `OG-INV`, `OG-CFG`, `OG-RED`, `OG-SNF`, `OG-HLD` content rules) | `fail` | 1 |
| Any input error (`OG-INP`), or an internal error | `error` | 2 |
| Both a violation and an input error | `error`. Findings include both | 2 |

These match gate-warden's exit code table: a non-zero exit always comes with `fail` or `error`,
and `pass` always comes with exit 0. **There is no skip.** A command with nothing to judge must
not be declared in the gate. Where an empty input is valid, for example a `red-check` with no
added tests, the result is `pass` with a summary that says so ([edge-cases.md](edge-cases.md)).

`notice` findings never change the status. `warning` is not used.

### Finding mapping

| gate-result/v1 finding field | oracle-guard value |
| --- | --- |
| `rule` | The rule ID, for example `OG-INV-001` ([rule-catalogue.md](rule-catalogue.md)) |
| `severity` | `error` for every violation and input error. `notice` for informational rules (`OG-SNF-009`) |
| `message` | A fixed template from the rule catalogue, filled only with repository paths, test IDs, rule IDs and counts. At most 300 characters (gate-warden's publish cap), cut at a character boundary with `…` |
| `location.path` | The repository path where one exists: a locked file, a config file, a source file. Never set for holdouts, and never set for paths that fail the repository path rules |
| `location.start_line` / `start_column` / `end_*` | Sniff findings only (L2) |
| `req_id` | The expected test's `req_id` from the manifest, or the holdout test's `req_id` |
| `visibility` | `holdout` for every finding about a holdout test or holdout file. Otherwise omitted, which means the default `visible` |

**Message content rules** (gate-warden security T-06 and feedback rules):

- Never test source, file content, fixture content, diff content or stderr from tests.
- Never content copied from untrusted inputs except validated test IDs and repository paths.
  Test IDs that fail the [test ID](vocabulary.md#test-id) rules are replaced by
  `<invalid test id>`.
- **Holdout findings** use only the fixed message `holdout test did not pass`. They have no
  location and no test name, and carry the `req_id` from the holdout set. gate-warden reduces
  them to `holdout_failures: [req_id]`. oracle-guard keeps them coarse anyway, as defence in
  depth.

### Metrics

Metric names follow `^[a-z0-9][a-z0-9._-]{0,99}$`. All metrics have `unit: "count"`.

| Command | Metrics |
| --- | --- |
| `verify` | `locked.files`, `locked.files.mismatched`, `locked.files.missing` |
| `inventory` | `tests.expected`, `tests.passed`, `tests.failed`, `tests.skipped`, `tests.not_run`, `tests.unexpected_locked`, `tests.flaky`; from L3 also `tests.holdout.expected` and `tests.holdout.not_passed` |
| `config-guard` | `config.entries`, `config.mismatched` |
| `red-check` | `red.targets`, `red.passed_against_stubs` |
| `sniff` | `sniff.files_scanned`, `sniff.violations`, `sniff.allowlisted` |
| all | `findings.dropped` (only when truncation happened) |

Holdout metrics are counts only. They give away no names.

## Limits and truncation

- At most 1 000 findings. If there are more, oracle-guard sorts all findings the way gate-warden
  does (severity, path, line, rule, message), keeps the first 999, and adds one finding:
  - `rule` `OG-INP-010`, severity `notice`;
  - message `N further findings omitted`.

  It also sets the metric `findings.dropped`. Truncation never changes the status.
- At most 100 metrics. oracle-guard emits at most 20.
- The 1 MiB document cap is guaranteed by these limits: 1 000 findings × at most about 1.5 KiB
  each.

## Examples

`verify` fails on an edited fixture:

```json
{
  "format": "gate-result/v1",
  "check_id": "oracle-guard.verify",
  "status": "fail",
  "summary": "verify: 1 of 42 locked files does not match manifest sha256:9f2c0b6e…",
  "findings": [
    {
      "location": { "path": "tests/locked/fixtures/invoice_negative.json" },
      "rule": "OG-INT-009",
      "severity": "error",
      "message": "locked file content does not match the manifest hash"
    }
  ],
  "metrics": [
    { "name": "locked.files", "value": 42, "unit": "count" },
    { "name": "locked.files.mismatched", "value": 1, "unit": "count" },
    { "name": "locked.files.missing", "value": 0, "unit": "count" }
  ],
  "tool": { "name": "oracle-guard", "version": "0.1.0" }
}
```

`inventory` fails on a deleted `mod tests;` and a holdout failure (L3):

```json
{
  "format": "gate-result/v1",
  "check_id": "oracle-guard.inventory",
  "status": "fail",
  "summary": "inventory: 2 of 121 expected tests did not pass",
  "findings": [
    {
      "rule": "OG-INV-001",
      "severity": "error",
      "message": "expected locked test did not run: billing invoice::tests::rejects_negative_amounts",
      "req_id": "REQ-017"
    },
    {
      "rule": "OG-INV-003",
      "severity": "error",
      "message": "holdout test did not pass",
      "req_id": "REQ-021",
      "visibility": "holdout"
    }
  ],
  "tool": { "name": "oracle-guard", "version": "0.3.0" }
}
```

## Conformance testing

Every gate result oracle-guard produces in its test suite is validated against a vendored copy of
`v1.schema.json`. The copy is pinned to the gate-warden commit it was taken from
([testing.md](testing.md#gate-result-conformance)).
