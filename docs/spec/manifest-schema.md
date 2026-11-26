# Manifest, lock spec, signature and trusted key set

This page defines four file formats:

- `oracle-guard-manifest/v1`: the manifest.
- `oracle-guard-lock/v1`: the lock spec, which is the input to `lock`.
- `oracle-guard-sig/v1`: the detached signature.
- `oracle-guard-keys/v1`: the trusted key set.

All four use the same **strict YAML subset**
([ADR-0002](../adr/ADR-0002-manifest-format-strict-yaml.md)).

## Strict YAML subset (all four formats)

A document is accepted only if all of these hold:

1. It is UTF-8 without a BOM, uses LF line endings, and is at most 16 MiB (the trusted key set
   and the signature: at most 64 KiB).
2. It is a single document. There is no `---` after content and no `...`.
3. It has no anchors (`&`), aliases (`*`), tags (`!`), merge keys (`<<`) or complex keys.
4. Every mapping has string keys and no duplicate keys.
5. It has no unknown keys, at any level.
6. Every scalar has the type its field declares. Implicit typing such as `yes`, `on`, `0x1F` or
   `1e3` is not accepted where a string is declared.

The **manifest** must also be in [canonical form](#canonical-form). The lock spec and the
trusted key set are human or agent authored and need not be canonical.

## Manifest (`tests/locked/MANIFEST`)

### Example

```yaml
format: "oracle-guard-manifest/v1"
language: "rust"
files:
  - path: "src/billing/invoice_tests.rs"
    sha256: "9f2c0b6e4d3a1f8e7c6b5a4d3c2b1a0f9e8d7c6b5a4f3e2d1c0b9a8f7e6d5c4b"
  - path: "tests/locked/LOCK.yaml"
    sha256: "1d0c..."
  - path: "tests/locked/fixtures/invoice_negative.json"
    sha256: "77d0..."
test_config: []
locked_prefixes:
  - binary: "billing"
    name_prefix: "invoice::tests::"
  - binary: "billing::invoice_api"
    name_prefix: ""
expected_tests:
  - binary: "billing"
    name: "invoice::tests::rejects_negative_amounts"
    req_id: "REQ-017"
  - binary: "billing::invoice_api"
    name: "creates_invoice"
sniff_allowlist: []
policy:
  all_expected_must_run: true
  skipped_allowed: false
  extra_locked_tests_allowed: false
```

### Fields

| Field | Type | Required | Level enforced | Rules |
| --- | --- | --- | --- | --- |
| `format` | string | yes | L1 | Exactly `oracle-guard-manifest/v1` |
| `language` | string | yes | L1 | `rust` (the only supported [language convention](vocabulary.md#language-convention)) |
| `files[]` | list | yes | L1 (`verify`) | At least one entry. Sorted by `path`, compared byte by byte. Paths unique |
| `files[].path` | [repository path](vocabulary.md#repository-path) | yes | L1 | Must not be `tests/locked/MANIFEST` or `tests/locked/MANIFEST.sig` |
| `files[].sha256` | 64 lowercase hex | yes | L1 | SHA-256 of the raw file bytes |
| `test_config[]` | list | yes (may be empty) | L2 (`config-guard`) | Sorted by (`path`, `section`, absent before present). Each (`path`, `section`) unique. L1 `lock` always writes `[]` |
| `test_config[].path` | repository path | yes | L2 | |
| `test_config[].section` | string | no | L2 | A dotted TOML table path, for example `features`, `profile.test` or `target.'cfg(unix)'.dev-dependencies` ([ADR-0005](../adr/ADR-0005-hashing.md)) |
| `test_config[].sha256` | 64 lowercase hex | yes | L2 | Hash of the raw file, or of the canonical section |
| `locked_prefixes[]` | list | yes | L1 (`inventory`) | At least one entry. Sorted by (`binary`, `name_prefix`). Unique |
| `locked_prefixes[].binary` | [binary ID](vocabulary.md#binary-id) | yes | L1 | |
| `locked_prefixes[].name_prefix` | string | yes | L1 | Empty, an exact test name, or a module path ending in `::` |
| `expected_tests[]` | list | yes | L1 (`inventory`) | At least one entry. Sorted by (`binary`, `name`). Test IDs unique. Every entry matches at least one locked prefix |
| `expected_tests[].binary` | binary ID | yes | L1 | |
| `expected_tests[].name` | [test name](vocabulary.md#test-name) | yes | L1 | |
| `expected_tests[].req_id` | [requirement ID](vocabulary.md#requirement-id) | no | L1 | Copied into findings as `req_id` |
| `sniff_allowlist[]` | list | yes (may be empty) | L2 (`sniff`) | Sorted by (`rule`, `path`). Unique. L1 `lock` always writes `[]` |
| `sniff_allowlist[].rule` | `OG-SNF-NNN` | yes | L2 | Must be a sniff rule ID defined in [rule-catalogue.md](rule-catalogue.md#sniff-og-snf-l2-reserved) |
| `sniff_allowlist[].path` | repository path | yes | L2 | |
| `sniff_allowlist[].reason` | string, 1–300 characters | yes | L2 | Free text for reviewers. Never shown in findings |
| `policy.all_expected_must_run` | bool | yes | L1 | Must be `true` |
| `policy.skipped_allowed` | bool | yes | L1 | Must be `false` |
| `policy.extra_locked_tests_allowed` | bool | yes | L1 | Must be `false` |

Limits: at most 100 000 entries each in `files`, `test_config`, `expected_tests` and
`locked_prefixes`, and at most 1 000 in `sniff_allowlist`.

**Why the L2 fields are defined in v1:** `test_config` and `sniff_allowlist` are part of v1 from
the start, so L2 needs no format change. A command enforces only its own fields:

- `verify` checks `files`;
- `config-guard` checks `test_config`;
- `sniff` uses `sniff_allowlist`.

The gate definition must declare every check whose fields a manifest uses
([edge-cases.md](edge-cases.md#manifest-and-signature), E-M7).

**Policy values are constants in v1.** Relaxing any of them is a breaking change and needs
`oracle-guard-manifest/v2`.

### Canonical form

`lock` writes exactly this form, and every command rejects a manifest whose bytes differ from
the re-serialisation of its parsed content (`OG-INT-006`). This leaves one valid byte sequence
for each manifest content, which closes YAML-parser differential attacks.

- Keys appear in the order of the table above. Optional keys are omitted when absent. Empty
  lists are written as `[]`.
- Indentation is two spaces. List items are written as `  - key: value` under their parent key,
  at the same indentation as the example.
- Every string scalar is double-quoted, with JSON-compatible escaping. Non-ASCII characters are
  written literally as UTF-8.
- Booleans are `true` or `false`.
- There are no comments and no trailing spaces. The file ends with exactly one `\n`.

### Manifest digest

`sha256:` followed by the lowercase hex SHA-256 of the exact manifest bytes. `lock` prints the
digest, and `--expect-digest` takes one.

### Exclusions

`files` never lists `tests/locked/MANIFEST` or `tests/locked/MANIFEST.sig`. The lock spec
`tests/locked/LOCK.yaml` is always listed.

## Signing message

Ed25519 (pure, RFC 8032) signs this message:

```
"oracle-guard/manifest-signature/v1" || 0x0A || <exact manifest bytes>
```

The domain prefix stops a manifest signature from being valid for any other message signed with
the oracle role key. sandbox-runner builds this message and has it signed by the key service.
oracle-guard only verifies it ([ADR-0003](../adr/ADR-0003-manifest-signing-and-trust.md)).

From M3, when chain-chronicler's library is available, a DSSE envelope replaces this format. The
payload type is `application/vnd.brightsmiths.oracle-guard.manifest.v1+yaml` and the payload is
the same manifest bytes. Both formats are verified behind the same port.

## Signature (`tests/locked/MANIFEST.sig`)

```yaml
format: "oracle-guard-sig/v1"
algorithm: "ed25519"
key_id: "oracle-role-2026-11"
signature: "base64 of the 64-byte signature"
```

| Field | Rules |
| --- | --- |
| `format` | Exactly `oracle-guard-sig/v1` |
| `algorithm` | Exactly `ed25519`. Any other value is rejected; there is no negotiation |
| `key_id` | A [key ID](vocabulary.md#key-id) that is present in the trusted key set |
| `signature` | Standard base64 with padding that decodes to exactly 64 bytes |

## Trusted key set (`--trusted-keys`)

This file is supplied by the harness and read from outside the workspace.

```yaml
format: "oracle-guard-keys/v1"
keys:
  - key_id: "oracle-role-2026-11"
    algorithm: "ed25519"
    public_key: "base64 of the 32-byte public key"
    purposes: ["manifest"]
```

| Field | Rules |
| --- | --- |
| `keys[]` | 1–64 entries. `key_id` values are unique |
| `keys[].algorithm` | `ed25519` |
| `keys[].public_key` | Base64 that decodes to 32 bytes. Weak (small-order) keys are rejected |
| `keys[].purposes` | A non-empty subset of `manifest` and `holdout-set` *(L3)* |

A key verifies a manifest only if `manifest` is in its `purposes`. Key rotation means adding the
new key, re-locking, then removing the old key. Expiry and revocation belong to the harness,
which simply stops listing the key.

## Lock spec (`tests/locked/LOCK.yaml`)

The lock spec is written by the oracle role. It is the input to `lock`, and it is itself a
locked file.

```yaml
format: "oracle-guard-lock/v1"
language: "rust"
files:
  - "src/**/*_tests.rs"
  - "tests/invoice_api.rs"
  - "tests/locked/fixtures/**"
locked_prefixes:
  - binary: "billing"
    name_prefix: "invoice::tests::"
  - binary: "billing::invoice_api"
    name_prefix: ""
requirements:
  - req_id: "REQ-017"
    binary: "billing"
    name_prefix: "invoice::tests::rejects_negative_amounts"
```

| Field | Rules |
| --- | --- |
| `files[]` | 1–1 000 glob patterns relative to the workspace root:<br>- `*` matches within one path segment;<br>- `**` matches zero or more segments;<br>- `?` matches one character;<br>- no braces, no negation.<br>Every pattern must match at least one regular file (`OG-LCK-002`). Hidden files are matched. `.git/` is never walked |
| `locked_prefixes[]` | As in the manifest. Each must match at least one test in the test list (`OG-LCK-004`) |
| `requirements[]` | Optional. Each entry is `{req_id, binary, name_prefix}`, using locked-prefix matching. It assigns `req_id` to the matched expected tests. If a test matches entries with different `req_id` values, the longest `name_prefix` wins; a tie is `OG-LCK-007`. Every entry must match at least one expected test (`OG-LCK-007`) |

The sniff allowlist and the test config are added to the lock spec at L2 (`test_config[]`:
`{path, section?}`; `sniff_allowlist[]`: `{rule, path, reason}`). They are reserved in
`oracle-guard-lock/v1`, and an L1 `lock` rejects them (`OG-LCK-001`).

## Versioning

- The `format` strings carry the major version. Any change to a field, a rule or the canonical
  form is breaking and creates `/v2`.
- oracle-guard reads only the major versions it knows, and rejects others: `OG-INT-005` for
  the manifest, `OG-INP-006` for the key set, `OG-LCK-001` for the lock spec.
- Reading `v1` stays supported for as long as any repository's default branch may still hold a
  `v1` manifest.
