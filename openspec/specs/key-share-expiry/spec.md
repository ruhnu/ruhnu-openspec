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
