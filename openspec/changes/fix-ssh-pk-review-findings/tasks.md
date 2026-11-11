# Tasks

Code for groups 1 and 2 is in cdoc2-shares-server, branch `feat/add-ssh-pk-recipient`.

## 1. Finding 2: default lifetime never exceeds the maximum (cdoc2-shares-server)

- [x] 1.1 Add a test class in the style of `RuhnuPkExpiryConfigTests` with the flag on, `cdoc2.ruhnu.pk.max-expiration-duration=P7D` and the default left unset (14 days). It creates a `ruhnu/pk/` share without `x-expiry-time` and asserts an expiry about 7 days from creation and `x-expiry-time-adjusted: false`. Verify: the test fails before the fix, with an expiry about 14 days from creation.
- [x] 1.2 In the `ruhnu/pk/` branch of `KeyShareApiService.getExpiryTime`, use the shorter of the default and the maximum as the default (design D1). Leave the global branch unchanged. Verify: the test from 1.1 passes, and `RuhnuPkExpiryConfigTests`, `RuhnuSshPkApiTests`, `RuhnuDisabledApiTests` and `KeyShareApiTests` still pass.
- [x] 1.3 Log one `WARN` at startup when the flag is on and the `ruhnu/pk/` default is longer than the maximum, naming both properties and their values and stating that the maximum is used. No warning with the flag off or with a default within the maximum. Verify: a test captures the log output and finds the warning in the misconfigured case and not in `RuhnuPkExpiryConfigTests`' configuration (default 1 day, maximum 7 days).
- [x] 1.4 In `admin-guide.md`, in the `cdoc2.ruhnu.pk.*` lifetimes bullet, state that a default longer than the maximum is replaced by the maximum and logged as a warning at startup. Add a CHANGELOG line if the unreleased entry lists the `cdoc2.ruhnu.pk.*` properties. Verify: the guide text matches the behaviour tested in 1.1 and 1.3.
- [x] 1.5 Run the shares-server build with checkstyle and spotbugs. Verify: `mvn verify` succeeds for the module.

## 2. Finding 1: malformed `aud` refused with 401 on the SSH path (cdoc2-shares-server)

- [x] 2.1 Add a parameterized test to `RuhnuSshPkApiTests` that sends validly signed `ruhnu/pk/` tokens whose `aud` nonce is `!!!`, `x&=`, `a&nonce=b` and `%%%`, and asserts 401 with no share and no audit log entry. Verify: before the fix the first three fail with 500.
- [x] 2.2 In `ValidateAuthToken.executeSshPk`, map every exception from parsing the `aud` URL, and from decoding the nonce as base64url, to `VerificationException` (design D2). Leave `execute`, `checkTokenAudience` and `checkNonceFromDB` unchanged. Verify: the test from 2.1 passes, and `shouldReturn404ForUnknownNonce` and `shouldReturn404ForNonceIssuedForAnotherShare` still pass.
- [x] 2.3 Run the shares-server build with checkstyle and spotbugs. Verify: `mvn verify` succeeds for the module.

## 3. Finding 3: `.pub` file with a non-ASCII comment (cdoc2-java-ref-impl)

Code is in cdoc2-java-ref-impl, branch `feat/add-ssh-pk-recipient`.

- [x] 3.1 Add a test to `SshPkKeySharesTest` that writes a key file with the comment `jüri@sülearvuti`, once as UTF-8 and once as ISO-8859-1, and asserts `EncryptionKeyMaterial.fromSshPublicKeyFile` gives the same recipient id as the plain key. Add a test to `SshPkCliTest` that runs `create --ssh-pubkey <file>` with such a file and asserts exit code 0 and the recipient in the create-share request. Verify: before the fix both fail with `MalformedInputException`.
- [x] 3.2 Read the public key file as ISO-8859-1 in `CDocCreateCmd.sshRecipient` and `EncryptionKeyMaterial.fromSshPublicKeyFile` (design D3). Verify: the tests from 3.1 pass.
- [x] 3.3 Build cdoc2-lib and cdoc2-cli with their tests and checkstyle. Verify: `mvn install` succeeds for both modules.

## Workflow follow-up

- Comment on ruhnu-openspec#1 with the outcome for each finding handled here.
- Archive the change after the cdoc2-shares-server PR is merged into `ruhnu`, so the delta lands in `openspec/specs/key-share-expiry/spec.md`.
