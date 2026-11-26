# ADR-0007: red-check target set and sniff exceptions

**Status:** Accepted
**Date:** 2026-11-27

## Context

Two L2 policy questions were decided by the product owner:

1. Which tests does `red-check` judge?
2. Can `sniff` findings be suppressed?

Both are oracle-integrity controls. Tests that cannot fail assert nothing. Test-only behaviour
in production code defeats the oracle.

## Decision

- **red-check target set:**
  - With `--previous-manifest <path>`, the targets are the expected tests that are not in the
    previous manifest (compared by test ID).
  - Without the flag, the targets are all expected tests.
  - The previous manifest is a trusted input. It is read from outside the workspace, typically
    the default-branch copy, and its signature is verified with the same trusted key set
    (`OG-INP-011` on failure).
  - Every target must have run in the stub run and failed (`OG-RED-001`…`003`). An empty target
    set is a `pass` with an explanatory summary.
- **sniff exceptions:**
  - There are no inline suppressions. Comments and attributes such as `// oracle-guard:allow`
    have no effect.
  - The only exception mechanism is `sniff_allowlist` in the **signed manifest**:
    `{rule, path, reason}`, written by the oracle role through the lock spec.
  - Allowlist entries apply to one rule in one file. Unused entries are reported as notices
    (`OG-SNF-009`).

## Consequences

- An implementer cannot exempt its own code. Every exception carries the oracle role's
  signature and is reviewed with the oracle.
- red-check without a previous manifest re-judges old tests. They should still fail against
  stubs, but callers that have no stubs for old code should pass `--previous-manifest`.
- Allowlist edits need a re-lock.

## Alternatives considered

- **Inline `#[allow]`-style suppressions.** The implementer controls the source, so this would
  be self-certification. Rejected.
- **red-check judged by a git diff.** Needs git logic, which is forbidden by ADR-0001.
  Rejected.
