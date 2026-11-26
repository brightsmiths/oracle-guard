# ADR-0002: Manifest format is strict, canonical YAML

**Status:** Accepted
**Date:** 2026-11-27

## Context

The handbook shows the manifest as YAML
([oracles.md](https://github.com/brightsmiths/handbook/blob/feat/initial/docs/architecture/oracles.md#manifest)).
Humans review it in PRs. General YAML has anchors, aliases, tags, implicit typing and duplicate
keys. These features allow parser-differential and resource-exhaustion attacks, and the same
content can be written as many different byte sequences. The signature covers bytes.

The product owner decided: strict YAML, with no anchors or aliases, unknown fields rejected and
deterministic output.

## Decision

- The manifest, the lock spec, the signature file and the trusted key set use a strict YAML
  subset:
  - one document;
  - no anchors, aliases, tags or merge keys;
  - no duplicate keys, no unknown keys;
  - explicit scalar types;
  - UTF-8, LF line endings, size-capped.
- The manifest must also be in **canonical form**: fixed key order, sorted lists, every string
  double-quoted, two-space indentation, no comments, and a single trailing newline. `lock`
  writes exactly that form, and readers reject any manifest whose re-serialisation differs from
  its bytes (`OG-INT-006`).
- The format is versioned through the `format` field (`oracle-guard-manifest/v1`).
- The v1 schema also defines the L2 fields `test_config` and `sniff_allowlist`, so L2 needs no
  format change.
- Expected tests are objects `{binary, name, req_id?}` rather than the plain strings in the
  handbook example ([ADR-0004](ADR-0004-test-identity-and-lock-input.md)).

Full details: [manifest-schema.md](../spec/manifest-schema.md).

## Consequences

- One valid byte sequence per content: no ambiguity between what was signed and what was
  understood.
- Diffs are reviewable and stable.
- Hand-editing a manifest always fails verification. That is intended, because only `lock`
  writes manifests.
- A YAML library must expose enough of the parse events to reject anchors, tags and duplicates
  (see [tradeoffs.md](../spec/tradeoffs.md#t-2-yaml-parser-choice)).

## Alternatives considered

- **Canonical JSON (RFC 8785).** A smaller attack surface, but less readable and different from
  the handbook. Rejected by the product owner.
- **Permissive YAML with signature over bytes only.** Leaves room for parser differentials.
  Rejected.
