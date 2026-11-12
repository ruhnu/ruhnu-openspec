# Spec Delta

## RENAMED Requirements

- FROM: `### Requirement: Malformed audience refused on the SSH path`
- TO: `### Requirement: Malformed audience refused on the SSH and DID paths`

## MODIFIED Requirements

### Requirement: Fork behaviour behind a feature flag
The shares server SHALL have a configuration option `cdoc2.ruhnu.enabled`, default `false`. While it is `false`, every request SHALL produce the same response status as upstream, including the header and recipient-length checks that the upstream API document makes mandatory. SSH public key recipients, DID recipients, per-type expiry and the release audit log SHALL apply only while it is `true`.

#### Scenario: Flag is off by default
- **WHEN** the shares server starts without `cdoc2.ruhnu.enabled` configured
- **THEN** the flag is off

#### Scenario: SSH recipient refused while off
- **WHEN** the flag is off and a client creates a share whose recipient is a `ruhnu/pk/` identifier
- **THEN** the server responds 400 Bad Request and stores nothing

#### Scenario: DID recipient refused while off
- **WHEN** the flag is off and a client creates a share whose recipient is a `ruhnu/did/` identifier, even for a host in `cdoc2.ruhnu.did.allowed-hosts`
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

#### Scenario: Flag on enables DID recipients
- **WHEN** the flag is on and a client creates a share whose recipient is a valid `ruhnu/did/` identifier for a host in `cdoc2.ruhnu.did.allowed-hosts`
- **THEN** the server stores the share

### Requirement: Authentication path chosen by token issuer
When the feature flag is enabled, the get-share endpoint SHALL verify an auth token whose `iss` starts with `ruhnu/pk/` as an SSH public key token, one whose `iss` starts with `ruhnu/did/` as a DID token, and any other token through the existing Smart-ID/Mobile-ID path in its existing order. On every path the server SHALL release the share only if the verified identity equals the share's stored recipient.

#### Scenario: SSH token for a Smart-ID share
- **WHEN** the flag is on and a caller requests a share whose recipient is `etsi/PNOEE-...` with a valid auth token signed by an SSH key, whose `iss` is that key's `ruhnu/pk/` identifier
- **THEN** the server responds 404 Not Found and does not return the share, because the verified identity does not equal the stored recipient

#### Scenario: Smart-ID credentials for an SSH share
- **WHEN** the flag is on and a caller requests a share whose recipient is `ruhnu/pk/...` with a valid Smart-ID auth token, session token and certificate
- **THEN** the server responds 404 Not Found and does not return the share, because the verified identity does not equal the stored recipient

#### Scenario: DID token for an SSH public key share of the same key
- **WHEN** the flag is on and a caller requests a share whose recipient is the `ruhnu/pk/` identifier of a key, with a valid DID auth token signed by that same key
- **THEN** the server responds 404 Not Found and does not return the share, because the verified identity does not equal the stored recipient

#### Scenario: Smart-ID token without session headers while on
- **WHEN** the flag is on and a caller requests a share with an auth token whose `iss` starts with neither `ruhnu/pk/` nor `ruhnu/did/`, without `x-cdoc2-session-token` or `x-cdoc2-session-x5c`
- **THEN** the server responds 400 Bad Request, as upstream

### Requirement: Nonce for SSH public key shares
When the feature flag is enabled, the nonce endpoint SHALL issue a nonce without a session token or certificate for a share whose stored recipient starts with `ruhnu/pk/` or `ruhnu/did/`. It SHALL NOT resolve a DID to issue a nonce. Requests that carry session headers SHALL be handled by the existing path unchanged.

#### Scenario: Nonce without session token
- **WHEN** the flag is on and a caller requests a nonce for an existing `ruhnu/pk/` share without session headers
- **THEN** the server creates and returns a nonce for that share

#### Scenario: Nonce for a DID share without resolving the DID
- **WHEN** the flag is on and a caller requests a nonce without session headers for an existing `ruhnu/did/` share, while its DID host is unavailable
- **THEN** the server creates and returns a nonce for that share, and sends no request to the DID host

#### Scenario: Nonce without session token for a Smart-ID share
- **WHEN** the flag is on and a caller requests a nonce for an existing `etsi/` share without session headers
- **THEN** the server responds 400 Bad Request and creates no nonce

#### Scenario: Nonce for unknown share
- **WHEN** the flag is on and a caller requests a nonce without session headers for a share id that does not exist
- **THEN** the server responds 404 Not Found

### Requirement: Audit log of released shares
When the feature flag is enabled, the shares server SHALL log every successful share release, for all recipient types. Each entry SHALL include the shareId, a recipient identifier, and the time of release. For a `ruhnu/pk/` recipient the recipient identifier SHALL be the key's OpenSSH SHA256 fingerprint, as shown by `ssh-keygen -l`. For a `ruhnu/did/` recipient it SHALL be the full `recipient_id`, which names the DID and the key fingerprint. Any other recipient SHALL be logged as redacted, so that personal identification codes of `etsi/` recipients are not written to the log, as in upstream logging.

#### Scenario: Release of an SSH public key share is logged
- **WHEN** the flag is on and the server returns a `ruhnu/pk/` share to a caller
- **THEN** a log entry records the shareId, the recipient key's SHA256 fingerprint and the time of release

#### Scenario: Release of a DID share is logged
- **WHEN** the flag is on and the server returns a `ruhnu/did/` share to a caller
- **THEN** a log entry records the shareId, the share's `ruhnu/did/` recipient identifier and the time of release

#### Scenario: Release of a Smart-ID share is logged without the identification code
- **WHEN** the flag is on and the server returns an `etsi/` share to a caller
- **THEN** a log entry records the shareId, a redacted recipient and the time of release, and the entry does not contain the personal identification code

#### Scenario: Refusal is not logged as a release
- **WHEN** the flag is on and the server refuses a share request, including a `ruhnu/did/` request refused because of the DID check
- **THEN** no release entry is written for that request

### Requirement: Recipient length
When the feature flag is enabled, the shares server SHALL accept and store `ruhnu/pk/` and `ruhnu/did/` recipients of up to 255 characters, so that an identifier for a P-256 key or a DID recipient identifier fits, and SHALL refuse with 400 Bad Request any other recipient that starts with `ruhnu/` or is longer than 32 characters. While the flag is off, recipients longer than 32 characters SHALL be refused with 400 Bad Request, as upstream.

#### Scenario: Create a share with a P-256 SSH recipient
- **WHEN** the flag is on and a client creates a share whose recipient is a `ruhnu/pk/` identifier for a P-256 key
- **THEN** the server stores the share and later compares the full recipient, without truncating it

#### Scenario: Create a share with a DID recipient
- **WHEN** the flag is on and a client creates a share whose recipient is a valid `ruhnu/did/` identifier for a DID on a host in `cdoc2.ruhnu.did.allowed-hosts`
- **THEN** the server stores the share and later compares the full recipient, without truncating it

#### Scenario: Invalid SSH public key recipient
- **WHEN** the flag is on and a client creates a share whose recipient starts with `ruhnu/pk/` but is not a P-256 OpenSSH public key in canonical form (padded standard base64), for example an Ed25519 key, a P-384 key, malformed base64 or a truncated blob
- **THEN** the server responds 400 Bad Request and stores nothing

#### Scenario: Unknown ruhnu recipient kind
- **WHEN** the flag is on and a client creates a share whose recipient starts with `ruhnu/` but with neither `ruhnu/pk/` nor `ruhnu/did/`, for example `ruhnu/pK/AAAA...` or `ruhnu/xyz/abc`
- **THEN** the server responds 400 Bad Request and stores nothing

#### Scenario: Long non-SSH recipient while on
- **WHEN** the flag is on and a client creates a share whose recipient does not start with `ruhnu/` and is longer than 32 characters
- **THEN** the server responds 400 Bad Request and stores nothing, as upstream

#### Scenario: Long recipient while off
- **WHEN** the flag is off and a client creates a share whose recipient is longer than 32 characters
- **THEN** the server responds 400 Bad Request

### Requirement: Malformed audience refused on the SSH and DID paths
When the feature flag is enabled and an auth token takes the SSH public key path or the DID path, the server SHALL respond 401 Unauthorized if the token's `aud` cannot be parsed into a server URL, a shareId and a base64url nonce. It SHALL NOT respond with a server error, SHALL NOT fetch the DID document, and SHALL NOT return the share. The Smart-ID/Mobile-ID path SHALL keep its existing behaviour.

#### Scenario: Nonce is not base64url
- **WHEN** the flag is on and a caller presents a validly signed `ruhnu/pk/` or `ruhnu/did/` token whose `aud` carries a nonce that is not base64url, for example `?nonce=!!!`
- **THEN** the server responds 401 Unauthorized

#### Scenario: Malformed query in the audience
- **WHEN** the flag is on and a caller presents a validly signed `ruhnu/pk/` or `ruhnu/did/` token whose `aud` query has a parameter without a name, a duplicated `nonce` parameter or invalid percent-encoding
- **THEN** the server responds 401 Unauthorized

#### Scenario: Well-formed but unknown nonce unchanged
- **WHEN** the flag is on and the token's `aud` carries a well-formed nonce that this server did not issue for this share or that has expired
- **THEN** the server does not return the share and responds as before this change, as for existing recipients

## ADDED Requirements

### Requirement: Share release for DID shares
When the feature flag is enabled, the server SHALL release a `ruhnu/did/` share only if the token's `jwk` header is a P-256 key whose fingerprint equals the one in `iss`, the JWS verifies with that key, the existing audience, shareId and nonce checks pass, `iss` equals the stored recipient, and the DID document resolved for that request lists the key. It SHALL resolve the DID only after every other check has passed.

#### Scenario: Valid token and key listed
- **WHEN** the flag is on and a caller presents an `ES256` auth token for a `ruhnu/did/` share, with `iss` equal to the share's recipient, a `jwk` header key whose fingerprint matches `iss`, signed by that key, with an `aud` naming this server, this share and an unexpired nonce issued for it, and the DID document lists the key
- **THEN** the server returns the share

#### Scenario: Key removed from the DID document
- **WHEN** the token passes every other check, but the DID document no longer lists the key
- **THEN** the server responds 404 Not Found and does not return the share

#### Scenario: DID not found
- **WHEN** the token passes every other check, but the DID is not found, for example because the DID host responds 404
- **THEN** the server responds 404 Not Found and does not return the share

#### Scenario: DID host unavailable
- **WHEN** the token passes every other check, but the DID host is unavailable
- **THEN** the server responds 503 Service Unavailable and does not return the share

#### Scenario: Retry after a DID host outage
- **WHEN** a request got 503 because the DID host was unavailable, and the caller repeats it with the same token before the nonce expires, after the DID host is reachable again
- **THEN** the server returns the share

#### Scenario: Header key does not match the fingerprint
- **WHEN** the token's `jwk` header key has a fingerprint other than the one in `iss`
- **THEN** the server responds 401 Unauthorized and sends no request to the DID host

#### Scenario: Token signed by a different key
- **WHEN** the token's `jwk` header key matches the fingerprint in `iss`, but the signature was made with a different private key
- **THEN** the server responds 401 Unauthorized and sends no request to the DID host

#### Scenario: Missing or unsupported header key
- **WHEN** the token has no `jwk` header, its `jwk` is not a P-256 public key, or its JWS header `alg` is anything other than `ES256`
- **THEN** the server responds 401 Unauthorized and sends no request to the DID host

#### Scenario: Token for another recipient
- **WHEN** the token is validly signed but its `iss` names a different DID or fingerprint than the share's recipient
- **THEN** the server responds 404 Not Found, does not return the share, and sends no request to any DID host

#### Scenario: Token for another server or share
- **WHEN** the token's `aud` names a different server URL or a different shareId than the request
- **THEN** the server responds 401 Unauthorized and sends no request to the DID host

### Requirement: DID host allowlist
When the feature flag is enabled, the shares server SHALL have a configuration option `cdoc2.ruhnu.did.allowed-hosts`: a list of host entries, each optionally with a port, empty by default. The server SHALL store a `ruhnu/did/` share only if its recipient is a valid DID recipient identifier whose DID host and port match an entry in the list. It SHALL NOT send a request to a DID host that matches no entry.

#### Scenario: Share for an allowed host
- **WHEN** `cdoc2.ruhnu.did.allowed-hosts` contains `example.org` and a client creates a share whose recipient is `ruhnu/did/did:web:example.org:p:alice/hash/<fingerprint>`
- **THEN** the server stores the share

#### Scenario: Share for a host not in the list
- **WHEN** a client creates a share whose recipient is a `ruhnu/did/` identifier for a host that is not in `cdoc2.ruhnu.did.allowed-hosts`
- **THEN** the server responds 400 Bad Request and stores nothing

#### Scenario: No hosts configured
- **WHEN** `cdoc2.ruhnu.did.allowed-hosts` is not configured and a client creates a share whose recipient is a `ruhnu/did/` identifier
- **THEN** the server responds 400 Bad Request and stores nothing

#### Scenario: Port not in the list
- **WHEN** `cdoc2.ruhnu.did.allowed-hosts` contains `example.org` but not `example.org:8443`, and a client creates a share whose recipient's DID is `did:web:example.org%3A8443:p:alice`
- **THEN** the server responds 400 Bad Request and stores nothing

#### Scenario: Malformed DID recipient
- **WHEN** a client creates a share whose recipient starts with `ruhnu/did/` but is not a valid DID recipient identifier, for example a `did:key` DID, a `did:web` DID with an IP address, a fingerprint that is not 43 characters of base64url, or an extra path segment
- **THEN** the server responds 400 Bad Request and stores nothing

#### Scenario: Host removed from the list after the share was created
- **WHEN** a `ruhnu/did/` share was created while its host was in `cdoc2.ruhnu.did.allowed-hosts`, the host has since been removed from the list, and a caller presents a token that passes every other check
- **THEN** the server responds 404 Not Found, does not return the share, and sends no request to the DID host

### Requirement: Wildcard DID host entries
An entry in `cdoc2.ruhnu.did.allowed-hosts` MAY be a wildcard `*.<domain>`, where `<domain>` has at least two labels. A wildcard entry SHALL match a host that consists of exactly one DNS label followed by `.<domain>`, and no other host. The server SHALL NOT start with a malformed entry.

#### Scenario: Host matched by a wildcard entry
- **WHEN** `cdoc2.ruhnu.did.allowed-hosts` contains `*.github.io` and a client creates a share whose recipient's DID is `did:web:alice.github.io`
- **THEN** the server stores the share

#### Scenario: Wildcard does not match the domain itself or deeper subdomains
- **WHEN** `cdoc2.ruhnu.did.allowed-hosts` contains `*.github.io` but no other entry for these hosts, and a client creates a share whose recipient's DID is `did:web:github.io:p:alice` or `did:web:a.b.github.io`
- **THEN** the server responds 400 Bad Request and stores nothing

#### Scenario: Malformed allowlist entry
- **WHEN** `cdoc2.ruhnu.did.allowed-hosts` contains an entry such as `*`, `*.org` or `a.*.org`
- **THEN** the server does not start
