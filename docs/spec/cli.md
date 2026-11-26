# Command-line interface

`oracle-guard <command> [flags]`. The interface designer chooses flag spellings and the argument
parser. The behaviour, inputs, outputs and exit codes on this page are binding.

## Common rules

| Rule | Detail |
| --- | --- |
| Exit codes | `0` pass, `1` fail, `2` error (input, usage or internal). `lock` and `holdout-inject` use `0` and `2` only, except that `holdout-inject` uses `1` for content violations |
| Gate result | Checks write one gate-result/v1 document ([result-format.md](result-format.md#emission-contract)) |
| Workspace | `--root <dir>`, else `GATE_WORKSPACE`, else the current directory. All repository paths are relative to it |
| Trusted inputs | `--trusted-keys`, `--previous-manifest` and `--holdouts` must resolve, after following symlinks, to paths outside the workspace (`OG-INP-005`) |
| Manifest loading | Every command that reads a manifest loads it through the same steps ([architecture.md](architecture.md#manifest-loading-pipeline)): read bytes → verify signature → check the pinned digest if given → parse → check canonical form. The manifest is always `<root>/tests/locked/MANIFEST`, and the signature is always `<root>/tests/locked/MANIFEST.sig` |
| `--expect-digest <sha256:…>` | Accepted by every manifest-consuming command. On a mismatch the result is `OG-INT-004` |
| Determinism | The same inputs give byte-identical outputs: the manifest, the gate result and the stderr diagnostics, apart from timing |
| Offline | No network access, no subprocesses, no reading of the environment beyond the three `GATE_*` variables in [result-format.md](result-format.md#emission-contract) |
| Diagnostics | Human-readable lines on stderr, `<rule-id>: <message>`. Never needed to determine the verdict |
| `--version` | Prints `oracle-guard <semver>` and the supported format versions |

## `lock` (L1)

Builds the unsigned manifest.

```
oracle-guard lock --test-list <nextest-list.json> [--root <dir>] [--out <path>]
```

| Input | Source |
| --- | --- |
| Lock spec | `<root>/tests/locked/LOCK.yaml` ([manifest-schema.md](manifest-schema.md#lock-spec-testslockedlockyaml)) |
| Test list | Output of `cargo nextest list --message-format json`, run in the same checkout without filter arguments |
| `--out` | Default `<root>/tests/locked/MANIFEST` |

Steps:

1. Parse and validate the lock spec (`OG-LCK-001`).
2. Expand the file globs over a sorted walk of the workspace (`.git/` excluded). Require at least
   one match per glob (`OG-LCK-002`) and require every match to be a regular file
   (`OG-LCK-003`). Add the lock spec. Drop `MANIFEST` and `MANIFEST.sig`.
3. Hash each file (raw bytes, SHA-256).
4. Read the test list. Select every test whose test ID matches a locked prefix. Every locked
   prefix must select at least one test (`OG-LCK-004`).
5. Reject any selected test that nextest reports as `ignored` (`OG-LCK-005`) or as not matching
   the filter (`OG-LCK-006`).
6. Apply the `requirements` mapping (`OG-LCK-007`). Require at least one expected test
   (`OG-LCK-008`).
7. Serialise in canonical form. Write it atomically to `--out`: a temporary file in the same
   directory, then a rename.
8. Print the [manifest digest](vocabulary.md#manifest-digest) as the only line on stdout.

On any error, nothing is written, all `OG-LCK` and `OG-INP` diagnostics go to stderr, and the
exit code is `2`. `lock` never writes `MANIFEST.sig` and never touches any key.

## `verify` (L1)

Checks the manifest's integrity against the workspace.

```
oracle-guard verify --trusted-keys <keys.yaml> [--expect-digest <d>] [--root <dir>]
                    [--result-file <p>] [--check-id <id>] [--format json|text]
```

- Loads the manifest (`OG-INT-001`…`007`).
- For each `files[]` entry, checks that the file exists (`OG-INT-008`), is a regular file and
  not a symlink (`OG-INT-010`), and has the listed hash (`OG-INT-009`).
- Reports every file under `tests/locked/` that is not listed, other than `MANIFEST` and
  `MANIFEST.sig` (`OG-INT-011`).
- Reports all violations, and does not stop at the first one.

## `inventory` (L1)

Checks the test report against the expected tests.

```
oracle-guard inventory --trusted-keys <keys.yaml> --test-report <junit.xml>
                       [--holdout-expectations <path>]   (L3)
                       [--expect-digest <d>] [--root <dir>] [--result-file <p>] [--check-id <id>]
```

Under the v1 policy, for every expected test, plus every holdout test from L3:

| Test outcome | Rule |
| --- | --- |
| `passed` | ok |
| `not-run` | `OG-INV-001` |
| `skipped` | `OG-INV-002` |
| `failed` | `OG-INV-003` |
| `flaky` | `OG-INV-006` |
| reported more than once | `OG-INV-005` |

Every test ID in the report that matches a locked prefix but is not expected gives
`OG-INV-004`. Tests outside every locked prefix are ignored: they are the implementer's own
tests.

A test report that is malformed, too large (over 256 MiB) or has an empty `<testsuites>` root
gives `OG-INP-003`, `OG-INP-002` or `OG-INP-003` respectively, and the status is `error`.

## `config-guard` (L2, outline)

```
oracle-guard config-guard --trusted-keys <keys.yaml> [--expect-digest <d>] [--root <dir>] ...
```

- Checks each `test_config[]` entry: `OG-CFG-001` to `OG-CFG-005`.
- Discovers test-affecting config files that exist in the workspace but have no manifest entry
  (`OG-CFG-006`). For Rust this covers every `Cargo.toml`, every `build.rs`,
  `.cargo/config.toml`, `.cargo/config`, `.config/nextest.toml` and `rust-toolchain.toml`.
- `lock` gains `test_config` in the lock spec at the same level.

## `red-check` (L2, outline)

```
oracle-guard red-check --trusted-keys <keys.yaml> --test-report <stub-run-junit.xml>
                       [--previous-manifest <path>] [--expect-digest <d>] ...
```

- The **target set** is the expected tests that are not in the previous manifest when
  `--previous-manifest` is given. Otherwise it is every expected test
  ([ADR-0007](../adr/ADR-0007-oracle-policy-checks-red-check-and-sniff.md)).
- The previous manifest is a trusted input, for example the default-branch copy under
  `GATE_DEFAULT_BRANCH_DIR`. Its signature is verified too; if it is invalid the result is
  `OG-INP-011`.
- Each target must have run and failed: `OG-RED-001` if it passed, `OG-RED-002` if it did not
  run, `OG-RED-003` if it was skipped. An empty target set gives `pass` with the summary
  `no new oracle tests`.
- The caller produces the stub run. oracle-guard runs nothing.

## `sniff` (L2, outline)

```
oracle-guard sniff --trusted-keys <keys.yaml> [--expect-digest <d>] [--root <dir>] ...
```

- Parses every `.rs` file under the workspace's `src/` directories (all workspace crates) that
  is not a locked file, and applies `OG-SNF-001` to `OG-SNF-008`.
- Exceptions come only from `sniff_allowlist` in the signed manifest. Inline suppressions
  (comments, attributes) are ignored and have no effect.
- Unused allowlist entries give `OG-SNF-009` at severity `notice`.

## `holdout-inject` (L3, outline)

```
oracle-guard holdout-inject --trusted-keys <keys.yaml> --holdouts <dir>
                            --holdout-expectations-out <path> [--root <dir>] ...
```

- Reads the holdout set from a local directory outside the workspace, which the gate caller
  has already fetched. Verifies its signature with a key whose purposes include `holdout-set`
  (`OG-HLD-001`).
- Copies files into `tests/`, or replaces the content of the holdout placeholder
  (`OG-HLD-002`…`004`).
- Writes the holdout expectations (test IDs and requirement IDs) to a path outside the
  workspace, for `inventory --holdout-expectations`.
- Reports holdout ID collisions (`OG-HLD-005`).
- Every finding has `visibility: holdout` and no location.
