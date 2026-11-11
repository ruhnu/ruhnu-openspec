# Design

## Context

See proposal.md for the findings. This document records the decision taken for each one. It grows as findings are added.

Findings 1, 2 and 8 are in cdoc2-shares-server; findings 3, 5, 9 and 10 in cdoc2-java-ref-impl.

In cdoc2-shares-server, `KeyShareApiService.getExpiryTime` picks a default and a maximum duration: the `cdoc2.ruhnu.pk.*` pair for a `ruhnu/pk/` recipient when the flag is on, otherwise the global `key-share.*` pair. The maximum is compared only with a client-requested `x-expiry-time`. The comparison logic is inherited from upstream.

## Goals / Non-Goals

**Goals:**
- No input from an unauthenticated caller on the SSH path produces a server error.
- A `ruhnu/pk/` share never outlives `cdoc2.ruhnu.pk.max-expiration-duration`.
- Setting only the maximum is enough to cap the lifetime.

**Non-Goals:**
- Changing the global `key-share.*` lifetimes path. It has the same gap, but it is upstream behaviour and the spec promises it stays as upstream.
- Client-side handling of `x-expiry-time-adjusted` (review finding 10, not decided yet).

## Decisions

### D1 (finding 2): clamp the default to the maximum, and warn

For `ruhnu/pk/` shares the effective default is `min(default, max)`.

- **Why clamp, not refuse to start:** an operator lowering only the maximum expresses a clear intent, a cap. Refusing to start until the default is lowered as well adds a step without adding safety. Considered and rejected: failing startup when the default exceeds the maximum.
- **Warning:** logged once at startup, at `WARN`, only when the flag is on, naming both properties and their values. Not logged per request, which would flood the log on a busy server. With the flag off the values are unused, so nothing is logged.
- **`x-expiry-time-adjusted` stays `false`:** the header reports that a client-requested expiry was changed. Here the client requested none.
- **Where:** the clamp sits in the `ruhnu/pk/` branch of `getExpiryTime` only, so the global branch stays byte-for-byte as upstream.

### D2 (finding 1): parse failures of `aud` are verification failures, on the SSH path only

`ValidateAuthToken.executeSshPk` turns every exception raised while parsing the `aud` URL into a `VerificationException`, which the caller already maps to 401. It also decodes the nonce as base64url there, before the shared audience check, so a nonce that cannot be decoded fails at the same place.

- **Why only `executeSshPk`:** the shared `checkNonceFromDB`, the Smart-ID/Mobile-ID `execute` and `ShareAccessData` in cdoc2-auth have the same gaps, but they are upstream code and reachable only with a trusted certificate. The spec requires the same response statuses as upstream with the flag off, so they are left as they are. Considered and rejected: fixing `ShareAccessData.fromURL` in cdoc2-auth, or catching in `checkNonceFromDB`, both of which would change the Smart-ID/Mobile-ID path.
- **Why a broad catch around parsing is safe:** the block contains only URL parsing and base64 decoding, no database access and no security decision. The deliberate `ResponseStatusException(NOT_FOUND)` for an unknown or expired nonce is thrown later, in the shared check, and passes through unchanged.
- **Logging:** the failure is logged like other verification failures, at `WARN` without a stack trace.

### D3 (finding 3): read the public key file as ISO-8859-1

`CDocCreateCmd.sshRecipient` (cdoc2-cli) and `EncryptionKeyMaterial.fromSshPublicKeyFile` (cdoc2-lib) read the file with ISO-8859-1 instead of US-ASCII.

- **Why ISO-8859-1, not UTF-8:** ISO-8859-1 maps every byte to a character, so reading never fails. UTF-8 would accept `jüri@sülearvuti` saved as UTF-8 but still fail on a comment saved in a legacy encoding. The comment is discarded, so how it decodes does not matter.
- **Why this is safe:** the key type and the base64 key are ASCII. A non-ASCII byte in either of them reaches the parser and is refused there with the readable "not an OpenSSH public key" error, as before for other malformed input.
- **No spec delta:** the existing requirements already describe the intended behaviour.

### D4 (finding 5): split `--ssh-agent` into a flag and `--ssh-agent-socket`

`--ssh-agent` becomes a flag without a value (uses `$SSH_AUTH_SOCK`), and `--ssh-agent-socket <path>` selects another socket. Both are in the existing exclusive group of decryption options, so picocli refuses giving both.

- **Why split, not keep one option with the socket only in the `=` form:** an option with an optional value cannot tell a socket path from a following positional argument. Accepting the value only after `=` needs a custom parser hook, which is more code and easy to get subtly wrong. Two options remove the ambiguity entirely.
- **Breaking change accepted:** `--ssh-agent=<path>` stops working. The feature is unreleased and no script in the workspace uses that form.
- **Not changed:** upstream's `-pw`/`--password` has the same optional-value pattern; it is upstream code and left as it is.

### D5 (finding 8): complete the recipient checks for the flag-on case

On create, with the flag on: a `ruhnu/pk/` recipient is validated as before; any other recipient starting with `ruhnu/` is refused with 400; every other recipient gets the upstream 32-character check. With the flag off only the 32-character check runs, as before.

- **Why refuse every other `ruhnu/` kind:** `ruhnu/` is the fork's namespace and `pk` is its only kind, so anything else there is a typo or a client from a newer version. Refusing it surfaces the mistake at encrypt time instead of storing a share nothing can release.
- **Follow-up:** a new recipient kind (for example from `add-did-recipient`) must add its own branch to this check.

### D6 (finding 9): an empty mpint is a malformed signature

`SshAgentJWSSigner.toJwsSignature` also turns `NumberFormatException` (thrown by `new BigInteger(byte[])` for a zero-length value) into the existing `JOSEException("Malformed ssh-agent ECDSA signature")`. A zero `r` or `s` is invalid ECDSA anyway, and the existing check already refuses non-positive values.

### D7 (finding 10): check the lifetime early, warn where the response is read

- **Non-positive lifetime:** refused in the CLI by a converter for `--share-expiry` only (`PositiveDurationConverter`), so the user gets picocli's usage error before anything is sent. The converter of upstream's `--expiry` for key capsules is left unchanged. `SshPkEncryptionKeyMaterial` refuses it too, with `IllegalArgumentException`, for applications using cdoc2-lib directly.
- **Shortened lifetime:** `Cdoc2KeySharesApiClient.createKeyShare` logs a `WARN` naming the server and the expiry time when the response has `x-expiry-time-adjusted: true` for a request that set `x-expiry-time`. The header lookup ignores case, because the generated client copies headers into a case-sensitive map.
- **Why warn, not return the applied expiry to the caller:** returning it would change return types through the API client, the shares client and the recipient builder, mostly in upstream files. A warning closes the gap (the sender learns that access is shorter) with a few lines.
- **Testing note:** the warning is asserted in cdoc2-lib (`SshPkKeySharesTest`) with a logback `ListAppender`. The CLI's test logger configuration (upstream file) caches the original `System.err`, so CLI tests cannot capture log output; it is left unchanged.

## Risks / Trade-offs

- [The Smart-ID/Mobile-ID path still answers 500 for the same malformed `aud`] → accepted: it needs a certificate from a trusted issuer, and it is upstream behaviour.

- [A misconfigured default is corrected silently for clients] → the startup warning tells the operator, and the admin guide states the rule.
- [The global lifetimes keep the same gap] → accepted to stay aligned with upstream; out of scope here.
