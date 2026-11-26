# Operations

oracle-guard is a command-line tool, not a service. It has no deployment, no monitoring endpoint
and no state. "Operations" here means how it is built, distributed, pinned and invoked.

## Build and release

- Built in CI on this repository, with a reproducible build: pinned toolchain via
  `rust-toolchain.toml`, `--locked`, and `SOURCE_DATE_EPOCH`.
- The release artefact is a statically linked Linux binary (`x86_64` and `aarch64` musl). It is
  attached to a GitHub Release by release-regent, with SHA-256 checksums. From M6 it also carries
  release-ratifier provenance.
- Versioning follows SemVer:
  - **major** for breaking command-line or format changes (a new manifest `format` major, a
    removed flag, a changed exit code meaning);
  - **minor** for new commands or rules (L2, L3);
  - **patch** for fixes.
- A new rule ID is a minor change. Changing an existing rule's trigger so that it fires more
  often is called out in the release notes.

## Pinning and installation

- The harness gate definitions pin oracle-guard by **version and SHA-256**. The gate sandbox
  image installs that exact binary.
- Build agents' sandboxes may include the same binary, for feedback runs.
- The oracle stage sandbox includes it for `lock`.

## Invocation by gate-warden (reference wiring)

The harness owns gate definitions. This is the expected shape, not a gate-warden contract.

```sh
# check: oracle-guard.verify
oracle-guard verify \
  --trusted-keys "$GATE_HARNESS_DIR/packs/software/oracle-keys.yaml" \
  --expect-digest "<pinned manifest digest from the oracle stage attestation>"

# check: oracle-guard.inventory (after the gate's test run produced the JUnit report)
oracle-guard inventory \
  --trusted-keys "$GATE_HARNESS_DIR/packs/software/oracle-keys.yaml" \
  --test-report "<report path outside test-writable locations>"
```

`GATE_RESULT_FILE`, `GATE_CHECK_ID` and `GATE_WORKSPACE` are picked up automatically.

**Operational requirements on callers:**

- The gate checkout must be byte-identical to the committed tree: no `autocrlf`, and no smudge
  filters for locked paths.
- The nextest gate profile must set `retries = 0`. It must neither use `--run-ignored` nor
  override `default-filter` beyond what the locked config allows.
- The JUnit report must be written where test code cannot overwrite it (security T-07).

## Invocation in the oracle stage

```sh
cargo nextest list --message-format json > "$TMP/nextest-list.json"
oracle-guard lock --test-list "$TMP/nextest-list.json"
# stdout: sha256:<digest>  → sandbox-runner signs and records the digest
```

## Diagnostics

- stderr carries `<rule-id>: <message>` lines plus, on internal errors, a short backtrace
  without input content.
- gate-warden never publishes stderr. Operators see it in sandbox diagnostics.
- There is no telemetry. chain-chronicler records gate results as evidence (from M3).

## Compatibility matrix

Each release documents:

- the supported manifest, lock spec, signature and key set formats;
- the gate-result schema version and the gate-warden commit it was vendored from;
- the cargo-nextest version range validated for list and JUnit output.
