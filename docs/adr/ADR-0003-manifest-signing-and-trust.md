# ADR-0003: Manifest signing, trust roots and replay defence

**Status:** Accepted
**Date:** 2026-11-27

## Context

- The manifest is signed by the oracle role.
- The sandbox holds no keys: `lock` runs inside it, and sandbox-runner signs outside with the
  oracle role key in the key service.
- chain-chronicler's DSSE library arrives at M3. Until then, a minimal Ed25519 signature is
  required.
- gate-warden requires checks to read keys and policy from `GATE_HARNESS_DIR` or
  `GATE_DEFAULT_BRANCH_DIR`, never from the workspace
  ([gate-result-format.md](https://github.com/brightsmiths/gate-warden/blob/doc/initial-spec/docs/spec/interfaces/gate-result-format.md#emission-contract)).
- An agent cannot forge a signature, but it could restore an **older, validly signed**
  manifest, for example one with fewer expected tests.

## Decision

1. **Signature:** detached Ed25519 (RFC 8032, strict verification) in `tests/locked/MANIFEST.sig`
   (`oracle-guard-sig/v1`). The signed message is
   `"oracle-guard/manifest-signature/v1\n" || manifest bytes`. The domain prefix separates
   manifest signatures from anything else the key may sign. No algorithm negotiation: only
   `ed25519` is accepted.
2. **Trust root:** an explicit trusted key set file (`--trusted-keys`). Each key has a key ID
   and a list of purposes (`manifest`, `holdout-set`). The path must be outside the workspace
   (`OG-INP-005`). There is no key discovery and no key in the manifest.
3. **Verify before parse:** every command authenticates the bytes before parsing the YAML
   ([architecture.md](../spec/architecture.md#manifest-loading-pipeline)).
4. **Replay defence:** every manifest-consuming command accepts `--expect-digest`. The gate
   caller pins the digest from a trusted record, such as the oracle stage's signed attestation or
   the default-branch manifest. oracle-guard cannot find the right digest by itself
   ([ADR-0001](ADR-0001-pure-offline-cli.md)). Without the flag, any manifest validly signed by
   a trusted key is accepted, and the result summary says the digest was not pinned.
5. **Migration:** at M3, a DSSE verifier through chain-chronicler is added behind the same
   SignatureVerifier port (payload type
   `application/vnd.brightsmiths.oracle-guard.manifest.v1+yaml`). The `.sig` format becomes
   `/v2` or DSSE. The manifest format does not change.

## Consequences

- sandbox-runner must build the exact signing message. This is a cross-service contract,
  documented in [manifest-schema.md](../spec/manifest-schema.md#signing-message).
- Replay protection depends on the gate definition passing `--expect-digest`. If it does not,
  replay is a residual risk ([security.md](../spec/security.md), T-03).
- Key rotation is a harness edit: add the new key, re-lock, remove the old key.

## Alternatives considered

- **Embed the repository, issue or SHA in the manifest and check them.** oracle-guard has no
  trusted source for those values, so the caller would still have to supply them. Pinning the
  digest is simpler and stronger. Rejected for v1. Subject metadata may come later with DSSE.
- **Public key from the workspace.** It would be trivially forgeable. Rejected.
- **DSSE immediately.** It depends on chain-chronicler, which is M3. Rejected for M1.
