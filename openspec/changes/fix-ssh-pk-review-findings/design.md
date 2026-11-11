# Design

## Context

See proposal.md for the findings. This document records the decision taken for each one. It grows as findings are added.

In cdoc2-shares-server, `KeyShareApiService.getExpiryTime` picks a default and a maximum duration: the `cdoc2.ruhnu.pk.*` pair for a `ruhnu/pk/` recipient when the flag is on, otherwise the global `key-share.*` pair. The maximum is compared only with a client-requested `x-expiry-time`. The comparison logic is inherited from upstream.

## Goals / Non-Goals

**Goals:**
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

## Risks / Trade-offs

- [A misconfigured default is corrected silently for clients] → the startup warning tells the operator, and the admin guide states the rule.
- [The global lifetimes keep the same gap] → accepted to stay aligned with upstream; out of scope here.
