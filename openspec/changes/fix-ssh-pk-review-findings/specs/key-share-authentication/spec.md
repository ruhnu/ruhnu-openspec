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
