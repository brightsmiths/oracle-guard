# ADR-0004: Test identity, locked prefixes and lock input

**Status:** Accepted
**Date:** 2026-11-27

## Context

`inventory` must decide, from a JUnit report alone:

- whether every expected test ran;
- whether any *unexpected* locked test exists.

JUnit does not record source files, and Rust locked tests are mounted with
`#[path = "foo_tests.rs"] mod tests;`. So the module path does not follow from the file name in
every case, and the same test name can occur in two binaries.

The product owner decided that `lock` takes the output of
`cargo nextest list --message-format json`, filtered by `locked_prefixes`.

## Decision

- **Test ID** = (nextest binary ID, test name). In JUnit these are the `<testcase>` `classname`
  (or the enclosing `<testsuite name>`) and its `name`. If the two binary sources disagree, the
  report is invalid (`OG-INP-003`). The display form is `<binary-id> <test-name>`.
- **Locked prefixes** are declared by the oracle role in the lock spec and copied into the
  manifest. A prefix `{binary, name_prefix}` matches on whole `::` segments. The prefix
  `""` matches the whole binary, which suits integration test files.
- **Expected tests** = every test in the test list that matches a locked prefix. `lock` refuses
  tests that are ignored (`OG-LCK-005`) or filtered out (`OG-LCK-006`), and refuses prefixes
  that match nothing (`OG-LCK-004`).
- **Unexpected locked test** = a test in the report that matches a locked prefix and is not
  expected (`OG-INV-004`).
- Expected tests carry an optional requirement ID, assigned through the lock spec's
  `requirements` mapping. This feeds `req_id` in findings.

## Consequences

- The oracle role must keep locked tests in modules or binaries that the implementer does not
  use for its own tests. Implementer tests live in `tests/own/`, which are separate binaries.
- Renaming a locked test, or moving it to another module, changes its test ID. That requires a
  re-lock.
- The nextest list and JUnit formats become pinned dependencies. Fixtures are pinned to a
  nextest version ([testing.md](../spec/testing.md)).

## Alternatives considered

- **Map test IDs to files through source parsing.** Fragile with `#[path]`, macros and
  generated tests. Rejected.
- **Plain-string test IDs joined with `::`.** Ambiguous between the binary and the module path.
  Rejected.
- **A hand-written expected list.** Error-prone, and it lets an oracle silently omit tests.
  Rejected.
