# Spec Delta

## Purpose

Defines how long the shares server keeps a key share, per recipient type, so that recipients without other ways to revoke access lose it on a schedule the server operator controls.

## ADDED Requirements

### Requirement: Expiry configured per recipient type
When the feature flag `cdoc2.ruhnu.enabled` is on, the shares server SHALL have a configurable default and maximum share lifetime for `private/pk/` recipients, each 14 days unless configured otherwise. Other recipients SHALL keep the existing global lifetimes. While the flag is off, the global lifetimes SHALL apply to every share, as upstream.

#### Scenario: Default lifetime for an SSH recipient
- **WHEN** the flag is on and a client creates a share with a `private/pk/` recipient without requesting an expiry, on a server with the defaults
- **THEN** the share's expiry is 14 days from creation

#### Scenario: Operator configures a different lifetime
- **WHEN** the flag is on, the operator sets `cdoc2.ruhnu.pk.default-expiration-duration` to 1 day and `cdoc2.ruhnu.pk.max-expiration-duration` to 7 days, and a client creates a `private/pk/` share without requesting an expiry
- **THEN** the share's expiry is 1 day from creation

#### Scenario: Smart-ID lifetimes unchanged
- **WHEN** a client creates a share with an `etsi/` recipient, with the flag either off or on
- **THEN** the global default and maximum lifetimes apply, as before this change

### Requirement: Client may request a shorter lifetime
When a client requests an expiry with `x-expiry-time`, the server SHALL use it if it does not exceed the maximum that applies to the share. Otherwise the server SHALL cut it to that maximum and report `x-expiry-time-adjusted: true`.

#### Scenario: Shorter lifetime accepted
- **WHEN** the flag is on and a client creates a `private/pk/` share requesting an expiry 3 days from now, with a maximum of 14 days
- **THEN** the share's expiry is the requested time and the response reports `x-expiry-time-adjusted: false`

#### Scenario: Longer lifetime cut to the maximum
- **WHEN** the flag is on and a client creates a `private/pk/` share requesting an expiry 1 year from now, with a maximum of 14 days
- **THEN** the share's expiry is 14 days from creation and the response reports `x-expiry-time-adjusted: true`

#### Scenario: Expiry reported to the client
- **WHEN** the server creates any share
- **THEN** the response's `x-expiry-time` header carries the expiry actually stored

### Requirement: Expired shares cannot be fetched
When the feature flag is on, the shares server SHALL treat a share whose expiry has passed as not found, for all recipient types, even if the cleanup job has not yet removed it. The nonce and get-share endpoints SHALL respond 404 Not Found at the point where they look up the share, as they do for a deleted share.

#### Scenario: Fetch an expired SSH share
- **WHEN** the flag is on and a caller requests a nonce for, or the contents of, a `private/pk/` share whose expiry has passed but which the cleanup job has not yet removed
- **THEN** the server responds 404 Not Found, issues no nonce and returns no share

#### Scenario: Fetch an expired Smart-ID/Mobile-ID share
- **WHEN** the flag is on and an authenticated caller requests a nonce for, or the contents of, an `etsi/` share whose expiry has passed but which the cleanup job has not yet removed
- **THEN** the server responds 404 Not Found, issues no nonce and returns no share

#### Scenario: Upstream behaviour while off
- **WHEN** the flag is off and an authenticated caller requests a share whose expiry has passed but which the cleanup job has not yet removed
- **THEN** the server returns the share, as upstream
