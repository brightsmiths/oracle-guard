# ADR-0001: oracle-guard is a pure, offline command-line tool

**Status:** Accepted
**Date:** 2026-11-27

## Context

oracle-guard produces the evidence that a locked test oracle has not been changed or bypassed.
The Brightsmiths system brief
([handbook](https://github.com/brightsmiths/handbook/blob/feat/initial/docs/architecture/services.md))
sets three rules for it. It must not:

- act as an authority for build agents;
- contain workflow or GitHub logic;
- hold signing keys.

It runs in the gate sandbox, which has deny-by-default egress, and also in build agents'
sandboxes for feedback. gate-warden owns all GitHub interaction and runs the test commands
itself ([github-integration.md](https://github.com/brightsmiths/gate-warden/blob/doc/initial-spec/docs/spec/interfaces/github-integration.md)).

## Decision

oracle-guard is a single binary that:

- reads files and writes files, and nothing else: no network, no subprocesses (it never runs
  cargo, nextest or git), and no key material other than public keys;
- takes test lists and test reports as input files produced by the caller;
- reads only `GATE_RESULT_FILE`, `GATE_CHECK_ID` and `GATE_WORKSPACE` from the environment, as
  defaults for flags. Every other input is an explicit flag;
- has no notion of SHAs, branches, PRs, issues or attempts.

## Consequences

- Results are reproducible from the same inputs. gate-warden's recomputation guarantee extends
  to oracle-guard.
- The caller (the gate definition in the harness) must run `cargo nextest list` and
  `cargo nextest run`, and pass the outputs in. The stub run for `red-check` is also the
  caller's job.
- Running oracle-guard locally gives an agent nothing it could not compute itself. That is
  acceptable because the authoritative run is gate-warden's.
- No GitHub token, no key service client, and a small attack surface.

## Alternatives considered

- **oracle-guard runs nextest itself.** Simpler for callers, but it couples oracle-guard to the
  toolchain, makes it spawn untrusted code, and duplicates gate-warden's sandboxed execution.
  Rejected.
- **oracle-guard posts results to GitHub.** Forbidden by the brief and by gate-warden being the
  only source of required checks. Rejected.
