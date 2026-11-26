# oracle-guard specification

oracle-guard checks the integrity of **locked software test oracles** for the Brightsmiths
software discipline pack. It is a pure, offline command-line tool:

- `lock` turns the oracle role's locked tests into a manifest, which is signed outside the
  sandbox;
- the checks (`verify`, `inventory`, and later `config-guard`, `red-check` and `sniff`) prove
  that the oracle is unchanged and actually ran;
- results are reported in gate-warden's standard `gate-result/v1` format.

oracle-guard holds no keys and has no GitHub or workflow logic. Only gate-warden's run of it is
authoritative.

System context: [Brightsmiths handbook](https://github.com/brightsmiths/handbook/tree/feat/initial/docs),
especially [oracles and anti-faking](https://github.com/brightsmiths/handbook/blob/feat/initial/docs/architecture/oracles.md).
Result format: [gate-warden gate-result/v1](https://github.com/brightsmiths/gate-warden/blob/doc/initial-spec/docs/spec/interfaces/gate-result-format.md).

## Increments

| Level | Milestone | Built by | Scope | Spec depth |
| --- | --- | --- | --- | --- |
| L1 | M1 | Human-led | `lock` → `verify` → `inventory`; the manifest, signature and key set formats; gate-result mapping; faking-vector fixtures | Complete |
| L2 | M2 | Factory | `config-guard`, `red-check`, `sniff` | Outline; rule IDs reserved |
| L3 | M6 | Factory | `holdout-inject`; holdout expectations in `inventory` | Outline; rule IDs reserved |
| — | after factory role complete | — | C# and TypeScript conventions, DSSE-only signatures, more sniff rules | Not specified |

## Documents

| File | Content |
| --- | --- |
| [overview.md](overview.md) | Purpose, users, commands, context and sequence diagrams, non-goals |
| [vocabulary.md](vocabulary.md) | Domain terms and error concepts. **Read first** |
| [manifest-schema.md](manifest-schema.md) | Manifest, lock spec, signature, trusted key set, signing message, canonical form |
| [result-format.md](result-format.md) | How oracle-guard fills gate-result/v1: status, findings, metrics, holdout redaction, truncation |
| [cli.md](cli.md) | Commands, flags, inputs, outputs, exit codes |
| [rule-catalogue.md](rule-catalogue.md) | Every rule ID with its message template, status and fixture |
| [responsibilities.md](responsibilities.md) | CRC cards for core components and adapters |
| [architecture.md](architecture.md) | Business logic, ports and infrastructure; the manifest loading pipeline; data flows |
| [assertions.md](assertions.md) | Given/When/Then assertions (L1 complete, L2 and L3 outlined) |
| [edge-cases.md](edge-cases.md) | Non-standard inputs and failure modes |
| [security.md](security.md) | Assets, actors, STRIDE threats T-01 to T-20, implementation security rules |
| [testing.md](testing.md) | Test layers, fixture catalogue (faking vectors), schema conformance |
| [constraints.md](constraints.md) | Type, error, boundary, resource, performance and testing rules |
| [tradeoffs.md](tradeoffs.md) | Alternatives considered |
| [assumptions.md](assumptions.md) | Challenged assumptions, decided and assumed |
| [operations.md](operations.md) | Build, release, pinning, reference invocations, caller requirements |

Architecture decision records are in [../adr/](../adr/):

| ADR | Decision |
| --- | --- |
| [ADR-0001](../adr/ADR-0001-pure-offline-cli.md) | Pure, offline tool: no GitHub, no keys, no subprocesses |
| [ADR-0002](../adr/ADR-0002-manifest-format-strict-yaml.md) | Manifest is strict, canonical YAML |
| [ADR-0003](../adr/ADR-0003-manifest-signing-and-trust.md) | Ed25519 detached signature, explicit trusted key set, pinned digest against replay |
| [ADR-0004](../adr/ADR-0004-test-identity-and-lock-input.md) | Test ID = (binary ID, test name); locked prefixes; `lock` reads nextest list JSON |
| [ADR-0005](../adr/ADR-0005-hashing.md) | Raw-byte SHA-256; canonical section hashing for config (L2) |
| [ADR-0006](../adr/ADR-0006-gate-result-conformance-and-fail-closed.md) | Conform to gate-result/v1; `fail` vs `error`; fail closed |
| [ADR-0007](../adr/ADR-0007-oracle-policy-checks-red-check-and-sniff.md) | red-check target set; sniff exceptions only through the signed allowlist |
| [ADR-0008](../adr/ADR-0008-language-conventions-extension-point.md) | Language conventions as an extension point; Rust only |

## Reading order

vocabulary → overview → manifest-schema → result-format → cli → rule-catalogue →
responsibilities → architecture → ADRs → assertions → edge-cases → security → constraints →
testing.

## Workflow for downstream roles

1. **Interface designer:**
   - turn [responsibilities.md](responsibilities.md) and [architecture.md](architecture.md) into
     crates, types and port traits;
   - follow [constraints.md](constraints.md);
   - choose the YAML library ([tradeoffs.md](tradeoffs.md#t-2-yaml-parser-choice)).
2. **Task planner:** plan L1 in order, `lock` → `verify` → `inventory`, with the fixtures from
   [testing.md](testing.md#fixture-catalogue) as their own tasks. L2 and L3 stay in the backlog
   until their milestones.
3. **Tester:** one test per assertion in [assertions.md](assertions.md), and one fixture per
   rule in [rule-catalogue.md](rule-catalogue.md).
4. **Coder:** implement against the interfaces. Every finding uses a catalogued rule ID.
5. **Security reviewer:** review against [security.md](security.md), especially the loading
   pipeline order, path handling, XML and YAML hardening, and holdout redaction.

Gaps found downstream are fixed by editing the relevant file here (usually `edge-cases.md`,
`assertions.md` or `vocabulary.md`), not by working around them in code.

## Repository rules

This repository is a control-plane and security-boundary repository:

- every PR gets a human review;
- gate configuration is read from the default branch;
- this spec is owned by the design zone (zone A).
