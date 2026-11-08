# Spec Delta

## ADDED Requirements

### Requirement: Expiry configured per recipient type
When the feature flag `cdoc2.ruhnu.enabled` is on, the shares server SHALL have a configurable default and maximum share lifetime for `ruhnu/pk/` recipients, each 14 days unless configured otherwise. Other recipients SHALL keep the existing global lifetimes. While the flag is off, the global lifetimes SHALL apply to every share, as upstream.

#### Scenario: Default lifetime for an SSH recipient
- **WHEN** the flag is on and a client creates a share with a `ruhnu/pk/` recipient without requesting an expiry, on a server with the defaults
- **THEN** the share's expiry is 14 days from creation

#### Scenario: Operator configures a different lifetime
- **WHEN** the flag is on, the operator sets `cdoc2.ruhnu.pk.default-expiration-duration` to 1 day and `cdoc2.ruhnu.pk.max-expiration-duration` to 7 days, and a client creates a `ruhnu/pk/` share without requesting an expiry
- **THEN** the share's expiry is 1 day from creation

#### Scenario: Smart-ID lifetimes unchanged
- **WHEN** a client creates a share with an `etsi/` recipient, with the flag either off or on
- **THEN** the global default and maximum lifetimes apply, as before this change

### Requirement: Client may request a shorter lifetime
When a client requests an expiry with `x-expiry-time`, the server SHALL use it if it does not exceed the maximum that applies to the share. Otherwise the server SHALL cut it to that maximum and report `x-expiry-time-adjusted: true`.

#### Scenario: Shorter lifetime accepted
- **WHEN** the flag is on and a client creates a `ruhnu/pk/` share requesting an expiry 3 days from now, with a maximum of 14 days
- **THEN** the share's expiry is the requested time and the response reports `x-expiry-time-adjusted: false`

#### Scenario: Longer lifetime cut to the maximum
- **WHEN** the flag is on and a client creates a `ruhnu/pk/` share requesting an expiry 1 year from now, with a maximum of 14 days
- **THEN** the share's expiry is 14 days from creation and the response reports `x-expiry-time-adjusted: true`

#### Scenario: Expiry reported to the client
- **WHEN** the server creates any share
- **THEN** the response's `x-expiry-time` header carries the expiry actually stored
