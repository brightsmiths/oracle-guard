# Tradeoffs

The alternatives considered, with the choice made. Formal decisions are in the
[ADRs](../adr/).

## T-1: Who runs the tests

| Option | Pros | Cons |
| --- | --- | --- |
| **Caller runs nextest and passes the outputs in (chosen)** | Pure tool, no subprocesses, deterministic, fits gate-warden's sandboxed execution | Callers must wire two commands |
| oracle-guard runs nextest | One command for callers | Spawns untrusted code, couples to the toolchain, duplicates gate-warden |

Decision: [ADR-0001](../adr/ADR-0001-pure-offline-cli.md).

## T-2: YAML parser choice

The format decision (strict canonical YAML) is in
[ADR-0002](../adr/ADR-0002-manifest-format-strict-yaml.md). The library is still to be chosen.
The candidates must:

- expose anchors, aliases, tags and duplicate keys as errors (or as events we can reject);
- bound depth and size;
- be maintained.

| Option | Notes |
| --- | --- |
| Event-level parser (for example `saphyr-parser` or `yaml-rust2` events) plus our own strict builder | Full control over rejection. More code |
| `serde_yaml`-style deserialiser with `deny_unknown_fields` | Less code, but anchor and alias rejection may not be possible. Check maintenance status |
| Our own parser for the canonical subset only | Smallest surface for the manifest. Lock spec and key set are hand-written, so they still need a tolerant (but strict) parser |

Recommendation: an event-level parser for every format, plus the canonical round-trip check for
the manifest. The interface designer confirms the library.

## T-3: Expected test representation

| Option | Pros | Cons |
| --- | --- | --- |
| **Objects `{binary, name, req_id?}` (chosen)** | Unambiguous. Carries `req_id` for gate-result findings | Differs from the handbook's plain-string example |
| Plain strings `module::path::test` | Matches the handbook example | Ambiguous across binaries. No `req_id` |

Decision: [ADR-0004](../adr/ADR-0004-test-identity-and-lock-input.md). The handbook example is
illustrative; this is a refinement, not a conflict of purpose.

## T-4: Replay defence

| Option | Pros | Cons |
| --- | --- | --- |
| **`--expect-digest` pinned by the caller (chosen)** | Simple, strong, no git logic | Depends on the gate definition using it |
| Bind the manifest to repository, issue or SHA | Self-describing | Still needs trusted values from the caller. SHA binding is impossible because the implementation changes the SHA |
| Monotonic counter | Detects rollback | Needs state, which is forbidden in a pure tool |

Decision: [ADR-0003](../adr/ADR-0003-manifest-signing-and-trust.md).

## T-5: Input problems as `fail` or `error`

Chosen: workspace violations give `fail`, and caller or trusted input problems give `error`.
Both are non-passing. The split tells CogWorks whether the agent or the configuration is at
fault (gate-warden `error_kind: reported`).
[ADR-0006](../adr/ADR-0006-gate-result-conformance-and-fail-closed.md).

## T-6: Section hashing canonicalisation (L2)

| Option | Pros | Cons |
| --- | --- | --- |
| **Parsed value to canonical JSON (chosen)** | Formatting-insensitive. Precise meaning | Depends on the TOML value model |
| Raw text span | Simple | Brittle. Dotted keys and inline tables are ambiguous |
| Whole-file only | Simplest | Every version bump in `Cargo.toml` would need a re-lock |

Decision: [ADR-0005](../adr/ADR-0005-hashing.md).

## T-7: Sniff exceptions (L2)

Chosen: allowlist in the signed manifest only.
[ADR-0007](../adr/ADR-0007-oracle-policy-checks-red-check-and-sniff.md).

## T-8: One binary vs a library plus a binary

Chosen: a workspace with a core library crate and a thin binary crate. The interface designer
decides the exact crate layout. This keeps the core testable and lets other Rust tools reuse
manifest verification later, without changing the command-line interface.
