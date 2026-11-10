# Design

## Context

See `proposal.md` for motivation. This section covers only what the phase 1 code does today, read from the `feat/add-ssh-pk-recipient` branches on 2026-11-10. Those branches are not merged into `ruhnu` yet.

- **The client accepts only the `ruhnu/pk/` kind.**
  - The `KeySharesRecipient` constructor rejects any `RUHNU` recipient whose id does not start with `ruhnu/pk/`.
  - `CDocDecryptionHelper` (`--ssh-agent`) collects `ruhnu/pk/` recipients, turns each agent identity into a `ruhnu/pk/` id, and picks the first match.
- **Token creation:**
  - `AuthTokenCreator.Builder.withIssuer(String)` already sets any `iss`.
  - `AuthTokenCreator.sign()` builds the JWS header with only `alg` and `typ`, so a `jwk` header cannot be added yet.
  - `SshAgentJWSSigner` signs `ES256` with an agent key, identified by its blob.
- **Token verification:** `SshPkAuthTokenVerifier` (cdoc2-auth) takes the key from `iss`. `SshPublicKeyParser` already builds OpenSSH blobs and `SHA256:` fingerprints for P-256 keys.
- **Shares server (`KeyShareApiService`):**
  - `isSshPkRecipient()` gates four places: create validation, the session-less nonce path (`createAndSaveSshPkNonce`), the audit log format and `getExpiryTime()`.
  - `ValidateAuthToken.isSshPkToken()` makes an unverified read of `iss` and selects the get-share path.
  - `getExpiryTime()` uses the `cdoc2.ruhnu.pk.*` pair only for `ruhnu/pk/`, so every other recipient already gets the global pair.
- **Nonces are not consumed by a successful check.** `checkNonceFromDB` only checks existence and age (`cdoc2.nonce.expiration-seconds`, default 300). A token that got a transient error can be sent again until its nonce expires.
- **HTTP clients:**
  - The shares server already has Spring `RestClient` (`RpServerClient`).
  - cdoc2-lib uses a Jersey client generated from the OpenAPI document.
  - cdoc2-auth is Java 17 and has Nimbus (JWK parsing, JSON utilities), but no HTTP client.
- **Schema and API limits:** the recipient column and the OpenAPI `maxLength` are 255. The get-share operation documents 200, 400, 401 and 404.

## Goals / Non-Goals

**Goals:**
- The phase 2 demo works end to end:
  - encrypt to a `did:web` DID;
  - decrypt through ssh-agent;
  - remove the key from `did.json` and see decryption refused with 404;
  - stop the DID host and see 503.
- A compromised DID host cannot cause a share to be released to a key that was not pinned when the container was encrypted.
- The client and the shares server read DID documents with the same code, so they agree on which keys a document lists.
- With the flag off, the shares server behaves exactly as with phase 1. With the flag on but no allowlisted hosts, `ruhnu/did/` shares cannot be created, and `ruhnu/pk/` and Smart-ID/Mobile-ID behaviour is unchanged.

**Non-Goals:**
- A general DID resolver or support for DID resolution metadata. Only the subset of `did:web` that the spec describes is implemented.
- Production hardening of outbound fetches beyond the allowlist, redirect, timeout and size limits, for example DNS rebinding protection or egress proxies.
- Changing how `ruhnu/pk/` recipients behave.

## Decisions

### D1. Identifier `ruhnu/did/<did>/hash/<fingerprint>`

`<fingerprint>` is `base64url_nopad(SHA-256(OpenSSH wire blob))`. It is the same value `ssh-keygen -l` prints after `SHA256:`, with `+/` replaced by `-_`.

Parsing:
1. Strip `ruhnu/did/`.
2. Split the rest on `/`. It must give exactly three parts: `<did>`, `hash`, `<fingerprint>`.

This is unambiguous, because DID syntax has no `/` and base64url has none either. Standard base64 would break it, which is why the fingerprint is base64url.

Validation:
- `<did>` must be `did:web:` followed by `<host>[%3A<port>]` and zero or more `:<segment>`. The host must be a DNS name, not an IP literal, as the `did:web` method requires. Segments use DID `idchar`s.
- `<fingerprint>` must be 43 characters that decode to 32 bytes and re-encode to the same string, so it is canonical.
- The whole identifier is at most 255 characters.

DIDs are compared as exact strings, and the document's `id` must equal the DID (D3). A sender who types a DID in a different case therefore fails when encrypting, not later.

Alternatives considered:
- **The key blob, as in `ruhnu/pk/`:** the server would then never need the DID document's key bytes. But the fingerprint is enough for the server to check the document (D6), and the blob makes the identifier about 110 characters longer.
- **JWK thumbprint (RFC 7638)**, the form planned in phase 1's follow-ups: closer to DID conventions, but users and operators cannot compare it with `ssh-add -l` or `ssh-keygen -l`. The SSH fingerprint is computed directly from the agent blob on the client.
- **The label `sha256` instead of `hash`:** `hash` is what the user chose. There is a single algorithm, and a future one would get its own label.

### D2. One `did:web` resolver in cdoc2-auth, used by client and server

cdoc2-auth gets three pieces, written so that client and server classify every response the same way:
- a pure part: parse the DID, map it to a URL, and extract keys from the document (D3);
- a fetcher built on `java.net.http.HttpClient`:
  - HTTPS only, `Redirect.NEVER`;
  - connect and request timeouts, defaulting to 2 s and 5 s; the shares server takes its values from configuration (D7);
  - a 64 KiB body limit;
  - the JDK default trust store;
  - `ProxySelector.getDefault()`, so the standard `https.proxyHost` settings work;
- a result type with three outcomes:
  - `Found(keys)`;
  - `NotFound(reason)`: HTTP 404, 410, any other 4xx or 3xx, bad JSON, `id` mismatch or an oversized body;
  - `Unavailable(reason)`: connect or TLS error, timeout, 5xx or 429.

The fetcher takes a host predicate, so the server can enforce its allowlist (D7) inside the resolver. The client allows every host. Tests inject an `HttpClient` with a test `SSLContext`, so they can run against a local `HttpsServer`.

Alternatives considered:
- **A DID resolver library or the Universal Resolver:** an extra dependency, or an extra trusted service, for one method that comes down to a single GET.
- **Spring `RestClient` on the server and a separate client fetcher:** the two would then decide "not found" and "unavailable" differently.

### D3. Which keys a document lists

Entries in `authentication` are handled as follows:
- A string entry is a reference. A relative `#frag` is resolved against the DID, then looked up by `id` in `verificationMethod`.
- An object entry is an embedded verification method.

A method counts if `controller` equals the DID and `publicKeyJwk` is `kty: EC`, `crv: P-256` with `x` and `y`. A JWK that contains `d` is ignored, because a private key in a public document is a misconfiguration. `type` is not checked: `JsonWebKey2020` and `JsonWebKey` both carry `publicKeyJwk`.

Each listed key is converted to the OpenSSH blob `string "ecdsa-sha2-nistp256", string "nistp256", string 0x04||x||y` and hashed, which gives its fingerprint.

Rationale:
- In DID Core, `authentication` is the relationship for proving control as the subject, which is what downloading a share means. `assertionMethod` and others are for other purposes.
- `controller == DID` excludes delegated keys, which would need a second resolution.
- `publicKeyMultibase` (Multikey) is out of scope: P-256 Multikeys use compressed points, so supporting them means decompressing points for little gain in the PoC.

### D4. Encryption: resolve the DID and pin all keys (user decision)

The library resolves the DID before building the container, and returns one DID encryption key material per listed key. The CLI then:
- prints each `recipient_id` with its `SHA256:` fingerprint;
- warns about skipped unsupported keys;
- fails before any upload if the DID was not found, its host was unavailable, or it lists no supported key.

Resolving before the build keeps network I/O out of `RecipientFactory`, and ensures a failure happens before anything is uploaded.

The DID host is trusted at that moment. A DID host compromised at encryption time can pin its own key. The printed fingerprints and `cdoc2 info` make that visible to the sender and to reviewers of the container.

The client has no allowlist. If a shares server's allowlist refuses the DID host, the create request fails with 400, and the CLI reports it as "refused by shares server <url>; the DID host may not be allowed there".

Alternatives considered (raised in explore, not chosen for now):
- A committed recipients file of pinned `ruhnu/did/` lines, with a separate lookup helper.
- Explicit DID plus fingerprint arguments.

Either would confine trust in the DID host to a reviewed step. Both are listed as follow-ups.

### D5. Auth token: `iss` is the recipient id, the key travels in the `jwk` header

The client:
1. signs one SD-JWT for all shares through ssh-agent, as in phase 1;
2. sets `iss` to the `recipient_id` (`withIssuer`);
3. adds the public JWK of the agent key (`kty`, `crv`, `x`, `y`) as the protected `jwk` header parameter (RFC 7515 §4.1.3).

`AuthTokenCreator` gains an optional header JWK. Existing callers are unaffected.

Rationale:
- `iss == stored recipient` stays the security decision, exactly as in phase 1's D3. The DID path is only a different way of verifying the same claim.
- The key is protected by the signature, and Nimbus parses it.

Alternatives considered:
- **`iss` = the DID and `kid` = a DID URL, with the server taking the key from the document:** this breaks the `iss == recipient` invariant. Verification would also need the DID fetch before any crypto, so unauthenticated callers could trigger fetches.
- **The OpenSSH blob in a custom claim:** non-standard, and a JWK is easy to convert to a blob.

### D6. get-share order on the DID path

```
GET /key-shares/{id}   flag on, unverified iss starts with "ruhnu/did/"
  1. DID token verifier (cdoc2-auth), no network:
       typ, alg == ES256, iss parses as a ruhnu/did identifier  else 401
       header jwk is a public P-256 EC key                     else 401
       fingerprint(jwk) == fingerprint in iss                  else 401
       JWS verifies with jwk                                   else 401
     existing aud / shareId / nonce checks           (401, or 404 for nonce)
  2. look up unexpired share                                   else 404
  3. iss == stored recipient                                   else 404
  4. DID host in cdoc2.ruhnu.did.allowed-hosts                 else 404
  5. resolve DID, no cache
       Unavailable                                             --> 503
       NotFound                                                --> 404
       Found, but no listed key has the fingerprint            --> 404
  6. release + audit log
```

- Because `fingerprint(jwk) == fingerprint(iss) == fingerprint(listed key)`, the listed key is the signing key, by the collision resistance of SHA-256. No direct JWK comparison is needed.
- **The DID is resolved last.** An outbound request happens only after the caller has proven possession of the pinned key for an existing share.
- **404 for every persistent refusal**, to match how phase 1 treats a recipient mismatch. 503 only for transient failures, so clients know to retry. Since nonces are not consumed (Context), the same token can be retried within the nonce lifetime.
- Each shares server resolves on its own. With `N_OF_N`, a refusal from any one of them blocks decryption, which is the intended kill-switch behaviour.
- **Dispatch generalises phase 1's helpers.** The prefix checks `isSshPkToken` and `isSshPkRecipient` become kind checks (`pk`, `did`, or neither). The nonce path accepts both kinds and never resolves. `getExpiryTime()` keeps matching only `pk` (D8). The gates stay in the places phase 1's D8 lists.

### D7. DID host allowlist, against SSRF

`POST /key-shares` is unauthenticated. Without an allowlist, anyone could create a share for `ruhnu/did/did:web:<any host>:<path>/hash/<own key>`, download it with their own key, and make the server GET `https://<any host>/<path>/did.json`. Ordering (D6) does not prevent this, because the attacker owns both the share and the key.

`cdoc2.ruhnu.did.allowed-hosts` is a list of `host` or `host:port` entries:
- Host comparison is case-insensitive.
- The port must match exactly, and a missing port means 443.
- It is checked when a share is created (400) and again before every fetch (404). Removing a host therefore takes effect for existing shares.
- It is empty by default, so DID recipients stay off until an operator names the hosts they accept.

Further limits:
- The identifier parser already rejects IP-literal hosts (D1).
- The fetcher does not follow redirects, caps the body size, and times out.
- Timeouts are configurable: `cdoc2.ruhnu.did.connect-timeout` defaults to `PT2S`, and `cdoc2.ruhnu.did.request-timeout` to `PT5S`. They stay well below typical client read timeouts.

### D8. Lifetimes: global pair (user decision)

`ruhnu/did/` shares use `key-share.default-expiration-duration` and `key-share.max-expiration-duration`. `getExpiryTime` already behaves that way, so the change only adds tests. The `cdoc2.ruhnu.did.*` lifetime pair that phase 1's D6 anticipated is not added.

### D9. Audit log records the full `ruhnu/did/` identifier (assumption)

The audit log exists to show who received a share. A fingerprint alone does not say whose key it is, and the DID does. Unlike the `etsi/` personal identification codes that upstream removed from its logs, a DID is an identifier the organisation assigns and publishes, like a user name.

This was not discussed explicitly. If DIDs in a deployment embed personal identification codes, log `ruhnu/did:SHA256:<fingerprint>` instead. Only the audit log format and its spec scenario would change.

### D10. OpenAPI

`cdoc2-key-shares-openapi.yaml`:
- **get-share:** add `'503': Service Unavailable. The DID document of a "ruhnu/did/" recipient could not be retrieved; retry later.`
- **Descriptions:**
  - `KeyShare.recipient` mentions `ruhnu/did/<did>/hash/<fingerprint>`.
  - `x-cdoc2-auth-token` notes that DID tokens carry the key in the `jwk` header.
- **Version:** bump `info.version`.

This is a widening change only. The OpenAPI module generates both the server stubs and the Java client, so both must be rebuilt. To show the 503 message, the client needs the HTTP status from `getKeyShare`. If `ExtApiException` does not carry the status today, it must be made to.

### D11. Client decryption

- `KeySharesRecipient` accepts `ruhnu/did/` ids that pass the D1 validation. Other `ruhnu/` kinds are still rejected.
- **`--ssh-agent` matching:**
  - It collects `RUHNU` recipients of both kinds.
  - For each agent identity it computes the `ruhnu/pk/` id and the fingerprint.
  - It matches `pk` recipients by id and `did` recipients by fingerprint.
  - It uses the first matching recipient in container order.
  - If nothing matches, the error lists the `pk` fingerprints and the `did` DID and fingerprint pairs.
- **DID decryption key material** holds the `recipient_id` and the agent key blob. Its `KekTools` path follows phase 1's, plus the header JWK. It maps get-share 404 and 503 to the messages in the spec.
- **`cdoc2 info`** shows "SSH key via DID", with the DID and the `SHA256:` fingerprint.

### D12. Feature flag

No new gate is needed. With the flag off:
- a `ruhnu/did/` create fails the 32-character upstream check, since every DID identifier is at least about 70 characters;
- a `ruhnu/did/` token takes the Smart-ID/Mobile-ID path and fails the session-header check with 400.

Tests cover both cases.

## Risks / Trade-offs

- **[The DID host is trusted at encryption time]** → The CLI prints the pinned fingerprints and `cdoc2 info` shows them. A reviewed recipients file is the follow-up.
- **[The DID host is a hard dependency for decryption]** → Intended. 503 tells the client to retry, and the token stays usable until the nonce expires. One unreachable host blocks only its own DIDs.
- **[No cache: one fetch per share per server per decryption]** → Fine for the PoC. A short positive cache is a follow-up if load becomes a problem, at the cost of slower revocation.
- **[Allowlisted hosts can be probed at any path, at the caller's request rate]** → Anyone can create a share whose DID points to any path on an allowlisted host, and trigger a GET of `<path>/did.json` there. Nonces are not consumed, so one valid token can be replayed until its nonce expires, and each replay causes one fetch: the shares server forwards load to the DID host one-for-one, bounded only by the request rate it accepts. Accepted for the PoC, because the operator chose those hosts. Rate limiting get-share for `ruhnu/did/` is a follow-up, next to phase 1's nonce rate limiting. DNS rebinding of an allowlisted name is the operator's concern (Non-Goals).
- **[Long lifetimes depend on revocation discipline]** → With the global pair, shares last up to 5 years. Removing access depends on someone removing the key from the DID document. Stating the maximum in the admin guide keeps that visible.
- **[Key rotation versus revocation]** → A rotated-away key must stay listed until containers pinned to it are re-encrypted, or they become unreadable. This is the price of a host that can withdraw access but not grant it. The admin guide states it.
- **[Unauthenticated nonce endpoint for DID shares]** → As in phase 1. The nonce endpoint never resolves DIDs, so it adds no outbound traffic.
- **[The audit log contains DIDs]** → See D9. A privacy review should confirm the DID format.
- **[Built on unmerged phase 1 branches and a cdoc2-auth snapshot]** → Branch from `feat/add-ssh-pk-recipient` in each repo, or rebase once phase 1 is merged into `ruhnu`.

## Migration Plan

- **Opt-in:** the flag defaults to off and `cdoc2.ruhnu.did.allowed-hosts` defaults to empty, so DID recipients stay off until an operator sets both. A server already running phase 1 with the flag on sees one difference before any host is allowlisted: creating a `ruhnu/did/` share now fails with 400, where phase 1 stored it as an opaque recipient that no request could fetch. The demo turns the flag on and allowlists the demo DID host.
- **No schema changes:** there is no DB or FlatBuffers change, and the identifier fits the existing 255-character column.
- **OpenAPI:** the document only gains a response code, so existing clients keep working.
- **Rollback:**
  - Remove a host from the allowlist: its existing shares answer 404 and nothing is fetched.
  - Or switch the flag off: all `ruhnu/` shares become unreachable, as in phase 1.
  - Or redeploy the phase 1 server: it refuses `ruhnu/did/` tokens and creates.

## Open Questions

- The exact CLI option name for encrypting to a DID, for example `--did`, following phase 1's `--ssh-pubkey` style. To be chosen during implementation.
