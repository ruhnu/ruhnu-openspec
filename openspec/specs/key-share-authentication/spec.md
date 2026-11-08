# key-share-authentication Specification

## Purpose

Defines how the shares server decides whether a caller may get a nonce for a key share and download it, per recipient type, how the fork's behaviour is switched on, and what the server records when a share is released.

## Requirements

### Requirement: Fork behaviour behind a feature flag
The shares server SHALL have a configuration option `cdoc2.ruhnu.enabled`, default `false`. While it is `false`, every request SHALL produce the same response status as upstream, including the header and recipient-length checks that the upstream API document makes mandatory. SSH public key recipients, per-type expiry and the release audit log SHALL apply only while it is `true`.

#### Scenario: Flag is off by default
- **WHEN** the shares server starts without `cdoc2.ruhnu.enabled` configured
- **THEN** the flag is off

#### Scenario: SSH recipient refused while off
- **WHEN** the flag is off and a client creates a share whose recipient is a `ruhnu/pk/` identifier
- **THEN** the server responds 400 Bad Request and stores nothing

#### Scenario: Missing session headers refused while off
- **WHEN** the flag is off and a caller requests a nonce or a share without `x-cdoc2-session-token` or `x-cdoc2-session-x5c`, or requests a share without `x-cdoc2-auth-x5c`
- **THEN** the server responds 400 Bad Request, as upstream

#### Scenario: Smart-ID/Mobile-ID unaffected by the flag
- **WHEN** a client completes the Smart-ID or Mobile-ID flow for an unexpired share with an `etsi/` recipient, with the flag either off or on
- **THEN** the server returns the share

#### Scenario: Flag on enables SSH recipients
- **WHEN** the flag is on and a client creates a share whose recipient is a `ruhnu/pk/` identifier for a P-256 key
- **THEN** the server stores the share

### Requirement: Authentication path chosen by token issuer
When the feature flag is enabled, the get-share endpoint SHALL verify an auth token whose `iss` starts with `ruhnu/pk/` as an SSH public key token, and any other token through the existing Smart-ID/Mobile-ID path in its existing order. On every path the server SHALL release the share only if the verified identity equals the share's stored recipient.

#### Scenario: SSH token for a Smart-ID share
- **WHEN** the flag is on and a caller requests a share whose recipient is `etsi/PNOEE-...` with a valid auth token signed by an SSH key, whose `iss` is that key's `ruhnu/pk/` identifier
- **THEN** the server responds 404 Not Found and does not return the share, because the verified identity does not equal the stored recipient

#### Scenario: Smart-ID credentials for an SSH share
- **WHEN** the flag is on and a caller requests a share whose recipient is `ruhnu/pk/...` with a valid Smart-ID auth token, session token and certificate
- **THEN** the server responds 404 Not Found and does not return the share, because the verified identity does not equal the stored recipient

#### Scenario: Smart-ID token without session headers while on
- **WHEN** the flag is on and a caller requests a share with an auth token whose `iss` does not start with `ruhnu/pk/`, without `x-cdoc2-session-token` or `x-cdoc2-session-x5c`
- **THEN** the server responds 400 Bad Request, as upstream

### Requirement: Nonce for SSH public key shares
When the feature flag is enabled, the nonce endpoint SHALL issue a nonce without a session token or certificate for a share whose stored recipient starts with `ruhnu/pk/`. Requests that carry session headers SHALL be handled by the existing path unchanged.

#### Scenario: Nonce without session token
- **WHEN** the flag is on and a caller requests a nonce for an existing `ruhnu/pk/` share without session headers
- **THEN** the server creates and returns a nonce for that share

#### Scenario: Nonce without session token for a Smart-ID share
- **WHEN** the flag is on and a caller requests a nonce for an existing `etsi/` share without session headers
- **THEN** the server responds 400 Bad Request and creates no nonce

#### Scenario: Nonce for unknown share
- **WHEN** the flag is on and a caller requests a nonce without session headers for a share id that does not exist
- **THEN** the server responds 404 Not Found

### Requirement: Share release for SSH public key shares
When the feature flag is enabled, the server SHALL release a `ruhnu/pk/` share only if the token's `iss` equals the stored recipient, its JWS verifies with the key decoded from `iss`, and the existing audience, shareId and nonce checks pass. No session token, certificate, Smart-ID parameters or RP countersignature SHALL be required.

#### Scenario: Valid SSH-signed token
- **WHEN** the flag is on and a caller presents an `ES256` auth token for a `ruhnu/pk/` share, with `iss` equal to the share's recipient, signed by the matching private key, and with an `aud` naming this server, this share and an unexpired nonce issued for it
- **THEN** the server returns the share

#### Scenario: Token signed by a different key
- **WHEN** the token's `iss` equals the share's recipient but the signature was made with a different private key
- **THEN** the server responds 401 Unauthorized and does not return the share

#### Scenario: Token for another recipient
- **WHEN** the token is validly signed but its `iss` names a different `ruhnu/pk/` key than the share's recipient
- **THEN** the server responds 404 Not Found and does not return the share

#### Scenario: Unsupported algorithm
- **WHEN** the token's JWS header `alg` is anything other than `ES256`, or the key in `iss` is not P-256
- **THEN** the server responds 401 Unauthorized

#### Scenario: Expired or reused nonce
- **WHEN** the token's `aud` carries a nonce that has expired or that this server did not issue for this share
- **THEN** the server does not return the share, as for existing recipients

#### Scenario: Token for another server or share
- **WHEN** the token's `aud` names a different server URL or a different shareId than the request
- **THEN** the server responds 401 Unauthorized

### Requirement: Audit log of released shares
When the feature flag is enabled, the shares server SHALL log every successful share release, for all recipient types. Each entry SHALL include the shareId, a recipient identifier, and the time of release. For a `ruhnu/pk/` recipient the recipient identifier SHALL be the key's OpenSSH SHA256 fingerprint, as shown by `ssh-keygen -l`. Any other recipient SHALL be logged as redacted, so that personal identification codes of `etsi/` recipients are not written to the log, as in upstream logging.

#### Scenario: Release of an SSH public key share is logged
- **WHEN** the flag is on and the server returns a `ruhnu/pk/` share to a caller
- **THEN** a log entry records the shareId, the recipient key's SHA256 fingerprint and the time of release

#### Scenario: Release of a Smart-ID share is logged without the identification code
- **WHEN** the flag is on and the server returns an `etsi/` share to a caller
- **THEN** a log entry records the shareId, a redacted recipient and the time of release, and the entry does not contain the personal identification code

#### Scenario: Refusal is not logged as a release
- **WHEN** the flag is on and the server refuses a share request
- **THEN** no release entry is written for that request

### Requirement: Recipient length
When the feature flag is enabled, the shares server SHALL accept and store recipients of up to 255 characters, so that a `ruhnu/pk/` identifier for a P-256 key fits. While the flag is off, recipients longer than 32 characters SHALL be refused with 400 Bad Request, as upstream.

#### Scenario: Create a share with a P-256 SSH recipient
- **WHEN** the flag is on and a client creates a share whose recipient is a `ruhnu/pk/` identifier for a P-256 key
- **THEN** the server stores the share and later compares the full recipient, without truncating it

#### Scenario: Invalid SSH public key recipient
- **WHEN** the flag is on and a client creates a share whose recipient starts with `ruhnu/pk/` but is not a P-256 OpenSSH public key in canonical form (padded standard base64), for example an Ed25519 key, a P-384 key, malformed base64 or a truncated blob
- **THEN** the server responds 400 Bad Request and stores nothing

#### Scenario: Long recipient while off
- **WHEN** the flag is off and a client creates a share whose recipient is longer than 32 characters
- **THEN** the server responds 400 Bad Request
