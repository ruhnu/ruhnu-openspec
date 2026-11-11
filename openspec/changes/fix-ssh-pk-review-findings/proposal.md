# Proposal

## Why

The code review of `add-ssh-pk-recipient` ([ruhnu-openspec#1](https://github.com/ruhnu/ruhnu-openspec/issues/1)) found defects in the SSH public key recipient work. This change collects the fixes that alter specified behaviour or need a recorded decision. Findings are added one at a time as they are decided; so far it covers findings 1, 2 and 3.

**Finding 1 — malformed `aud` gives 500 before authentication.** On the SSH path the token signature is verified with the key in `iss`, which anyone can generate. The `aud` value is parsed afterwards, and a nonce that is not base64url, a query parameter without a name or a duplicated parameter raises an exception that nothing catches. An anonymous caller gets `500` and the server logs a stack trace at `ERROR` for each request. On the Smart-ID/Mobile-ID path the same input needs a trusted certificate first.

**Finding 2 — default lifetime can exceed the maximum.** The shares server applies the maximum lifetime only to an expiry the client requests with `x-expiry-time`. When the client requests none, the configured default is used as it is. An operator who sets only `cdoc2.ruhnu.pk.max-expiration-duration=P7D` to cap SSH access at 7 days still gets 14-day shares, because the built-in default of 14 days is never compared with the maximum. The CLI sends no `x-expiry-time` unless `--share-expiry` is given, so this is the normal case.

**Finding 3 — `.pub` file with a non-ASCII comment cannot be used.** The CLI and cdoc2-lib read the public key file as strict US-ASCII. A key file whose comment has a non-ASCII character, such as `jüri@sülearvuti`, makes `cdoc2 create --ssh-pubkey <file>` fail with a `MalformedInputException` stack trace. The same line given as text works. The `ssh-key-recipient` spec already says that the comment is ignored and that a key given as a file gives the same result as one given as text, so this is a defect against the existing spec.

## What Changes

Finding 1:

- An SSH-path auth token whose `aud` cannot be parsed is refused with `401`, like any other token that fails verification. No stack trace is logged.
- Unchanged: a well-formed nonce that is unknown or expired still gives `404`, as for existing recipients. The Smart-ID/Mobile-ID path keeps its upstream behaviour.

Finding 2:

- For `ruhnu/pk/` shares, with the flag `cdoc2.ruhnu.enabled` on, the lifetime given to a share created without a requested expiry is the shorter of the configured default and the configured maximum.
- The server logs a warning when the configured `ruhnu/pk/` default is longer than the maximum.
- `x-expiry-time-adjusted` stays `false` in this case, because the client requested no expiry that could have been adjusted.
- Unchanged: the global lifetimes (`key-share.*-expiration-duration`) used for other recipients and for every share while the flag is off. They keep the upstream behaviour, including a default that is not compared with the maximum.

Finding 3:

- A `.pub` file is accepted whatever bytes its comment contains. No requirement changes; the fix makes the client meet the existing `ssh-key-recipient` requirements.

## Capabilities

### New Capabilities

None.

### Modified Capabilities

- `key-share-authentication`: adds a requirement that an SSH-path token with an unparseable `aud` is refused with 401.
- `key-share-expiry`: adds a requirement that the default lifetime of a `ruhnu/pk/` share never exceeds the maximum.

## Impact

- **cdoc2-shares-server** (branch `feat/add-ssh-pk-recipient`): `aud` parsing on the SSH verification path; expiry calculation for `ruhnu/pk/` shares, a startup warning and one line in `admin-guide.md`; tests for both.
- **cdoc2-java-ref-impl** (branch `feat/add-ssh-pk-recipient`): reading of the public key file in cdoc2-cli and cdoc2-lib; tests for both.
- No API, schema or database change.
- Operators whose `ruhnu/pk/` default is longer than the maximum get shorter shares than before and a warning in the log. With the built-in values (both 14 days) nothing changes.
