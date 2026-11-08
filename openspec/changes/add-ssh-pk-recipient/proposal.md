# Proposal

## Why

Today only Smart-ID and Mobile-ID users can open a CDOC2 key-shares recipient. Engineers who keep encrypted secrets in Git need a recipient they can open with the SSH key already loaded in their ssh-agent, with access that ends on a short, server-enforced schedule. ssh-agent can sign but not decrypt, which is the limitation the key-shares scheme was built for, so the scheme carries over. What has to be replaced is the authentication leg, which is currently tied to the Smart-ID/Mobile-ID, RP-server and auth-server stack.

This is phase 1 of the CDOC2-ssh work. Its goal is a working end-to-end demo with boundaries shaped for later upstreaming. DID recipients (`ruhnu/did/...`) are phase 2 and a separate change.

**Prerequisite:** `fix-expired-share-release` must be applied and archived first. This change relies on its guarantee that expired shares are refused with 404 straight away.

## What Changes

- **Every shares-server behaviour change below sits behind the feature flag `cdoc2.ruhnu.enabled`, default OFF.** With the flag OFF, the shares server behaves like upstream, where upstream includes the always-on `fix-expired-share-release`. Two differences cannot be gated: the wider DB column, and the relaxed OpenAPI document. The client library, CLI and cdoc2-auth have no flag, because their changes only add code.
- New key-shares recipient type `RUHNU` for fork-defined identifiers of the form `ruhnu/<kind>/...`; the kind is read from the `recipient_id` prefix. Phase 1 adds one kind, an SSH public key: `recipient_id` is `ruhnu/pk/<base64 OpenSSH public key blob>`, so the container names the key itself and no trust anchor or registry exists. Phase 2 adds `ruhnu/did/...` under the same type.
- Phase 1 supports only ECDSA P-256 (`ecdsa-sha2-nistp256`) keys, signed as JWS `ES256`. Ed25519 and RSA are deferred.
- `cdoc2-cli` can encrypt to an SSH public key and decrypt by signing the per-share auth token (SD-JWT) through ssh-agent (`$SSH_AUTH_SOCK`). The signature is a genuine JWS, not SSHSIG.
- When the flag is enabled, the shares server accepts SSH public key recipients:
  - The get-share endpoint chooses the verification path from the auth token's `iss` prefix. For `ruhnu/pk/` it verifies the token against the public key in `iss`, with no session token, X.509 certificate, Smart-ID RPv3 parameters or RP countersignature. Every path verifies the token, including the existing audience, shareId and nonce checks, then requires the verified identity to equal the share's stored recipient.
  - The nonce endpoint issues nonces without a session token for shares whose stored recipient starts with `ruhnu/pk/`.
  - Smart-ID/Mobile-ID requests run through the existing code in the existing order.
- When the flag is enabled, share expiry is configurable per recipient type, as a default and a maximum. For `ruhnu/pk/` both default to 14 days. Clients may request a shorter lifetime with `x-expiry-time`; longer requests are cut to the maximum.
- When the flag is enabled, the shares server logs every successful share release (audit trail).
- Recipient length limits are widened (DB column, OpenAPI `maxLength`), because a P-256 `ruhnu/pk/` identifier is about 149 characters.
- **Not in scope:** Ed25519/RSA keys, DID recipients, a revoke-now API (revocation is expiry, or deleting rows out of band), nonce rate limiting, SSH certificates/CA.

## Capabilities

### New Capabilities

- `ssh-key-recipient`: the SSH public key recipient type. Covers the identifier format, the supported key types, and encrypting and decrypting with it in the client library and CLI, including signing through ssh-agent.
- `key-share-authentication`: how the shares server authenticates nonce and share requests: the feature flag, choosing the path per recipient type, SSH public key verification, and the audit log of released shares.
### Modified Capabilities

- `key-share-expiry` (created by `fix-expired-share-release`): adds a configurable default and maximum share lifetime per recipient type, and client-requested shorter lifetimes.

## Impact

- **cdoc2-java-ref-impl**
  - `cdoc2-schema` `recipients.fbs`: new `KeyShareRecipientType` value `RUHNU`.
  - `cdoc2-lib`: recipient (de)serialisation, encryption/decryption key material, a new ssh-agent JWS signer, and a token-creation path that needs no session token or RP client.
  - `cdoc2-cli`: options to encrypt to and decrypt with an SSH key.
- **cdoc2-auth**: a new verifier for SD-JWT auth tokens signed by an SSH public key named in `iss`. No X.509, no trust store.
- **cdoc2-shares-server**
  - The `cdoc2.ruhnu.*` configuration: the feature flag and the `pk` lifetimes.
  - Token-based dispatch in get-share, and header-based dispatch in nonce.
  - The success audit log line.
  - Header and recipient-length checks in code for flag OFF.
  - A Liquibase changeset widening `key_material_share.recipient`.
- **cdoc2-openapi** (`cdoc2-key-shares-openapi.yaml`): recipient `maxLength` 32 → 255; `x-cdoc2-auth-x5c`, `x-cdoc2-session-token` and `x-cdoc2-session-x5c` change from required to optional; version bump. These only widen the API, so existing clients keep working.
- **Smart-ID/Mobile-ID clients**: no change, with the flag either OFF or ON.
