# Spec Delta

## ADDED Requirements

### Requirement: Malformed audience refused on the SSH path
When the feature flag is enabled and an auth token takes the SSH public key path, the server SHALL respond 401 Unauthorized if the token's `aud` cannot be parsed into a server URL, a shareId and a base64url nonce. It SHALL NOT respond with a server error, and SHALL NOT return the share. The Smart-ID/Mobile-ID path SHALL keep its existing behaviour.

#### Scenario: Nonce is not base64url
- **WHEN** the flag is on and a caller presents a validly signed `ruhnu/pk/` token whose `aud` carries a nonce that is not base64url, for example `?nonce=!!!`
- **THEN** the server responds 401 Unauthorized

#### Scenario: Malformed query in the audience
- **WHEN** the flag is on and a caller presents a validly signed `ruhnu/pk/` token whose `aud` query has a parameter without a name, a duplicated `nonce` parameter or invalid percent-encoding
- **THEN** the server responds 401 Unauthorized

#### Scenario: Well-formed but unknown nonce unchanged
- **WHEN** the flag is on and the token's `aud` carries a well-formed nonce that this server did not issue for this share or that has expired
- **THEN** the server does not return the share and responds as before this change, as for existing recipients

## MODIFIED Requirements

### Requirement: Recipient length
When the feature flag is enabled, the shares server SHALL accept and store recipients of up to 255 characters, so that a `ruhnu/pk/` identifier for a P-256 key fits. Only `ruhnu/pk/` recipients SHALL be longer than 32 characters; any other recipient starting with `ruhnu/` SHALL be refused with 400 Bad Request, and any other recipient longer than 32 characters SHALL be refused with 400 Bad Request, as upstream. While the flag is off, recipients longer than 32 characters SHALL be refused with 400 Bad Request, as upstream.

#### Scenario: Create a share with a P-256 SSH recipient
- **WHEN** the flag is on and a client creates a share whose recipient is a `ruhnu/pk/` identifier for a P-256 key
- **THEN** the server stores the share and later compares the full recipient, without truncating it

#### Scenario: Invalid SSH public key recipient
- **WHEN** the flag is on and a client creates a share whose recipient starts with `ruhnu/pk/` but is not a P-256 OpenSSH public key in canonical form (padded standard base64), for example an Ed25519 key, a P-384 key, malformed base64 or a truncated blob
- **THEN** the server responds 400 Bad Request and stores nothing

#### Scenario: Unknown ruhnu recipient kind
- **WHEN** the flag is on and a client creates a share whose recipient starts with `ruhnu/` but not with `ruhnu/pk/`, for example `ruhnu/pK/AAAA...` or `ruhnu/xyz/abc`
- **THEN** the server responds 400 Bad Request and stores nothing

#### Scenario: Long non-SSH recipient while on
- **WHEN** the flag is on and a client creates a share whose recipient does not start with `ruhnu/` and is longer than 32 characters
- **THEN** the server responds 400 Bad Request and stores nothing, as upstream

#### Scenario: Long recipient while off
- **WHEN** the flag is off and a client creates a share whose recipient is longer than 32 characters
- **THEN** the server responds 400 Bad Request
