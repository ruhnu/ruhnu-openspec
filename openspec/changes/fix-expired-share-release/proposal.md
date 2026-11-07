# Proposal

## Why

A key share has an expiry time, but the shares server does not check it when serving requests. `POST /key-shares/{shareId}/nonce` and `GET /key-shares/{shareId}` keep working for an expired share until the scheduled `ExpiredKeyShareCleanUpJob` deletes the row. So the real lifetime of a share is its expiry plus up to one cleanup interval, which the client cannot see and the server operator does not directly control. This undermines any use of short share lifetimes.

This is a standalone bug fix, meant to be offered upstream on its own. It is done before `add-ssh-pk-recipient`, which relies on shares ending exactly at their expiry.

## What Changes

- The nonce and get-share endpoints treat a share whose `expiry_time` has passed as not found, and respond **404 Not Found**. This is the same status, at the same point in processing, as for a share the cleanup job has already deleted.
- The check applies to all recipient types and is always on. There is no configuration switch.
- Share creation, the cleanup job and the expiry configuration are unchanged.

## Capabilities

### New Capabilities

- `key-share-expiry`: how long the shares server keeps a key share available. This change covers refusing shares after their expiry. `add-ssh-pk-recipient` later adds per-recipient-type lifetimes to the same capability.

### Modified Capabilities

None. The store has no specs yet.

## Impact

- **cdoc2-shares-server:** share lookup in `KeyShareApiService` for the nonce and get-share operations; integration tests; CHANGELOG.
- **API:** no change to `cdoc2-key-shares-openapi.yaml`. 404 is already a documented response for both operations.
- **Clients:** a client fetching an expired share now gets 404 straight away rather than the share. Containers whose shares have expired could already stop opening at any moment once cleanup ran, so this makes existing behaviour exact rather than introducing new failures.
- **Upstream:** a small, self-contained change suitable for an upstream merge request.
