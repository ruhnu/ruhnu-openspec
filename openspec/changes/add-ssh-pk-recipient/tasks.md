# Tasks

Prerequisite: `fix-expired-share-release` is applied and archived before this change is started.

## 1. API and schema (cdoc2-openapi, cdoc2-java-ref-impl/cdoc2-schema)

- [ ] 1.1 In `cdoc2-key-shares-openapi.yaml`, apply the D7 edits: recipient `maxLength` 32 → 255; `x-cdoc2-auth-x5c`, `SessionTokenHeader` and `SessionX5cHeader` change to `required: false`, with descriptions stating they remain mandatory for `etsi/` recipients; bump `info.version`. Verify: the spec validates, and the module builds and installs locally.
- [ ] 1.2 Append `SSH_PK` to `KeyShareRecipientType` in `recipients.fbs`, after `SID_MID`, and update the `recipient_id` comment to document `private/pk/<base64 OpenSSH blob>`. Verify: `cdoc2-schema` regenerates and builds, and existing schema tests pass.

## 2. SSH public key parsing and verifier (cdoc2-auth)

- [ ] 2.1 Add a parser from a `private/pk/` identifier and from an OpenSSH public key line to a P-256 `ECPublicKey`. It strips the prefix rather than splitting on `/`, and rejects non-`ecdsa-sha2-nistp256` keys, a wrong curve name and malformed base64 or blobs with distinct errors. Verify: unit tests with a key from `ssh-keygen -t ecdsa -b 256`, an Ed25519 key, an RSA key and truncated or garbage blobs.
- [ ] 2.2 Add `SshPkAuthTokenVerifier` (design D4): `typ` check, `alg == ES256` only, `iss` prefix check, key from `iss`, Nimbus `ECDSAVerifier`, single `aud` disclosure. It returns the nonce URI and the `iss` identifier, with `TokenVerificationResponse` widened or a sibling type added. Verify: unit tests for a valid token, wrong key, `iss` without the prefix, `alg` RS256/EdDSA/none, a non-P-256 key in `iss`, a tampered payload, and zero or two `aud` elements.
- [ ] 2.3 Add a test that verifies a token whose signature comes from the real ssh-agent signature format (SSH `r`,`s` mpints re-encoded to `r‖s`), using a fixture produced by `cdoc2-ssh/jws_probe`. Verify: the test passes.
- [ ] 2.4 Bump the cdoc2-auth version to a snapshot and add a CHANGELOG entry. Verify: `mvn install` succeeds, including checkstyle and spotbugs.

## 3. Shares server (cdoc2-shares-server)

Every task in this group is verified with the flag on, and with at least one test showing the behaviour is absent with the flag off.

- [ ] 3.1 Add a Liquibase changeset widening `key_material_share.recipient` to `VARCHAR(255)`, and update the server to use the cdoc2-auth snapshot and the regenerated OpenAPI stubs. Verify: the server starts against a fresh DB and an existing DB, and existing tests pass.
- [ ] 3.2 Add the feature flag `cdoc2.ruhnu.enabled` (default `false`) and the in-code checks from D8: a missing session or auth-certificate header on the SID/MID path → 400 in both modes; a recipient longer than 32 characters on create → 400 with the flag off. Verify: with the flag off the existing test suite passes unchanged, and new tests cover each previously required header on each endpoint and the long-recipient 400, plus the SID/MID header checks with the flag on.
- [ ] 3.3 Per-recipient-type expiry (design D6): add `cdoc2.ruhnu.pk.default-expiration-duration` and `cdoc2.ruhnu.pk.max-expiration-duration` defaulting to `P14D`, and make `getExpiryTime` select the pair by recipient prefix when the flag is on. Verify: with the flag on, tests for a `pk` share with no requested expiry (14 d), a shorter request (accepted, adjusted=false), a longer request (cut, adjusted=true), configured overrides, and an `etsi/` share keeping the global values. With the flag off, the global values apply to everything.
- [ ] 3.4 Nonce endpoint dispatch (design D3): if either session header is present, take the existing path unchanged. If both are absent and the flag is on, look up the share and issue a nonce only for a `private/pk/` recipient; otherwise 400. Verify: tests for a `pk` nonce without headers, an `etsi/` share without headers (400), an unknown share without headers (404), a `pk` share with only one session header (400, existing path), and a `pk` share without headers with the flag off (400). Existing SID/MID nonce tests pass.
- [ ] 3.5 Get-share dispatch (design D3): with the flag on and the token's `iss` starting with `private/pk/`, call `SshPkAuthTokenVerifier`; otherwise take the existing SID/MID path in its existing order. Both paths verify the token, including the existing audience and nonce checks, then look up the share and require the verified identity to equal the stored recipient. Verify: tests for every scenario in `specs/key-share-authentication/spec.md` under "Authentication path chosen by token issuer" and "Share release for SSH public key shares", including SSH tokens for `etsi/` shares and Smart-ID tokens for `pk` shares being refused. Existing SID/MID tests pass.
- [ ] 3.6 With the flag on, log one audit line on every successful share release (shareId, recipient, time), for all recipient types. Verify: a test asserts the line on release and no line on refusal with the flag on, and no line with the flag off.
- [ ] 3.7 Document `cdoc2.ruhnu.enabled`, the `cdoc2.ruhnu.pk.*` properties, the flag-off-equals-upstream rule and the SSH recipient behaviour in the server README or config docs. Verify: the documented properties and defaults match the code and `application-local.properties`.

## 4. Client library (cdoc2-java-ref-impl/cdoc2-lib)

- [ ] 4.1 Point the library at the regenerated key-shares OpenAPI client and make the session, certificate and countersignature arguments optional in `KeySharesClient.getKeyShare` and in nonce creation. Verify: the library builds and existing key-shares client tests pass.
- [ ] 4.2 Add a minimal ssh-agent client over a Unix domain socket: list identities (msg 11/12) and sign request (msg 13/14, flags=0), with distinct errors for no `$SSH_AUTH_SOCK`, connection failure, agent refusal (`SSH_AGENT_FAILURE`) and key not present. Verify: an integration test against a real `ssh-agent` started by the test with a generated P-256 key, plus tests for each error case.
- [ ] 4.3 Add `SshAgentJWSSigner` (`IdentityJWSSigner`, ES256 only), which converts the agent's `string r, string s` signature to 64-byte `r‖s`. Verify: a test signs through a test-started agent and verifies with Nimbus `ECDSAVerifier` against the public key.
- [ ] 4.4 Recipient support for `SSH_PK`: encryption key material from an OpenSSH public key line or `.pub` file, serialisation and deserialisation with `recipient_id` `private/pk/<blob>`, and decryption key material identifying the agent key. Reject unsupported key types before any upload. Verify: round-trip serialisation tests, plus tests that the same key with different comments gives the same identifier and that Ed25519/RSA are rejected.
- [ ] 4.5 Decrypt path in `KekTools` for `SSH_PK` (design D5): nonce without session headers, then an agent-signed SD-JWT with `iss = recipient_id`, then get-share with only the auth token. Move the `RpClient`/`AuthClient` requirements into the SID/MID branches. Verify: a test against mocked shares servers (WireMock, as in the existing tests) checks the request headers and token claims; existing SID/MID decrypt tests pass.
- [ ] 4.6 Pass the requested share lifetime (`x-expiry-time`) through on upload for SSH recipients when the caller gives one, and omit it otherwise. Verify: a WireMock test asserts the header is present when given and absent when not.

## 5. CLI (cdoc2-java-ref-impl/cdoc2-cli)

- [ ] 5.1 Add an option to `create` that takes an SSH recipient as a `.pub` path or an OpenSSH public key line (repeatable), and an option for the requested share lifetime. Verify: CLI tests create a container with an SSH recipient against mocked shares servers, and `info` shows the recipient type and identifier.
- [ ] 5.2 Add decryption through ssh-agent to `decrypt` for containers with an SSH recipient. Errors for no agent, key not in agent and agent refusal must be readable. Verify: CLI tests for success against mocked servers with a test-started agent, and for each error case.
- [ ] 5.3 Document both options in the CLI README with an example using `ssh-add -L`. Verify: the documented commands run as written in the demo setup from group 6.

## 6. End-to-end demo

- [ ] 6.1 Script a local demo environment: two shares servers (N_OF_N) with `cdoc2.ruhnu.enabled=true` and `cdoc2.ruhnu.pk.*` set short (e.g. `PT5M`), a test ssh-agent holding a P-256 key, and `cdoc2-cli` configured for both servers. Verify: the script brings the stack up from a clean checkout.
- [ ] 6.2 Run the full flow: encrypt a secret to the agent key, decrypt it successfully, check that both servers logged the release, wait past expiry and confirm decryption fails with 404 before the cleanup job runs. Verify: a scripted run finishes with the expected success, then failure, and the audit lines are present.
- [ ] 6.3 Show revocation by deleting the recipient's rows on one server (SQL) and confirming decryption fails immediately while the other server still holds its share. Verify: a scripted run shows the failure and names the refusing server.
- [ ] 6.4 Confirm Smart-ID/Mobile-ID decryption against the modified shares server still works, with the flag off and on. Verify: the existing SID/MID end-to-end or integration tests pass unchanged in both modes.

## Workflow follow-up

- Update `cdoc2-ssh/HANDOFF.md` with what the demo shows and what changed from the original pitch.
- Archive the change once the demo works and the changes are reviewed.
- Start the phase 2 change (`private/did/<did>/<jwk-thumbprint>`) from the decisions recorded in the explore session.
