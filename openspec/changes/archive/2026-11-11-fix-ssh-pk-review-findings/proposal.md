# Proposal

## Why

The code review of `add-ssh-pk-recipient` ([ruhnu-openspec#1](https://github.com/ruhnu/ruhnu-openspec/issues/1)) found defects in the SSH public key recipient work. This change collects the fixes that alter specified behaviour or need a recorded decision. Findings are added one at a time as they are decided; so far it covers findings 1, 2, 3, 5, 8, 9 and 10.

**Finding 1 — malformed `aud` gives 500 before authentication.** On the SSH path the token signature is verified with the key in `iss`, which anyone can generate. The `aud` value is parsed afterwards, and a nonce that is not base64url, a query parameter without a name or a duplicated parameter raises an exception that nothing catches. An anonymous caller gets `500` and the server logs a stack trace at `ERROR` for each request. On the Smart-ID/Mobile-ID path the same input needs a trusted certificate first.

**Finding 2 — default lifetime can exceed the maximum.** The shares server applies the maximum lifetime only to an expiry the client requests with `x-expiry-time`. When the client requests none, the configured default is used as it is. An operator who sets only `cdoc2.ruhnu.pk.max-expiration-duration=P7D` to cap SSH access at 7 days still gets 14-day shares, because the built-in default of 14 days is never compared with the maximum. The CLI sends no `x-expiry-time` unless `--share-expiry` is given, so this is the normal case.

**Finding 3 — `.pub` file with a non-ASCII comment cannot be used.** The CLI and cdoc2-lib read the public key file as strict US-ASCII. A key file whose comment has a non-ASCII character, such as `jüri@sülearvuti`, makes `cdoc2 create --ssh-pubkey <file>` fail with a `MalformedInputException` stack trace. The same line given as text works. The `ssh-key-recipient` spec already says that the comment is ignored and that a key given as a file gives the same result as one given as text, so this is a defect against the existing spec.

**Finding 5 — `--ssh-agent` takes the next argument as the socket.** The decrypt option `--ssh-agent` accepts an optional socket path. A positional file name after it, as in `cdoc2 decrypt -f x.cdoc2 --ssh-agent report.pdf`, is read as the socket, so decryption fails with "cannot connect to report.pdf" instead of extracting that file.

**Finding 8 — unvalidated recipients accepted while the flag is on.** With the flag on, the upstream 32-character limit no longer applies and only recipients starting with exactly `ruhnu/pk/` are validated. A typo such as `ruhnu/pK/...`, an unknown kind such as `ruhnu/xyz/...` or a long `etsi/...` string is stored with 201 and the global multi-year lifetime. No authentication path can ever release such a share, so the mistake goes unnoticed at encrypt time.

**Finding 9 — empty signature value from ssh-agent.** An agent that returns an empty `r` or `s` in its ECDSA signature makes the client fail with a raw `NumberFormatException` instead of the readable "Malformed ssh-agent ECDSA signature" error.

**Finding 10 — requested share lifetime unchecked, server's answer ignored.** `--share-expiry` accepts a zero or negative duration, which creates shares that have already expired, so the container can never be decrypted. And when a shares server shortens a requested lifetime to its maximum, it reports so in `x-expiry-time-adjusted`, but the client ignores it: the sender believes the recipient has access for longer than they do.

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

Finding 5:

- **BREAKING (CLI, unreleased):** `--ssh-agent` becomes a flag without a value and always uses `$SSH_AUTH_SOCK`. A new option `--ssh-agent-socket <path>` uses another agent socket. The two are mutually exclusive, like the other decryption options. `--ssh-agent=<path>` no longer works.

Finding 8:

- With the flag on, only `ruhnu/pk/` recipients may be longer than 32 characters. Any other recipient starting with `ruhnu/` is refused with 400, and every other recipient keeps the upstream 32-character limit.
- Unchanged: behaviour with the flag off.

Finding 9:

- An empty value in the agent's signature gives the existing readable "Malformed ssh-agent ECDSA signature" error. No requirement changes.

Finding 10:

- A zero or negative `--share-expiry` is refused before any shares server is contacted. cdoc2-lib refuses it as well.
- When a shares server reports that it shortened the requested lifetime, the client logs a warning naming the server and the expiry time it applied.

## Capabilities

### New Capabilities

None.

### Modified Capabilities

- `key-share-authentication`: adds a requirement that an SSH-path token with an unparseable `aud` is refused with 401, and modifies "Recipient length" so that only `ruhnu/pk/` recipients may exceed 32 characters while the flag is on.
- `key-share-expiry`: adds a requirement that the default lifetime of a `ruhnu/pk/` share never exceeds the maximum.
- `ssh-key-recipient`: adds a requirement that a requested share lifetime is positive and that a shortened lifetime is reported to the user.

## Impact

- **cdoc2-shares-server** (branch `feat/add-ssh-pk-recipient`): `aud` parsing on the SSH verification path; expiry calculation for `ruhnu/pk/` shares, a startup warning; recipient checks on create; `admin-guide.md`; tests.
- **cdoc2-java-ref-impl** (branch `feat/add-ssh-pk-recipient`): reading of the public key file; the `--ssh-agent` and new `--ssh-agent-socket` options; ssh-agent signature parsing; share lifetime check and the shortened-lifetime warning (in cdoc2-client, cdoc2-lib and cdoc2-cli); the CLI README; tests.
- Scripts that use `--ssh-agent=<path>` must switch to `--ssh-agent-socket <path>`. None are known; the feature is unreleased.
- No API, schema or database change.
- Operators whose `ruhnu/pk/` default is longer than the maximum get shorter shares than before and a warning in the log. With the built-in values (both 14 days) nothing changes.
