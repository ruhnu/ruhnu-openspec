# Design

## Context

See `proposal.md` for motivation. The current code in `cdoc2-shares-server` (read on 2026-11-07):

- **Nonce:** `KeyShareApiService.createNonce` validates the session token, then calls `keyShareRepository.findById(shareId)` and responds 404 if the row is missing.
- **Get-share:** `getKeyShareByShareId` validates the session token and the auth token, including the audience and nonce checks, then calls `findById` and responds 404 if the row is missing or the recipient does not match.
- **Expiry:** `key_material_share.expiry_time` is set at creation. Rows are deleted by `ExpiredKeyShareCleanUpJob`, which calls the DB function `expired_key_material_share_cleanup()` on a cron schedule. Neither endpoint reads `expiry_time`.

## Goals / Non-Goals

**Goals:**
- A share stops being served at its expiry, independent of the cleanup schedule.
- Status codes and check order stay as they are; an expired share is indistinguishable from a deleted one.

**Non-Goals:**
- Changing the cleanup job, its schedule or the DB function.
- Changing how expiry is set at creation, or its configuration.

## Decisions

### Expired is treated as not found, at the existing lookup

Both endpoints replace "row missing → 404" with "row missing or expired → 404", in the same place. Authentication still runs first, so callers learn nothing they could not learn today. Once the cleanup job runs, the response is identical anyway.

**Alternatives considered:**
- **401 or 403:** misleading, because the caller may be properly authenticated. It would also tell callers apart from the post-cleanup 404, which leaks the cleanup schedule.
- **410 Gone:** more precise, but not in the API document, and it differs from what clients see after cleanup.

### Filter in application code via one shared lookup helper

A small helper in `KeyShareApiService` returns `Optional.empty()` when `expiry_time` is not after the current time. Both endpoints call it instead of `findById`. This keeps the change in one place and easy to review.

A repository query such as `findByShareIdAndExpiryTimeAfter` is an equally valid alternative. Choose whichever reads better in review; behaviour is the same.

### Time source

The check uses the application clock (the injected `Clock` bean, which tests can control), while the cleanup function uses the DB clock. Small clock skew only shifts the cut-off by the skew; it does not reopen expired shares for longer than today's behaviour already allows. The comparison is `now >= expiry_time → expired`.

## Risks / Trade-offs

- **[Clients that relied on the grace period]** → An expired share now fails immediately. This is the documented meaning of expiry; the old grace period was never specified. Mention it in the CHANGELOG.
- **[Clock skew between app and DB]** → Bounded by the skew; see Time source.

## Migration Plan

No data or configuration migration. Deploying the new version enables the check; rolling back restores the old grace period.
