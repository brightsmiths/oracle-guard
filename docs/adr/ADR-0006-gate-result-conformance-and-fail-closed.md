# ADR-0006: Conform to gate-result/v1 and fail closed

**Status:** Accepted
**Date:** 2026-11-27

## Context

gate-warden defines the standard gate result format and its emission contract:

- [v1 schema](https://github.com/brightsmiths/gate-warden/blob/doc/initial-spec/schemas/gate-result/v1.schema.json)
- [format](https://github.com/brightsmiths/gate-warden/blob/doc/initial-spec/docs/spec/interfaces/gate-result-format.md)
- [feedback](https://github.com/brightsmiths/gate-warden/blob/doc/initial-spec/docs/spec/interfaces/feedback-format.md)

Its main rules:

- the result goes to `GATE_RESULT_FILE`;
- the status is `pass`, `fail` or `error`, with no skip;
- no unknown fields;
- holdout findings are reduced to `req_id`;
- messages are coarse;
- logs are never published.

An earlier plan had oracle-guard define its own result format. The product owner confirmed that
gate-warden's format is authoritative.

## Decision

1. Every check emits exactly one `gate-result/v1` document. oracle-guard defines only the
   *mapping* ([result-format.md](../spec/result-format.md)): rule IDs go in `rule`, typed
   template messages go in `message`, `req_id` comes from the manifest, holdouts get
   `visibility: holdout`, and `tool` is `oracle-guard`.
2. **Status classes.** Workspace violations give `fail` (exit 1). Problems with caller-supplied
   or trusted inputs, and internal errors, give `error` (exit 2). If both occur, the status is
   `error`. A `pass` is reported only if every step completed and found no violation.
3. **Fail closed.** Any parse failure, oversized input, unknown format version, ambiguity or
   caught panic gives `fail` or `error`, never `pass`.
4. **Coarse messages by construction.** Messages come only from templates whose placeholders
   are typed (repository path, validated test ID, rule ID, count, digest). Holdout findings get
   a fixed message with no location or name.
5. **No new fields.** oracle-guard emits only fields that are in the pinned v1 schema.

## Consequences

- gate-warden can run oracle-guard checks like any other pack check. github-integration.md needs
  nothing from oracle-guard.
- The status and exit code always agree with gate-warden's exit code table, so the
  "non-zero + pass gives error" path never triggers.
- Upgrading the schema means vendoring a new copy and re-running conformance tests.

## Alternatives considered

- **An oracle-guard-specific format translated by a wrapper.** An extra component with no
  benefit. Rejected.
- **Report input errors as `fail`.** That would blame the workspace for caller mistakes, and
  gate-warden distinguishes check errors in feedback. Rejected.
