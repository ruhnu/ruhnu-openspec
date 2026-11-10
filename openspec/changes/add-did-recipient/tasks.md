# Tasks

Prerequisite: in each of the four repos, branch from `feat/add-ssh-pk-recipient`, which holds phase 1 and is not yet merged into `ruhnu`. Rebase if it is merged before this change lands.

## 1. API (cdoc2-openapi)

- [ ] 1.1 In `cdoc2-key-shares-openapi.yaml`, apply the D10 edits:
  - get-share: add `503`;
  - descriptions: `KeyShare.recipient` mentions `ruhnu/did/<did>/hash/<fingerprint>`, and `x-cdoc2-auth-token` mentions the `jwk` header for DID tokens;
  - bump `info.version`.

  Verify: the spec validates, and the module builds and installs locally.

## 2. DID identifier, resolver and verifier (cdoc2-auth)

- [ ] 2.1 Add parsing, building and validation of `ruhnu/did/<did>/hash/<fingerprint>` identifiers (design D1):
  - `did:web` syntax, with IP-literal hosts rejected and `%3A<port>` and `:<segment>` parts allowed;
  - a canonical 43-character base64url fingerprint;
  - at most 255 characters in total;
  - a function giving the fingerprint of an OpenSSH blob.

  Verify: unit tests for:
  - building the identifier for a fixture key, whose fingerprint equals `ssh-keygen -l` on its `.pub` with `+/` replaced by `-_`;
  - rejection of `did:key`, an IP host, an extra path segment, a padded, short or non-canonical fingerprint, and an identifier over 255 characters;
  - acceptance of a port.
- [ ] 2.2 Add P-256 JWK to OpenSSH blob conversion and DID document key extraction (design D3):
  - `authentication` entries as absolute references, `#frag` references and embedded methods;
  - the `controller == DID` check;
  - JWKs with `d` and non-P-256 keys ignored, with unsupported keys reported separately for the CLI.

  Verify:
  - unit tests for every scenario in `specs/did-recipient/spec.md`, requirement "Keys listed by a DID document";
  - a test that the JWK of the fixture key gives the same fingerprint as its OpenSSH blob;
  - a test that a JWK containing `d` is ignored.
- [ ] 2.3 Add the `did:web` resolver (design D2):
  - URL mapping;
  - `java.net.http.HttpClient` with HTTPS only, `Redirect.NEVER`, connect and request timeouts, a 64 KiB body limit, the JDK trust store and `ProxySelector.getDefault()`;
  - a host predicate checked before any request;
  - the `Found`/`NotFound`/`Unavailable` outcomes;
  - an injectable `HttpClient` for tests.

  Verify: tests against a local `HttpsServer` with a test `SSLContext`, covering:
  - every scenario of "Resolving a did:web DID" (path, no path, port, 404 and 410, a redirect whose target receives no request, an `id` mismatch);
  - refused connection, timeout, 5xx, 429 and an untrusted certificate, all `Unavailable`;
  - an oversized body, `NotFound`;
  - a refusing host predicate, with no request sent.
- [ ] 2.4 Add an optional header JWK to `AuthTokenCreator` (design D5). Verify: a token created with it has a protected `jwk` header with only public members and verifies with that key, and the existing `AuthTokenCreator` tests pass unchanged.
- [ ] 2.5 Add the DID auth token verifier (design D6 step 1):
  - `typ`, and `alg == ES256` only;
  - `iss` parses as a DID identifier;
  - `jwk` is present and is a public P-256 key;
  - `fingerprint(jwk)` equals the fingerprint in `iss`;
  - the JWS verifies with `jwk`;
  - there is exactly one `aud` disclosure;
  - it returns the nonce URI and `iss`.

  Verify: unit tests for a valid token, a token signed through `SshAgentJWSSigner` with a test agent, a missing `jwk`, a P-384 or RSA `jwk`, a fingerprint mismatch, a signature by another key, `alg` RS256/none, a tampered payload, and zero or two `aud` elements.
- [ ] 2.6 Bump the cdoc2-auth snapshot version and add a CHANGELOG entry. Verify: `mvn install` succeeds, including checkstyle and spotbugs.

## 3. Shares server (cdoc2-shares-server)

Every task in this group is verified with the flag on, and with at least one test showing that DID behaviour is absent with the flag off. Tests use a local HTTPS DID host stub that counts the requests it receives.

- [ ] 3.1 Switch to the new cdoc2-auth snapshot and the regenerated OpenAPI stubs. Verify: the server builds, and the existing suite passes with the flag off and with `CDOC2_RUHNU_ENABLED=true`.
- [ ] 3.2 Add the `cdoc2.ruhnu.did.allowed-hosts` property (list, default empty), and `cdoc2.ruhnu.did.connect-timeout` and `cdoc2.ruhnu.did.request-timeout` (defaults `PT2S` and `PT5S`), to `RuhnuConfigProperties`. Wire a resolver bean that uses the allowlist as its host predicate (design D7). Verify: binding tests for the defaults and for overrides, following `RuhnuPkExpiryConfigTests`.
- [ ] 3.3 Create validation (design D7): with the flag on, refuse a `ruhnu/did/` recipient that fails identifier validation, or whose host and port are not allowlisted, with 400, storing nothing. Verify: tests for every create scenario under "DID host allowlist", plus "DID recipient refused while off" and "Flag on enables DID recipients".
- [ ] 3.4 Nonce endpoint: accept `ruhnu/did/` shares on the session-less path, without resolving the DID. Verify:
  - a test issues a nonce for a DID share while the stub is stopped, and the stub receives no request;
  - an `etsi/` share without headers still gets 400, and a DID share with the flag off gets 400.
- [ ] 3.5 Get-share DID path (design D6). Generalise the phase 1 prefix checks into a recipient and token kind check (`pk`, `did`, or neither), and add the allowlist re-check, resolution, and the 404/503 mapping. Verify:
  - tests for every scenario under "Share release for DID shares", including that the stub receives no request for every 401 or 404 before resolution, and the retry after an outage;
  - tests for "DID token for an SSH public key share of the same key", "Smart-ID token without session headers while on" and "Host removed from the list after the share was created";
  - the existing `pk` and SID/MID tests pass.
- [ ] 3.6 Audit log: write the full `ruhnu/did/` recipient id for DID releases (design D9). Verify: a test asserts the line for a DID release, and no line for a DID-check refusal (404 and 503).
- [ ] 3.7 Expiry: confirm that `ruhnu/did/` shares use the global lifetimes (design D8). Verify:
  - with `cdoc2.ruhnu.pk.*` set to non-default values, a DID share without a requested expiry gets the global default;
  - a request beyond the global maximum is cut to it, with `x-expiry-time-adjusted: true`.
- [ ] 3.8 Document DID recipients in `admin-guide.md`, with commented entries in `application-local.properties`:
  - the allowlist and timeouts;
  - outbound HTTPS to DID hosts;
  - the meaning of 503;
  - revocation by removing the key from the DID document;
  - the long-lifetime and key-rotation caveats from the design's Risks.

  Verify: the documented properties and defaults match the code.

## 4. Client library (cdoc2-java-ref-impl/cdoc2-lib)

- [ ] 4.1 Switch to the regenerated key-shares client, and make the HTTP status of failed get-share requests available to callers (design D10). Nonce failures already carry it: phase 1 reads `ApiException.getCode()`. Verify: a WireMock test that returns 503 for get-share reads the status.
- [ ] 4.2 Accept the `ruhnu/did/` kind in `KeySharesRecipient`, validated as in D1. Unknown `ruhnu/` kinds are still refused. Verify:
  - serialisation round-trip tests with type `RUHNU`;
  - a container with a malformed `ruhnu/did/` recipient fails to decrypt, with WireMock receiving no request;
  - the `ruhnu/xyz/` test still passes.
- [ ] 4.3 DID encryption (design D4): resolve a DID into one encryption key material per listed key plus the list of skipped keys, and fail on `NotFound`, on `Unavailable`, or when no supported key is listed. `RecipientFactory` builds a `RUHNU` recipient with the DID id and passes the requested lifetime through. Verify, with the DID stub and WireMock shares servers:
  - two keys give two recipients;
  - an Ed25519 key is reported as skipped;
  - with no supported key, not found or unavailable, WireMock receives no create request;
  - an invalid DID (`did:key`, an IP host, or an identifier over 255 characters) fails before WireMock receives any create request;
  - a requested lifetime sends `x-expiry-time`.
- [ ] 4.4 DID decryption key material and the `KekTools` path (design D11):
  - nonce without session headers;
  - token with `iss = recipient_id` and the agent key as header `jwk`, signed through ssh-agent;
  - get-share 404 and 503 mapped to the messages in the spec.

  Verify: a WireMock test with a test agent checks the token's `jwk`, `iss` and fingerprint, and checks both error messages.

## 5. CLI (cdoc2-java-ref-impl/cdoc2-cli)

- [ ] 5.1 Add a repeatable option to `create` for encrypting to a DID. It prints each pinned `recipient_id` and `SHA256:` fingerprint, warns about skipped keys, and reports a 400 on create as "refused by shares server <url>; the DID host may not be allowed there". Verify: CLI tests against the DID stub and mocked shares servers, for every scenario under "Encrypt to a DID", and a test that a 400 on create gives the "may not be allowed there" message.
- [ ] 5.2 Make `decrypt --ssh-agent` match DID recipients by fingerprint (design D11), and make `info` show "SSH key via DID" with the DID and `SHA256:` fingerprint. Verify: CLI tests for:
  - a container with only a DID recipient;
  - a mixed container with `pk` key A and `did` key B, the agent holding only B;
  - the no-match error naming the DID and fingerprint;
  - the 404 and 503 messages;
  - no agent available, and agent refusal, with a DID-only container;
  - `info` output.
- [ ] 5.3 Document DID encryption and decryption in the CLI README, with an example `did.json`. Verify: the documented commands run as written in the demo setup from group 6.

## 6. End-to-end demo

- [ ] 6.1 Extend the phase 1 demo:
  - add a local HTTPS DID host serving `did.json` for `did:web:<host>:p:alice`, which lists the test agent key under `authentication`;
  - add that host to the allowlist of both shares servers;
  - make the CLI and both servers trust the demo CA through a truststore.

  Verify: the script brings the stack up from a clean checkout, and the CLI's resolver fetches the document.
- [ ] 6.2 Run the full flow: encrypt a secret to the DID, decrypt it, and check that both servers logged the release with the `ruhnu/did/` recipient. Verify: a scripted run succeeds and the audit lines are present.
- [ ] 6.3 Revocation: remove the key from `did.json`, confirm decryption fails at once with the "no longer listed" message, then restore the key and confirm decryption works again. Verify: a scripted run shows failure, then success.
- [ ] 6.4 Outage: stop the DID host, confirm decryption fails with the "could not reach the DID host" message, then restart it and confirm decryption works. Verify: a scripted run shows failure, then success.
- [ ] 6.5 Compromised host: add a second P-256 key, not held by the test agent, to `did.json`. Confirm that an agent holding only that key cannot decrypt the existing container. Confirm also that a token for an existing share, signed by that key with its own DID identifier, is refused with 404. Verify: a scripted run shows both refusals.
- [ ] 6.6 Regression check: the phase 1 `ruhnu/pk/` demo flow still passes, and the shares-server suite passes with the flag off and with `CDOC2_RUHNU_ENABLED=true`. Verify: both runs pass.

## Workflow follow-up

- Update the CDOC2-ssh handoff notes with what the phase 2 demo shows.
- Archive the change once the demo works and the changes are reviewed.
- Follow-ups recorded in the design:
  - a committed recipients file of pinned `ruhnu/did/` lines, with a lookup helper (D4);
  - a short positive DID cache if load requires it;
  - `publicKeyMultibase`/Multikey keys;
  - Ed25519;
  - a privacy review of DIDs in the audit log (D9);
  - rate limiting get-share for `ruhnu/did/` (design Risks).
- After archive, the main `key-share-authentication` spec's "Audit log of released shares" requirement has a description over 500 characters, which `openspec validate --strict` flags. Split it in a separate change if wanted.
