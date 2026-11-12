# Spec Delta

## MODIFIED Requirements

### Requirement: Expiry configured per recipient type
When the feature flag `cdoc2.ruhnu.enabled` is on, the shares server SHALL have a configurable default and maximum share lifetime for `ruhnu/pk/` recipients, each 14 days unless configured otherwise. Other recipients, including `ruhnu/did/` recipients, SHALL keep the existing global lifetimes. While the flag is off, the global lifetimes SHALL apply to every share, as upstream.

#### Scenario: Default lifetime for an SSH recipient
- **WHEN** the flag is on and a client creates a share with a `ruhnu/pk/` recipient without requesting an expiry, on a server with the defaults
- **THEN** the share's expiry is 14 days from creation

#### Scenario: Operator configures a different lifetime
- **WHEN** the flag is on, the operator sets `cdoc2.ruhnu.pk.default-expiration-duration` to 1 day and `cdoc2.ruhnu.pk.max-expiration-duration` to 7 days, and a client creates a `ruhnu/pk/` share without requesting an expiry
- **THEN** the share's expiry is 1 day from creation

#### Scenario: Smart-ID lifetimes unchanged
- **WHEN** a client creates a share with an `etsi/` recipient, with the flag either off or on
- **THEN** the global default and maximum lifetimes apply, as before this change

#### Scenario: DID recipients use the global lifetimes
- **WHEN** the flag is on and a client creates a share with a `ruhnu/did/` recipient, with or without requesting an expiry
- **THEN** the global `key-share.default-expiration-duration` and `key-share.max-expiration-duration` apply, and the `cdoc2.ruhnu.pk.*` lifetimes do not
