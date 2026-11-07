# Proposal

## Why

Today only Smart-ID and Mobile-ID users can open a CDOC2 key-shares recipient. Engineers who keep encrypted secrets in Git need a recipient they can open with the SSH key already loaded in their ssh-agent, with access that ends on a short, server-enforced schedule. ssh-agent can sign but not decrypt, which is the limitation the key-shares scheme was built for, so the scheme carries over. What has to be replaced is the authentication leg, which is currently tied to the Smart-ID/Mobile-ID, RP-server and auth-server stack.

This is phase 1 of the CDOC2-ssh work. Its goal is a working end-to-end demo with boundaries shaped for later upstreaming. DID recipients (`private/did/...`) are phase 2 and a separate change.

## What Changes

- **Every shares-server behaviour change below sits behind the feature flag `cdoc2.ruhnu.enabled`, default OFF.** With the flag OFF, the shares server behaves like upstream. Two differences cannot be gated: the wider DB column, and the relaxed OpenAPI document. The client library, CLI and cdoc2-auth have no flag, because their changes only add code.
- New key-shares recipient type for an SSH public key. `recipient_id` is `private/pk/<base64 OpenSSH public key blob>`, so the container names the key itself and no trust anchor or registry exists.
- Phase 1 supports only ECDSA P-256 (`ecdsa-sha2-nistp256`) keys, signed as JWS `ES256`. Ed25519 and RSA are deferred.
- `cdoc2-cli` can encrypt to an SSH public key and decrypt by signing the per-share auth token (SD-JWT) through ssh-agent (`$SSH_AUTH_SOCK`). The signature is a genuine JWS, not SSHSIG.
- When the flag is enabled, the shares server accepts SSH public key recipients:
  - The get-share endpoint chooses the verification path from the auth token's `iss` prefix. For `private/pk/` it verifies the token against the public key in `iss`, with no session token, X.509 certificate, Smart-ID RPv3 parameters or RP countersignature. Every path still ends with the existing check that the verified identity equals the share's stored recipient, followed by the audience, shareId and nonce checks.
  - The nonce endpoint issues nonces without a session token for shares whose stored recipient starts with `private/pk/`.
  - Smart-ID/Mobile-ID requests run through the existing code in the existing order.
- When the flag is enabled, share expiry is configurable per recipient type, as a default and a maximum. For `private/pk/` both default to 14 days. Clients may request a shorter lifetime with `x-expiry-time`; longer requests are cut to the maximum.
- When the flag is enabled, the shares server refuses expired shares with 404, as if the cleanup job had already removed them. This applies to all recipient types. Upstream releases expired shares until the cleanup job runs.
- When the flag is enabled, the shares server logs every successful share release (audit trail).
- Recipient length limits are widened (DB column, OpenAPI `maxLength`), because a P-256 `private/pk/` identifier is about 151 characters.
- **Not in scope:** Ed25519/RSA keys, DID recipients, a revoke-now API (revocation is expiry, or deleting rows out of band), nonce rate limiting, SSH certificates/CA.

## Capabilities

### New Capabilities

- `ssh-key-recipient`: the SSH public key recipient type. Covers the identifier format, the supported key types, and encrypting and decrypting with it in the client library and CLI, including signing through ssh-agent.
- `key-share-authentication`: how the shares server authenticates nonce and share requests: the feature flag, choosing the path per recipient type, SSH public key verification, and the audit log of released shares.
- `key-share-expiry`: server-side share lifetime per recipient type, covering the configurable default and maximum, client-requested shorter lifetimes, and refusal of expired shares.

### Modified Capabilities

None. The store has no specs yet.

## Impact

- **cdoc2-java-ref-impl**
  - `cdoc2-schema` `recipients.fbs`: new `KeyShareRecipientType` value.
  - `cdoc2-lib`: recipient (de)serialisation, encryption/decryption key material, a new ssh-agent JWS signer, and a token-creation path that needs no session token or RP client.
  - `cdoc2-cli`: options to encrypt to and decrypt with an SSH key.
- **cdoc2-auth**: a new verifier for SD-JWT auth tokens signed by an SSH public key named in `iss`. No X.509, no trust store.
- **cdoc2-shares-server**
  - The `cdoc2.ruhnu.*` configuration: the feature flag and the `pk` lifetimes.
  - Token-based dispatch in get-share, and header-based dispatch in nonce.
  - The expiry check at share lookup, and the success audit log line.
  - Header and recipient-length checks in code for flag OFF.
  - A Liquibase changeset widening `key_material_share.recipient`.
- **cdoc2-openapi** (`cdoc2-key-shares-openapi.yaml`): recipient `maxLength` 32 → 255; `x-cdoc2-auth-x5c`, `x-cdoc2-session-token` and `x-cdoc2-session-x5c` change from required to optional; version bump. These only widen the API, so existing clients keep working.
- **Smart-ID/Mobile-ID clients**:
  - With the flag OFF, nothing changes.
  - With the flag ON, the only difference is that expired shares get 404 immediately rather than after the cleanup job runs.
