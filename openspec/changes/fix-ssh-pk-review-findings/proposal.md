# Proposal

## Why

The code review of `add-ssh-pk-recipient` ([ruhnu-openspec#1](https://github.com/ruhnu/ruhnu-openspec/issues/1)) found defects in the SSH public key recipient work. This change collects the fixes that alter specified behaviour or need a recorded decision. Findings are added one at a time as they are decided; so far it covers finding 2.

**Finding 2 — default lifetime can exceed the maximum.** The shares server applies the maximum lifetime only to an expiry the client requests with `x-expiry-time`. When the client requests none, the configured default is used as it is. An operator who sets only `cdoc2.ruhnu.pk.max-expiration-duration=P7D` to cap SSH access at 7 days still gets 14-day shares, because the built-in default of 14 days is never compared with the maximum. The CLI sends no `x-expiry-time` unless `--share-expiry` is given, so this is the normal case.

## What Changes

- For `ruhnu/pk/` shares, with the flag `cdoc2.ruhnu.enabled` on, the lifetime given to a share created without a requested expiry is the shorter of the configured default and the configured maximum.
- The server logs a warning when the configured `ruhnu/pk/` default is longer than the maximum.
- `x-expiry-time-adjusted` stays `false` in this case, because the client requested no expiry that could have been adjusted.
- Unchanged: the global lifetimes (`key-share.*-expiration-duration`) used for other recipients and for every share while the flag is off. They keep the upstream behaviour, including a default that is not compared with the maximum.

## Capabilities

### New Capabilities

None.

### Modified Capabilities

- `key-share-expiry`: adds a requirement that the default lifetime of a `ruhnu/pk/` share never exceeds the maximum.

## Impact

- **cdoc2-shares-server** (branch `feat/add-ssh-pk-recipient`): expiry calculation for `ruhnu/pk/` shares, a startup warning, a test, and one line in `admin-guide.md`.
- No API, schema or database change. No client change.
- Operators whose `ruhnu/pk/` default is longer than the maximum get shorter shares than before and a warning in the log. With the built-in values (both 14 days) nothing changes.
