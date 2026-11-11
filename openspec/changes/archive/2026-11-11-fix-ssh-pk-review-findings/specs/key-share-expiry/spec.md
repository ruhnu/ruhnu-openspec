# Spec Delta

## ADDED Requirements

### Requirement: Default lifetime never exceeds the maximum
When the feature flag `cdoc2.ruhnu.enabled` is on and the default lifetime configured for `ruhnu/pk/` recipients is longer than the maximum configured for them, the shares server SHALL give a share created without a requested expiry the maximum lifetime, and SHALL log a warning that the configured default is longer than the maximum. The global lifetimes used for other recipients SHALL NOT be affected.

#### Scenario: Default longer than the maximum
- **WHEN** the flag is on, the operator sets `cdoc2.ruhnu.pk.max-expiration-duration` to 7 days and leaves the default at 14 days, and a client creates a `ruhnu/pk/` share without requesting an expiry
- **THEN** the share's expiry is 7 days from creation and the response reports `x-expiry-time-adjusted: false`

#### Scenario: Operator warned about the configuration
- **WHEN** the server starts with the flag on and a `ruhnu/pk/` default lifetime longer than the maximum
- **THEN** the server log contains a warning naming both properties and stating that the maximum is used as the default

#### Scenario: Default within the maximum
- **WHEN** the flag is on, the `ruhnu/pk/` default lifetime is 1 day and the maximum is 7 days, and a client creates a `ruhnu/pk/` share without requesting an expiry
- **THEN** the share's expiry is 1 day from creation and no such warning is logged

#### Scenario: Global lifetimes unchanged
- **WHEN** a client creates a share with an `etsi/` recipient without requesting an expiry, with the flag either off or on
- **THEN** the global default lifetime applies as before this change, whether or not it is longer than the global maximum
