# Tasks

Prerequisite: in each of the four repos, branch from `feat/add-ssh-pk-recipient`, which holds phase 1 and is not yet merged into `ruhnu`. Rebase if it is merged before this change lands.

## 1. API (cdoc2-openapi)

- [x] 1.1 In `cdoc2-key-shares-openapi.yaml`, apply the D10 edits:
  - get-share: add `503`;
  - descriptions: `KeyShare.recipient` mentions `ruhnu/did/<did>/hash/<fingerprint>`, and `x-cdoc2-auth-token` mentions the `jwk` header for DID tokens;
  - bump `info.version`.

  Verify: the spec validates, and the module builds and installs locally.

## 2. DID identifier, resolver and verifier (cdoc2-auth)

- [x] 2.1 Add parsing, building and validation of `ruhnu/did/<did>/hash/<fingerprint>` identifiers (design D1):
  - `did:web` syntax, with IP-literal hosts rejected and `%3A<port>` and `:<segment>` parts allowed;
  - a canonical 43-character base64url fingerprint;
  - at most 255 characters in total;
  - a function giving the fingerprint of an OpenSSH blob.

  Verify: unit tests for:
  - building the identifier for a fixture key, whose fingerprint equals `ssh-keygen -l` on its `.pub` with `+/` replaced by `-_`;
  - rejection of `did:key`, an IP host, an extra path segment, a padded, short or non-canonical fingerprint, and an identifier over 255 characters;
  - acceptance of a port.
- [x] 2.2 Add P-256 JWK to OpenSSH blob conversion and DID document key extraction (design D3):
  - `authentication` entries as absolute references, `#frag` references and embedded methods;
  - the `controller == DID` check;
  - JWKs with `d` and non-P-256 keys ignored, with unsupported keys reported separately for the CLI.

  Verify:
  - unit tests for every scenario in `specs/did-recipient/spec.md`, requirement "Keys listed by a DID document";
  - a test that the JWK of the fixture key gives the same fingerprint as its OpenSSH blob;
  - a test that a JWK containing `d` is ignored.
- [x] 2.3 Add the `did:web` resolver (design D2):
  - URL mapping;
  - `java.net.http.HttpClient` with HTTPS only, `Redirect.NEVER`, a connect timeout and an overall deadline covering the body (D2), a 64 KiB body limit, the JDK trust store and `ProxySelector.getDefault()`;
  - a host predicate checked before any request;
  - the `Found`/`NotFound`/`Unavailable` outcomes;
  - an injectable `HttpClient` for tests.

  Verify: tests against a local `HttpsServer` with a test `SSLContext`, covering:
  - every scenario of "Resolving a did:web DID" (path, no path, port, 404 and 410, a redirect whose target receives no request, an `id` mismatch);
  - refused connection, timeout, a body that arrives too slowly, 5xx, 429 and an untrusted certificate, all `Unavailable`;
  - an oversized body, `NotFound`;
  - a refusing host predicate, with no request sent.
- [x] 2.4 Add an optional header JWK to `AuthTokenCreator` (design D5). Verify: a token created with it has a protected `jwk` header with only public members and verifies with that key, and the existing `AuthTokenCreator` tests pass unchanged.
- [x] 2.5 Add the DID auth token verifier (design D6 step 1):
  - `typ`, and `alg == ES256` only;
  - `iss` parses as a DID identifier;
  - `jwk` is present and is a public P-256 key;
  - `fingerprint(jwk)` equals the fingerprint in `iss`;
  - the JWS verifies with `jwk`;
  - there is exactly one `aud` disclosure;
  - it returns the nonce URI and `iss`.

  Verify: unit tests for a valid token, a missing `jwk`, a P-384 or RSA `jwk`, a fingerprint mismatch, a signature by another key, `alg` RS256/none, a tampered payload, and zero or two `aud` elements. A token signed through ssh-agent is checked in 4.4 instead, because `SshAgentJWSSigner` lives in cdoc2-lib.
- [x] 2.6 Bump the cdoc2-auth snapshot version and add a CHANGELOG entry. Verify: `mvn install` succeeds, including checkstyle and spotbugs.

## 3. Shares server (cdoc2-shares-server)

Every task in this group is verified with the flag on, and with at least one test showing that DID behaviour is absent with the flag off. Tests use a local HTTPS DID host stub that counts the requests it receives.

- [x] 3.1 Switch to the new cdoc2-auth snapshot and the regenerated OpenAPI stubs. Verify: the server builds, and the existing suite passes with the flag off and with `CDOC2_RUHNU_ENABLED=true`.
- [x] 3.2 Add the `cdoc2.ruhnu.did.allowed-hosts` property (list, default empty), and `cdoc2.ruhnu.did.connect-timeout` and `cdoc2.ruhnu.did.request-timeout` (defaults `PT2S`, and `PT3S` as the overall deadline, design D7), to `RuhnuConfigProperties`. Wire a resolver bean that uses the allowlist as its host predicate (design D7). Verify: binding tests for the defaults and for overrides, following `RuhnuPkExpiryConfigTests`.
- [x] 3.3 Create validation (design D7): with the flag on, refuse a `ruhnu/did/` recipient that fails identifier validation, or whose host and port are not allowlisted, with 400, storing nothing. Verify: tests for every create scenario under "DID host allowlist", plus "DID recipient refused while off" and "Flag on enables DID recipients".
- [x] 3.4 Nonce endpoint: accept `ruhnu/did/` shares on the session-less path, without resolving the DID. Verify:
  - a test issues a nonce for a DID share while the stub is stopped, and the stub receives no request;
  - an `etsi/` share without headers still gets 400, and a DID share with the flag off gets 400.
- [x] 3.5 Get-share DID path (design D6). Generalise the phase 1 prefix checks into a recipient and token kind check (`pk`, `did`, or neither), and add the allowlist re-check, resolution, and the 404/503 mapping. Verify:
  - tests for every scenario under "Share release for DID shares", including that the stub receives no request for every 401 or 404 before resolution, and the retry after an outage;
  - tests for "DID token for an SSH public key share of the same key", "Smart-ID token without session headers while on" and "Host removed from the list after the share was created";
  - the existing `pk` and SID/MID tests pass.
- [x] 3.6 Audit log: write the full `ruhnu/did/` recipient id for DID releases (design D9). Verify: a test asserts the line for a DID release, and no line for a DID-check refusal (404 and 503).
- [x] 3.7 Expiry: confirm that `ruhnu/did/` shares use the global lifetimes (design D8). Verify:
  - with `cdoc2.ruhnu.pk.*` set to non-default values, a DID share without a requested expiry gets the global default;
  - a request beyond the global maximum is cut to it, with `x-expiry-time-adjusted: true`.
- [x] 3.8 Document DID recipients in `admin-guide.md`, with commented entries in `application-local.properties`:
  - the allowlist and timeouts;
  - outbound HTTPS to DID hosts;
  - the meaning of 503;
  - revocation by removing the key from the DID document;
  - the long-lifetime and key-rotation caveats from the design's Risks.

  Verify: the documented properties and defaults match the code.

## 4. Client library (cdoc2-java-ref-impl/cdoc2-lib)

- [x] 4.1 Switch to the regenerated key-shares client, and make the HTTP status of failed get-share requests available to callers (design D10). Nonce failures already carry it: phase 1 reads `ApiException.getCode()`. Verify: a WireMock test that returns 503 for get-share reads the status.
- [x] 4.2 Accept the `ruhnu/did/` kind in `KeySharesRecipient`, validated as in D1. Unknown `ruhnu/` kinds are still refused. Verify:
  - serialisation round-trip tests with type `RUHNU`;
  - a container with a malformed `ruhnu/did/` recipient fails to decrypt, with WireMock receiving no request;
  - the `ruhnu/xyz/` test still passes.
- [x] 4.3 DID encryption (design D4): resolve a DID into one encryption key material per listed key plus the list of skipped keys, and fail on `NotFound`, on `Unavailable`, or when no supported key is listed. `RecipientFactory` builds a `RUHNU` recipient with the DID id and passes the requested lifetime through. Verify, with the DID stub and WireMock shares servers:
  - two keys give two recipients;
  - an Ed25519 key is reported as skipped;
  - with no supported key, not found or unavailable, WireMock receives no create request;
  - an invalid DID (`did:key`, an IP host, or an identifier over 255 characters) fails before WireMock receives any create request;
  - a requested lifetime sends `x-expiry-time`.
- [x] 4.4 DID decryption key material and the `KekTools` path (design D11):
  - nonce without session headers;
  - token with `iss = recipient_id` and the agent key as header `jwk`, signed through ssh-agent;
  - get-share 404 and 503 mapped to the messages in the spec.

  Verify: a WireMock test with a test agent checks the token's `jwk`, `iss` and fingerprint, verifies the agent-signed token with cdoc2-auth's DID token verifier (moved here from 2.5), and checks both error messages.

## 5. CLI (cdoc2-java-ref-impl/cdoc2-cli)

- [x] 5.1 Add a repeatable option to `create` for encrypting to a DID. It prints each pinned `recipient_id` and `SHA256:` fingerprint, warns about skipped keys, and reports a 400 on create as "refused by shares server <url>; the DID host may not be allowed there". Verify: CLI tests against the DID stub and mocked shares servers, for every scenario under "Encrypt to a DID", and a test that a 400 on create gives the "may not be allowed there" message.
- [x] 5.2 Make `decrypt --ssh-agent` match DID recipients by fingerprint (design D11), and make `info` show "SSH key via DID" with the DID and `SHA256:` fingerprint. Verify: CLI tests for:
  - a container with only a DID recipient;
  - a mixed container with `pk` key A and `did` key B, the agent holding only B;
  - the no-match error naming the DID and fingerprint;
  - the 404 and 503 messages;
  - no agent available, and agent refusal, with a DID-only container;
  - `info` output.
- [x] 5.3 Document DID encryption and decryption in the CLI README, with an example `did.json`. Verify: the documented commands run as written in the demo setup from group 6.

## 6. End-to-end demo

- [x] 6.1 Extend the phase 1 demo in the `ruhnu-demo` repo (branch `feat/add-did-recipient`): `up.sh` also starts an nginx DID host, and the new `did.sh` runs the DID steps. A first version lived in `cdoc2-java-ref-impl/test/ruhnu-demo/` before the `ruhnu-demo` repo was found, and was moved there. The demo:
  - starts two shares servers (flag on) and a local HTTPS DID host serving `did.json` for `did:web:localhost%3A8444:p:alice`, which lists the test agent key under `authentication`;
  - adds that host to the allowlist of both shares servers;
  - makes the CLI and both servers trust the DID host's certificate through a JVM truststore.

  Verify: the script brings the stack up from a clean checkout, and the CLI's resolver fetches the document.
- [x] 6.2 Run the full flow: encrypt a secret to the DID, decrypt it, and check that both servers logged the release with the `ruhnu/did/` recipient. Verify: a scripted run succeeds and the audit lines are present.
- [x] 6.3 Revocation: remove the key from `did.json`, confirm decryption fails at once with the "no longer listed" message, then restore the key and confirm decryption works again. Verify: a scripted run shows failure, then success.
- [x] 6.4 Outage: stop the DID host, confirm decryption fails with the "could not reach the DID host" message, then restart it and confirm decryption works. Verify: a scripted run shows failure, then success.
- [x] 6.5 Compromised host: add a second P-256 key, not held by the test agent, to `did.json`. Confirm that an agent holding only that key cannot decrypt the existing container. Confirm also that a token for an existing share, signed by that key with its own DID identifier, is refused with 404. Verify: a scripted run shows both refusals.
- [x] 6.6 Regression check: the phase 1 `ruhnu/pk/` flow (`demo.sh` with expiry, `revoke.sh`) passes on the same stack, and the shares-server suite passes with the flag off and with `CDOC2_RUHNU_ENABLED=true`. Verify: both runs pass.

## 7. Review fixes (issue #4 review, after phase 1 merged)

- [x] 7.1 Specs: the phase 1 review fixes refuse every `ruhnu/` kind but `ruhnu/pk/` ("Recipient length"); the delta now accepts `ruhnu/did/` there, and widens "Malformed audience refused on the SSH path" to the DID path. Verify: `openspec validate --strict`.
- [x] 7.2 cdoc2-auth: `DidWeb` refuses `.` and `..` path segments, also percent-encoded, and segments that decode to `/`, `\`, `?`, `#` or `;`, so `did:web:example.org:p:alice:..:mallory` cannot name Mallory's document. Verify: `DidWebTest`.
- [x] 7.3 cdoc2-shares-server: `spring.jpa.open-in-view=false`, so a get-share does not hold a database connection while the DID document is fetched. Verify: a concurrent request is not blocked by a slow DID host with a small pool.
- [x] 7.4 cdoc2-shares-server tests: DID tests for a malformed `aud` (401, no DID fetch); `shouldReturn404WhenHostRemovedFromList` checks that no DID request was made in a way that can fail. CHANGELOG entry for DID recipients.
- [x] 7.5 cdoc2-java-ref-impl: a nonce 404 (share expired or deleted) gives a readable error instead of a stack trace; the create 400 hint no longer blames the DID host outright; a malformed `ruhnu/` recipient in a container is reported, not silently skipped; `SshDidEncryptionKeyMaterial` refuses a zero or negative lifetime; README lifetimes for DID shares (3 years by default, at most 5); CHANGELOG entry. Verify: lib and CLI tests.
- [x] 7.6 cdoc2-java-ref-impl tests: RUHNU `ruhnu/did/` round-trip through the container header; the "no request" and "no upload" assertions for malformed recipients and DID failures can fail.
- [ ] 7.7 Before merging cdoc2-shares-server#6 and cdoc2-java-ref-impl#2: point `.github/ruhnu-deps.env` back at `ruhnu` once cdoc2-openapi#2 and cdoc2-auth#2 are merged.
- [x] 7.8 cdoc2-auth: a `ruhnu/pk/` or `ruhnu/did/` token whose disclosed `aud` element is not a string, such as `[null]`, is refused with a `VerificationException` (401), not a `NullPointerException` (500). Verify: `DidAuthTokenVerifierTest`, `SshPkAuthTokenVerifierTest`, both failing without the fix.
- [x] 7.9 cdoc2-auth: text from a DID document (document `id`, verification method ids, `controller`, `crv`, JWK parse errors) is shortened and has its control and format characters escaped before it goes into a resolution reason or a skipped entry, so a DID document cannot forge shares-server log lines or hide the CLI's "Pinned" lines with ANSI escapes. Verify: `DidDocumentKeysTest`, `DidWebResolverTest`.
- [x] 7.10 ruhnu-demo: `demo.sh` and `revoke.sh` match the nonce 404 message of 7.5 ("refused share ID ...") and fail with the CLI output if it is missing. Verify: `build.sh`, `up.sh`, `demo.sh`, `revoke.sh`, `did.sh` pass.

## 8. Wildcard allowlist entries (user decision 2026-11-11, design D7)

- [ ] 8.1 cdoc2-shares-server: `DidHostAllowlist` accepts `*.<domain>[:port]` entries, matching exactly one DNS label, with `<domain>` of at least two labels; `*`, `*.org` and `a.*.org` are refused at startup. Verify: tests for every scenario under "Wildcard DID host entries", including create tests through the API.
- [ ] 8.2 Docs: the admin guide describes wildcard entries, with the warning against wildcard DNS services such as `nip.io`; the `RUHNU_DID_ALLOWED_HOSTS` comment in ruhnu-ops' `.env.example` no longer says wildcards stop the server. Verify: both documents match the scenarios.

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
- Pre-existing, seen while running group 3: `KeyShareIntegrationTest.shouldCleanUpExpiredKeyShareNonce`, which every shares-server integration test class inherits, fails when its run crosses a five-minute boundary. The scheduled clean-up (`key-share-nonce.expired.clean-up.cron=0 0/5 * * * ?`) deletes the day-old nonce the test saves before the test counts it. Not caused by this change.
- Local build note: the shares-server tests need `ee.cyber.cdoc2:cdoc2-client:2.3.2` (and its parent `cdoc2:3.5.2`), which no reachable Maven repository has; here it was built from `cdoc2-java-ref-impl` `origin/ruhnu`.
