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

## 4. Finding 8: recipient checks with the flag on (cdoc2-shares-server)

- [x] 4.1 Add a test to `RuhnuSshPkApiTests` that creates shares for `ruhnu/pK/<valid blob>`, `ruhnu/xyz/whatever` and a 42-character `etsi/` recipient and asserts 400 for each and nothing stored. Verify: before the fix the first is accepted with 201.
- [x] 4.2 In `KeyShareApiService.createKeyShare`, refuse other `ruhnu/` kinds and apply the 32-character limit to every non-`ruhnu/pk/` recipient (design D5). Add a line to `admin-guide.md`. Verify: the test from 4.1 passes, the flag-off tests in `RuhnuDisabledApiTests` still pass, and `mvn verify` succeeds with checkstyle and spotbugs.

## 5. Finding 5: `--ssh-agent` flag and `--ssh-agent-socket` (cdoc2-java-ref-impl)

- [x] 5.1 Add a test to `SshPkCliTest` that runs `decrypt ... --ssh-agent payload.txt` and asserts the error does not say it cannot connect to `payload.txt`. Verify: before the fix it does.
- [x] 5.2 Make `--ssh-agent` a flag and add `--ssh-agent-socket <path>` in `DecryptionKeyExclusiveArgument` (design D4); switch the existing tests from `--ssh-agent=<socket>` to `--ssh-agent-socket=<socket>`; update the CLI README. Verify: all `SshPkCliTest` tests pass.

## 6. Finding 9: empty value in the ssh-agent signature (cdoc2-java-ref-impl)

- [x] 6.1 Add a test to `SshAgentClientTest` that passes a signature blob with an empty `r`, then an empty `s`, to `SshAgentJWSSigner.toJwsSignature` and asserts `JOSEException("Malformed ssh-agent ECDSA signature")`. Verify: before the fix it fails with `NumberFormatException`.
- [x] 6.2 Catch `NumberFormatException` in `toJwsSignature` (design D6). Verify: the test from 6.1 and all `SshAgentClientTest` tests pass.

## 7. Finding 10: requested share lifetime checked and reported (cdoc2-java-ref-impl)

- [x] 7.1 Add a test to `SshPkCliTest` that runs `create` with `--share-expiry PT0S` and `-P1D` and asserts a non-zero exit code, an error saying the duration must be positive, no stack trace and no create-share request. Verify: before the fix `PT0S` is accepted.
- [x] 7.2 Add `PositiveDurationConverter` for `--share-expiry`, and a check in `SshPkEncryptionKeyMaterial` (design D7). Add a test to `SshPkKeySharesTest` that `fromSshPublicKey` refuses a zero and a negative lifetime. Verify: the tests pass.
- [x] 7.3 In `Cdoc2KeySharesApiClient.createKeyShare`, log a warning naming the server and the applied expiry when the response reports `x-expiry-time-adjusted: true` (design D7). Add a test to `SshPkKeySharesTest` where one of two stubbed servers reports an adjusted expiry, asserting exactly one warning with that server and expiry. Update the CLI README. Verify: the test passes. (The warning was first observed missing through a CLI test, which could not capture log output and was replaced by this one.)
- [x] 7.4 Build cdoc2-client, cdoc2-lib and cdoc2-cli with their tests and checkstyle. Verify: `mvn install` succeeds for all three.

## Workflow follow-up

- Comment on ruhnu-openspec#1 with the outcome for each finding handled here.
- Archive the change after the cdoc2-shares-server and cdoc2-java-ref-impl PRs are merged into `ruhnu`, so the deltas land in the main specs.
