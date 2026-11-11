# Tasks

Code is in cdoc2-shares-server, branch `feat/add-ssh-pk-recipient`.

## 1. Finding 2: default lifetime never exceeds the maximum (cdoc2-shares-server)

- [x] 1.1 Add a test class in the style of `RuhnuPkExpiryConfigTests` with the flag on, `cdoc2.ruhnu.pk.max-expiration-duration=P7D` and the default left unset (14 days). It creates a `ruhnu/pk/` share without `x-expiry-time` and asserts an expiry about 7 days from creation and `x-expiry-time-adjusted: false`. Verify: the test fails before the fix, with an expiry about 14 days from creation.
- [x] 1.2 In the `ruhnu/pk/` branch of `KeyShareApiService.getExpiryTime`, use the shorter of the default and the maximum as the default (design D1). Leave the global branch unchanged. Verify: the test from 1.1 passes, and `RuhnuPkExpiryConfigTests`, `RuhnuSshPkApiTests`, `RuhnuDisabledApiTests` and `KeyShareApiTests` still pass.
- [x] 1.3 Log one `WARN` at startup when the flag is on and the `ruhnu/pk/` default is longer than the maximum, naming both properties and their values and stating that the maximum is used. No warning with the flag off or with a default within the maximum. Verify: a test captures the log output and finds the warning in the misconfigured case and not in `RuhnuPkExpiryConfigTests`' configuration (default 1 day, maximum 7 days).
- [x] 1.4 In `admin-guide.md`, in the `cdoc2.ruhnu.pk.*` lifetimes bullet, state that a default longer than the maximum is replaced by the maximum and logged as a warning at startup. Add a CHANGELOG line if the unreleased entry lists the `cdoc2.ruhnu.pk.*` properties. Verify: the guide text matches the behaviour tested in 1.1 and 1.3.
- [x] 1.5 Run the shares-server build with checkstyle and spotbugs. Verify: `mvn verify` succeeds for the module.

## Workflow follow-up

- Comment on ruhnu-openspec#1 with the outcome for each finding handled here.
- Archive the change after the cdoc2-shares-server PR is merged into `ruhnu`, so the delta lands in `openspec/specs/key-share-expiry/spec.md`.
