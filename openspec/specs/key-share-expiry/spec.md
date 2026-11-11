# key-share-expiry Specification

## Purpose

Defines how long the shares server keeps a key share available, so that a share stops working at its expiry time regardless of when expired rows are physically removed.

## Requirements

### Requirement: Expired shares cannot be fetched
The shares server SHALL treat a share whose expiry time has passed as not found, for all recipient types, even if the cleanup job has not yet removed it. The nonce and get-share endpoints SHALL respond 404 Not Found at the point where they look up the share, as they do for a deleted share.

#### Scenario: Get an expired share
- **WHEN** an authenticated caller requests the contents of a share whose expiry has passed but which the cleanup job has not yet removed
- **THEN** the server responds 404 Not Found and does not return the share

#### Scenario: Nonce for an expired share
- **WHEN** an authenticated caller requests a nonce for a share whose expiry has passed but which the cleanup job has not yet removed
- **THEN** the server responds 404 Not Found and creates no nonce

#### Scenario: Share expires between nonce and fetch
- **WHEN** a caller obtained a nonce while the share was valid, and requests the share after its expiry has passed
- **THEN** the server responds 404 Not Found and does not return the share

#### Scenario: Unexpired share still served
- **WHEN** an authenticated caller requests a nonce for, or the contents of, a share whose expiry is in the future
- **THEN** the server behaves as before this change

#### Scenario: Share without an expiry time still served
- **WHEN** an authenticated caller requests a nonce for a share that has no expiry time, because it was created before expiry times were recorded
- **THEN** the server behaves as before this change, and the share does not expire

#### Scenario: Authentication failures unchanged
- **WHEN** an unauthenticated or wrongly authenticated caller requests an expired share
- **THEN** the server responds with the same status it would give for an unexpired share, because authentication is checked before the share is looked up

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
