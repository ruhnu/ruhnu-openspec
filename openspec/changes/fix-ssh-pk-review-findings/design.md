# Design

## Context

See proposal.md for the findings. This document records the decision taken for each one. It grows as findings are added.

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

## Risks / Trade-offs

- [The Smart-ID/Mobile-ID path still answers 500 for the same malformed `aud`] → accepted: it needs a certificate from a trusted issuer, and it is upstream behaviour.

- [A misconfigured default is corrected silently for clients] → the startup warning tells the operator, and the admin guide states the rule.
- [The global lifetimes keep the same gap] → accepted to stay aligned with upstream; out of scope here.
