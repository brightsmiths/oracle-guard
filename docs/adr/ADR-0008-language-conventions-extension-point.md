# ADR-0008: Language conventions as an extension point, Rust only

**Status:** Accepted
**Date:** 2026-11-27

## Context

The software pack supports Rust first, with C# (`tests/App.Tests.Locked/`, TRX) and TypeScript
(`*.locked.test.ts`, JUnit) later. The brief scopes every level (L1 to L3) to Rust. Work beyond
L3 belongs to "after factory role complete".

## Decision

- Everything that is language-specific sits behind a LanguageConvention role:
  - test ID formation and the test report and test list readers;
  - the list of test-affecting config files;
  - sniff rules and source analysis;
  - holdout placement.
- The manifest and the lock spec carry `language`. v1 accepts only `rust`.
- The core (manifest, signing, integrity, inventory policy, result building) is
  language-neutral.
- No C# or TypeScript code is written before the factory role is complete. Those conventions
  are recorded as "after factory role complete".

## Consequences

- Adding a language means a new convention implementation plus fixtures. The core does not
  change.
- Rust names (nextest binary ID) appear only in the Rust convention and its documentation.

## Alternatives considered

- **Hard-code Rust everywhere.** Cheaper now, but it would force a redesign later. Rejected,
  because the seam costs little.
- **Implement all three languages now.** Out of scope for the factory levels. Rejected.
