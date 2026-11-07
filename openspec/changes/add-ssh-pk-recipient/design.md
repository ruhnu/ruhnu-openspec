# Design

## Context

See `proposal.md` for motivation. This section covers only what the code does today, read from local checkouts on 2026-11-07.

- **The shares server ties both endpoints to the Smart-ID/Mobile-ID stack.**
  - `POST /key-shares/{id}/nonce` validates a session token, signed by cdoc2-auth-server, before it looks up the share.
  - `GET /key-shares/{id}` validates the session token, then the auth token, and only then looks up the share. `AuthTokenVerifier.verify` requires an X.509 certificate chained to the trust store, an `etsi/` `iss`, and either Smart-ID RPv3 parameters or an RP-server RFC 9421 countersignature (`KeyShareApiService`, `ValidateAuthToken`).
- **Authorisation is a string comparison** between the verified identity from the token and `key_material_share.recipient` (`VARCHAR(32)`, OpenAPI `maxLength: 32`).
- **The OpenAPI document enforces header presence.** In `cdoc2-key-shares-openapi.yaml`, `x-cdoc2-auth-x5c`, `x-cdoc2-session-token` and `x-cdoc2-session-x5c` are `required: true`. The generated server layer rejects requests missing them with 400 before any service code runs.
- **Expiry is one global pair**, `key-share.default-expiration-duration` and `key-share.max-expiration-duration`. `getExpiryTime` ignores the recipient. Expired rows are removed by a cron job (`ExpiredKeyShareCleanUpJob`). The prerequisite change `fix-expired-share-release` makes share lookup in both endpoints treat expired rows as not found (404), always on.
- **Client side**: `KekTools.signShareAccessTokens` switches on SID/MID and passes an `IdentityJWSSigner` (a Nimbus `JWSSigner`) to `SidMidAuthTokenCreator`. That class also needs a `SessionToken`, both for nonce requests and for share requests. `RpClient` is required up front.
- **Versions:** cdoc2-auth is Java 17 with nimbus-jose-jwt 10.9.1. cdoc2-shares-server is Java 21 and pins cdoc2-auth-token 0.8.3.
- **Agent signing is proven.** `cdoc2-ssh/jws_probe` shows a P-256 key in ssh-agent producing a valid `ES256` JWS once the signature is re-encoded.

## Goals / Non-Goals

**Goals:**
- The phase 1 demo works end to end: `cdoc2-cli` encrypts to an SSH key, decrypts through ssh-agent, the shares server releases the shares, and they expire on the configured schedule.
- With `cdoc2.ruhnu.enabled=false`, the shares server returns the same response status as upstream for every request.
- With the flag on, the Smart-ID/Mobile-ID flow keeps working through the existing code in the existing order, with no difference for those recipients.
- The new code sits behind boundaries that can be upstreamed without rework: a recipient type, a verifier in cdoc2-auth, and a flag-gated dispatch in the shares server.

**Non-Goals:**
- Refactoring the SID/MID token flow beyond what is needed to make the session token and RP client optional for SSH recipients.
- Production hardening: rate limiting the now-unauthenticated nonce endpoint for SSH shares, a revoke-now API, or structured audit storage (the audit trail is log lines).
- A feature flag in cdoc2-auth, cdoc2-lib or cdoc2-cli. Their changes only add code and do nothing unless called.

## Decisions

### D1. Identifier is the OpenSSH key blob, not a fingerprint

`recipient_id = "private/pk/" + base64(OpenSSH wire blob)`.

- The OpenSSH wire format is canonical, so string comparison on the server is correct. It carries the key type and is what `ssh-add -L` and `.pub` files contain.
- Putting the full key in the identifier lets the server verify without the client sending the key, and leaves no trust anchor to manage.

Parsers MUST strip the `private/pk/` prefix rather than split on `/`, because standard base64 contains `/`.

Alternatives considered:
- **SHA-256 fingerprint:** shorter, but the client then has to send the key, and the server can't verify anything from the share record alone.
- **SPKI DER:** EC points can be encoded compressed or uncompressed, so it isn't canonical, and `ssh-keygen` can't export Ed25519 as SPKI for the later phase.
- **JWK thumbprint:** a hash, with the same drawback as a fingerprint.

### D2. Genuine JWS (ES256) signed through ssh-agent, not SSHSIG

The client builds the usual SD-JWT. It sends the JWS signing input (`base64url(header).base64url(payload)`) to the agent in an `SSH2_AGENTC_SIGN_REQUEST` with `flags=0`. The agent returns `ecdsa-sha2-nistp256` as `string r, string s` (mpints), which the client converts to fixed-width 32-byte `r‖s`.

- The existing SD-JWT structure, `aud` disclosure and nonce logic are reused unchanged on both sides.
- The server verifies with a stock Nimbus `ECDSAVerifier`.

SSHSIG was rejected because it signs a different byte string (`"SSHSIG"` + namespace + `H(msg)`). The server would then need a second token format.

The agent client is a small custom implementation over Java Unix domain sockets (Java 16+), covering two messages: request identities, and sign request. It adds no dependency.

### D3. Dispatch on the token issuer (get-share) and on session headers (nonce)

With the flag on:

```
GET /key-shares/{id}
  iss = unverified read of the token's "iss" (routing only)
  1. verify:
       iss starts with "private/pk/":  SshPkAuthTokenVerifier          -> verified identity = iss
       otherwise:                      existing SID/MID path, existing order
                                       (header checks, session token, auth token)
     both include the existing aud / shareId / nonce checks
     (wrong audience -> 401, unknown or expired nonce -> 404)
  2. look up share (missing or expired = not found,
     per fix-expired-share-release)                                 -> 404
  3. verified identity == stored recipient, else                   -> 404

POST /key-shares/{id}/nonce
  either session header present:  existing path, unchanged
                                  (a missing second header -> 400, as upstream)
  both session headers absent:    look up share (missing or expired -> 404)
                                  recipient starts with "private/pk/" -> issue nonce
                                  otherwise                           -> 400
```

This is the existing order (verify, look up, compare), with the SSH verifier as an alternative in step 1. Each kind of failure therefore gets the same status code as upstream.

Reading `iss` before verification is safe because the security decision is still the final comparison between the **verified** identity and the stored recipient, which already exists:
- Forcing the SSH path for an `etsi/` share yields the verified identity `private/pk/<attacker key>`, which never equals the stored recipient.
- Forcing the SID/MID path for a `private/pk/` share needs a trusted X.509 certificate and yields an `etsi/` identity, which also never matches.
- The SSH verifier takes the key from the same `iss` it returns, so a token cannot verify under one key while claiming another.

Smart-ID/Mobile-ID requests therefore keep upstream's order and status codes: authenticate first, then look up.

Alternatives considered:
- **Look up the share first and dispatch on the stored recipient.** Equally safe, but it reorders the SID/MID path (404 before 401 for unknown ids) for no security gain. This was the first draft, replaced for that reason.
- **A separate `/ssh/key-shares/{id}` path.** It duplicates the endpoints and the aud/nonce checks, and the identity comparison is still needed.

### D4. New verifier in cdoc2-auth, beside `AuthTokenVerifier`

`SshPkAuthTokenVerifier.verify(token)`:
1. Parse the SD-JWT. Require `typ` as for existing tokens and `alg == ES256`.
2. Read `iss`. Require the `private/pk/` prefix. Decode the blob. Require key type `ecdsa-sha2-nistp256` and curve `nistp256`, and build an EC public key from the 65-byte uncompressed point.
3. Verify the JWS with Nimbus `ECDSAVerifier`.
4. Decode the disclosures and return the single `aud` element together with `iss`, in the same response shape (nonce URI and identifier) that `ValidateAuthToken` already consumes.

The shares server then compares `iss` with the stored recipient, and runs `checkTokenAudience` and `checkNonceFromDB` unchanged.

`TokenVerificationResponse` currently carries an `EtsiIdentifier`. The PoC may widen it to a plain identifier string, or add a sibling response type. Pick whichever touches less code; upstreaming can tidy it later.

The shares server takes this cdoc2-auth change as a snapshot dependency until it is released.

### D5. Client token creation without session token or RP client

Add an SSH path in `KekTools` that:
- requests nonces without session headers,
- signs with `SshAgentJWSSigner` (an `IdentityJWSSigner` whose identity is the `recipient_id` and which has no certificate),
- fetches shares with only `x-cdoc2-auth-token`.

Whether this is a new token-creator class or `SidMidAuthTokenCreator` with an optional session token is an implementation choice. Prefer a separate class if making it optional spreads null checks through the SID/MID path.

`RpClient` and `AuthClient` checks move into the SID/MID branches only.

### D6. Per-recipient-type expiry (flag on)

Add a `pk` lifetime pair beside the existing global pair, under the fork prefix:

```
key-share.default-expiration-duration=P1095D       # existing, upstream; all shares while the flag is off
key-share.max-expiration-duration=P1825D
cdoc2.ruhnu.pk.default-expiration-duration=P14D    # new; private/pk/ shares while the flag is on
cdoc2.ruhnu.pk.max-expiration-duration=P14D
```

`getExpiryTime` takes the recipient and selects the pair by prefix when the flag is on. Phase 2 adds a `cdoc2.ruhnu.did.*` pair in the same shape.

Shares stop working exactly at their expiry because of the prerequisite `fix-expired-share-release`, which is always on and not part of this change. The 14-day lifetime is therefore a hard bound, not "14 days plus one cleanup interval".

### D7. Schema and API widening

**OpenAPI (`cdoc2-key-shares-openapi.yaml`)** – an API description cannot be feature-flagged, because the generated layer enforces it before service code runs:

| Location | Change |
| --- | --- |
| `components.schemas.KeyShare.recipient` | `maxLength` 32 → 255, description mentions `private/pk/<blob>` |
| `getKeyShareByShareId` → `x-cdoc2-auth-x5c` | `required: true` → `false` |
| `components.parameters.SessionTokenHeader`, `SessionX5cHeader` (shared by get-share and nonce) | `required: true` → `false` |
| `info.version` | bump |

The descriptions state that these headers remain mandatory for `etsi/` recipients. Unchanged:
- `x-cdoc2-auth-token`: stays required, and the SSH token reuses it.
- The Smart-ID RPv3 and RP countersignature headers: already optional.
- `x-expiry-time`: already exists.
- Response codes: 400, 401 and 404 are already documented.

All changes only widen the API, so existing generated clients keep working.

A second, fork-only API file was rejected: it would mean two sets of generated stubs and clients to keep in sync, in exchange for three edits.

**Liquibase:** a changeset widens `key_material_share.recipient` to `VARCHAR(255)`. It always runs; a wider column is harmless.

**FlatBuffers:** a new `KeyShareRecipientType` value, `SSH_PK`, appended after `SID_MID` so existing ordinals are unchanged.

The OpenAPI module generates both the server stubs and the Java client, so the generated code in both repos must be rebuilt.

### D8. Feature flag `cdoc2.ruhnu.enabled` (shares server only)

The rule: **flag off = upstream.** With the flag off, every request goes through the original code path in the original order. The server code also repeats the checks the OpenAPI document no longer enforces:
- a missing `x-cdoc2-session-token` or `x-cdoc2-session-x5c` (nonce, get-share), or `x-cdoc2-auth-x5c` (get-share) → 400. This check runs on **every request that takes the SID/MID path**, whatever the flag says, so Smart-ID/Mobile-ID clients get upstream status codes in both modes. With the flag on, only `private/pk/` requests (D3) skip it.
- a recipient longer than 32 characters on create → 400 (flag off only).

So upstream clients see identical status codes. A `private/pk/` create is refused by the length check, exactly as upstream would refuse it.

Behind the flag: D3 dispatch, the SSH verifier call, D6 per-type expiry, and the release audit log. The expiry refusal from `fix-expired-share-release` is not gated; it is treated as part of upstream.

Not behind the flag:
- The Liquibase changeset and the OpenAPI document, which can't be gated (D7).
- `cdoc2-auth`, `cdoc2-lib` and `cdoc2-cli`, whose changes only add code.

Implementation keeps the gate in as few places as possible, ideally one branch per endpoint plus one in `getExpiryTime`. The upstream diff is then easy to read, and removing the flag after upstreaming is mechanical.

Alternative considered: separate flags per feature (SSH recipients, per-type expiry, audit log). Rejected for the PoC, because one switch matches the "fork vs upstream" boundary, and the features can be split when they are upstreamed one by one.

## Risks / Trade-offs

- **[Two server modes]** → Every shares-server behaviour has to be tested with the flag off (upstream behaviour unchanged) and on. The existing test suite must pass unchanged with the flag off.
- **[Header checks move from generated code to hand-written code]** → If the flag-off checks miss a case, upstream behaviour changes silently. Mitigation: tests for each previously required header on each endpoint, with the flag off.
- **[Nonce endpoint is unauthenticated for SSH shares]** → A nonce is useless without the private key and expires in 300 s. The risk is table growth or DoS. Accepted for the PoC; rate limiting is listed as follow-up.
- **[`POST /key-shares` is unauthenticated (existing)]** → Anyone can create `private/pk/` shares when the flag is on. This grants no access to other shares; the risk is storage only, which already exists for `etsi/` recipients.
- **[Forwarded agents]** → Root on a remote host can ask a forwarded agent for signatures. `ssh-add -h` does not help (it refuses all non-user-auth signing, per `openssh_probe`). Document `ssh-add -c` or FIDO keys, and prefer decrypting locally.
- **[Revocation is expiry only]** → Within the 14-day window, removing access requires deleting or expiring the recipient's rows on at least one shares server (N_OF_N, so one server suffices). Out-of-band SQL is acceptable for the demo.
- **[Key rotation]** → A rotated key loses access to existing containers. The 14-day lifetime and the re-encrypt cadence bound this.
- **[cdoc2-auth snapshot dependency]** → The shares server builds against an unreleased cdoc2-auth. Pin a local snapshot for the demo, and release before any shared deployment.

## Migration Plan

- The flag defaults to off, so deploying the fork changes no behaviour until an operator sets `cdoc2.ruhnu.enabled=true`. The demo environment sets it.
- The Liquibase changeset widening `recipient` is backward compatible, and existing rows are unaffected.
- The new `cdoc2.ruhnu.pk.*` properties have defaults, so no config change is needed.
- Existing containers and clients are unaffected, because the new enum value is appended and the OpenAPI changes only widen the API.
- Rollback: switch the flag off, or redeploy the previous server. With the flag off, `private/pk/` shares are unreachable because their requests lack session headers. With the previous server, they also cannot be read. Both only affect SSH shares.

## Open Questions

- The exact CLI option names for the SSH recipient and for agent-based decryption. They follow existing `cdoc2-cli` conventions and can be chosen during implementation.
- The exact log format of the release audit line, for example whether it adds the caller IP. Any format that meets the spec is fine.
