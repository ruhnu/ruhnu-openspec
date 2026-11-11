# Proposal

## Why

A `ruhnu/pk/` recipient can only lose access by expiring, so it relies on a 14-day lifetime and frequent re-encryption. Organisations already list their people's keys in a DID document. A recipient that names the person's DID as well as the SSH key lets the organisation revoke access at once, by removing the key from the DID document. The DID host still must not be able to grant access: if it is compromised, it must not be able to add keys that open existing containers.

This is phase 2 of the CDOC2-ssh work, which phase 1 (`add-ssh-pk-recipient`) reserved as `ruhnu/did/...` under the `RUHNU` recipient type.

**Prerequisite:** phase 1's code is on `feat/add-ssh-pk-recipient` in all four repos and is not yet merged into `ruhnu`. This change builds on those branches.

## What Changes

- **Every shares-server behaviour change below sits behind the existing feature flag `cdoc2.ruhnu.enabled`.** With the flag off, `ruhnu/did/` shares are refused as upstream would refuse them. One change cannot be gated: the OpenAPI document gains a `503` response.
- New `RUHNU` recipient kind: `recipient_id` is `ruhnu/did/<did>/hash/<fingerprint>`, for example `ruhnu/did/did:web:example.org:p:alice/hash/NzbLsXh8uDCcd-6MNwXF4W_7noWXFZAfHkxZsRGC9Xs`.
  - `<did>` is a `did:web` DID.
  - `<fingerprint>` is the SHA-256 of the OpenSSH public key blob, in base64url without padding: the `ssh-keygen -l` fingerprint with `+/` replaced by `-_`.
  - Only ECDSA P-256 keys are supported, as in phase 1.
- **A share is released only if both hold:** the caller proves possession of the key with that fingerprint, and the DID document, fetched at download time, still lists that key under `authentication`.
  - The fingerprint fixes the key in the container. A DID host that is later compromised can withdraw access but cannot grant it.
  - The share is refused when the DID host is down, the DID document is missing, or the key has been removed from it.
- `cdoc2-cli` can encrypt to a DID. It resolves the DID and creates one recipient per P-256 key listed under `authentication`, printing the pinned fingerprints. The DID host is trusted at that moment; this is an accepted trade-off (see design).
- `cdoc2-cli` decrypts `ruhnu/did/` recipients through ssh-agent, as in phase 1. It finds the agent key by fingerprint and does not resolve the DID itself. The auth token carries the public key in the JWS `jwk` header, because the identifier holds only its hash.
- With the flag on, the shares server:
  - accepts `ruhnu/did/` shares only for DID hosts in a new allowlist, `cdoc2.ruhnu.did.allowed-hosts` (host names or `*.<domain>` wildcards), which is empty by default;
  - issues nonces for them without session headers, as for `ruhnu/pk/`;
  - verifies the token, then fetches the DID document over HTTPS on every share download, with no cache;
  - returns `404` when the DID document is missing or no longer lists the key, and `503` when the DID host cannot be reached;
  - logs every `ruhnu/did/` release in the audit log.
- `ruhnu/did/` shares use the existing global share lifetimes. No new lifetime configuration is added.
- **Not in scope:**
  - DID methods other than `did:web`;
  - keys in a DID document given in any form other than `publicKeyJwk`;
  - Ed25519/RSA keys;
  - caching DID documents;
  - encrypting to a fingerprint the sender pinned in advance, as opposed to one read from the DID document;
  - key rotation without re-encrypting.

## Capabilities

### New Capabilities

- `did-recipient`: the DID-bound SSH key recipient. It covers:
  - the identifier format;
  - which keys a DID document lists;
  - encrypting to a DID and decrypting through ssh-agent in the client library and CLI;
  - the auth token format.

### Modified Capabilities

- `key-share-authentication`:
  - the feature flag also gates `ruhnu/did/` recipients;
  - the token issuer selects a DID path;
  - nonces are issued without a session for `ruhnu/did/` shares;
  - new requirements cover DID share release and the DID host allowlist;
  - the audit log records `ruhnu/did/` releases with the full recipient id.
- `key-share-expiry`: states that `ruhnu/did/` recipients use the global lifetimes.

## Impact

- **cdoc2-auth**:
  - parsing and building `ruhnu/did/` identifiers;
  - SSH fingerprints of OpenSSH blobs and of P-256 JWKs;
  - a `did:web` resolver and DID document key extraction, shared by client and server;
  - a verifier for DID auth tokens;
  - `AuthTokenCreator` support for a `jwk` header.
- **cdoc2-java-ref-impl**:
  - `cdoc2-lib`: the `ruhnu/did/` kind is accepted, plus DID encryption key material (resolve and pin), DID decryption key material, and a `KekTools` path that sends the `jwk` header and maps `404`/`503` to readable errors.
  - `cdoc2-cli`: an option to encrypt to a DID. `--ssh-agent` also matches `ruhnu/did/` recipients, and `info` shows the DID and fingerprint.
  - No FlatBuffers change: `RUHNU` already covers the new kind.
- **cdoc2-shares-server**:
  - `cdoc2.ruhnu.did.*` configuration: the allowlist and resolver timeouts;
  - create validation, plus nonce and get-share dispatch for `ruhnu/did/`;
  - the DID check, the audit log entry and the admin guide.
  - The server makes outbound HTTPS requests to the allowlisted DID hosts.
- **cdoc2-openapi** (`cdoc2-key-shares-openapi.yaml`): get-share documents `503`, the recipient description mentions `ruhnu/did/`, and the version is bumped. This only widens the API.
- **DID hosts**: each share download makes one HTTPS request per shares server to the recipient's DID host. Decryption fails while the host is unreachable.
- **Smart-ID/Mobile-ID and `ruhnu/pk/` recipients**: no change.
