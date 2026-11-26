# ADR-0005: Hashing of locked files and test config

**Status:** Accepted
**Date:** 2026-11-27

## Context

Locked files must be detected as changed if any byte changes. Some test-affecting config sits
inside mixed files, for example `Cargo.toml`, which holds both production and test settings.
The definition requires section-level hashing for those files (L2).

## Decision

- **Locked files and whole-file config:** SHA-256 over the raw bytes, with no normalisation of
  line endings, BOMs or trailing whitespace. Symlinks and other non-regular files are never
  hashed; they are violations (`OG-INT-010`, `OG-LCK-003`).
- **Config sections (L2):**
  1. Parse the TOML file.
  2. Select the table at the dotted section path.
  3. Serialise it as canonical JSON (RFC 8785: sorted keys, no whitespace).
  4. Hash `"oracle-guard/config-section/v1\n" || path || "\n" || section || "\n" || json`.

  Formatting and comment changes do not change the hash. Changes in meaning do.
- Hashes are written in lowercase hex. Digests carry an algorithm prefix where they appear alone
  (`sha256:`).

## Consequences

- A `.gitattributes` or `autocrlf` difference between the oracle stage checkout and the gate
  checkout changes the hashes. Checkouts must be byte-identical; this is an operational
  requirement ([operations.md](../spec/operations.md)).
- Section hashing depends on the TOML parser's value model. The canonicalisation is versioned
  (`/v1`) so that it can change later.

## Alternatives considered

- **Normalise line endings.** It hides real changes, for example in snapshot files. Rejected.
- **Hash the raw text span of a section.** Brittle under reformatting, and ambiguous with dotted
  keys and inline tables. Rejected.
