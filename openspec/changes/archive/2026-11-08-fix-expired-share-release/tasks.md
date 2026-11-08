# Tasks

## 1. Refuse expired shares (cdoc2-shares-server)

- [x] 1.1 Add a lookup helper in `KeyShareApiService` that treats a share with `expiry_time <= now` as absent, and use it in place of `findById` in the nonce and get-share operations, keeping authentication before lookup. Verify: the project builds, including checkstyle and spotbugs, and the existing test suite passes unchanged.
- [x] 1.2 Add integration tests (alongside `KeyShareIntegrationTest`/`KeyShareApiTests`) for each scenario in `specs/key-share-expiry/spec.md`: an expired-but-not-cleaned-up share gives 404 on nonce and on get-share with valid authentication; a nonce obtained before expiry and used after expiry gives 404; an unexpired share is still served; a request for an expired share with invalid authentication gets the same status as for an unexpired one. Verify: the new tests pass, and fail when the helper from 1.1 is bypassed.
- [x] 1.3 Add a CHANGELOG entry under a new unreleased version, describing the fix and that expired shares are no longer served during the cleanup grace period. Verify: the entry is present and matches the behaviour tested in 1.2.

## Workflow follow-up

- Offer the fix upstream as its own merge request.
- Archive this change before applying `add-ssh-pk-recipient`, so that `key-share-expiry` exists as a main spec when that change adds to it.
