# Tasks

All work is in `cdoc2-java-ref-impl`, on a topic branch. Pushing tags and publishing to ghcr.io are outward-facing: ask the user before each push.

## 1. Image and smoke test (local)

- [x] 1.1 Add `cdoc2-cli/docker/Dockerfile` (design D1), with no `USER` and `WORKDIR /work`. Stage the shaded jar as `cdoc2-cli/target/docker/cdoc2-cli.jar` with the Dockerfile next to it (D2), and build the image locally. Verify: `docker run --rm <image> --version` prints the `cdoc2-cli` version from the pom and exits 0, and a failing command (`decrypt -f /nonexistent`) exits non-zero.
- [x] 1.2 Create the fixture `cdoc2-cli/docker/smoke-test/ssh-recipient.cdoc2` with ruhnu-demo: encrypt a short text file to a fresh throwaway P-256 key, then delete the private key. Verify: `info -f` on the fixture shows one SSH public key recipient, and the fixture is under 10 KB.
- [x] 1.3 Add `cdoc2-cli/docker/smoke-test.sh <image>` (D4): check `--version`; with a throwaway agent and `--network none`, expect `No matching key found in ssh-agent` and a non-zero exit; without the socket, expect `ssh-agent is not available`. Verify: the script passes against the local image, and fails when the agent socket mount is removed from the second check.
- [x] 1.4 Add a "Docker image" section to `cdoc2-cli/README-ruhnu.md`. (2026-11-10, on the user's request: the SSH recipient section was also moved there from `cdoc2-cli/README.md`, which now only links to it, and the shell function was renamed `ruhnu`.) The section contains:
  - `docker run` for `create --ssh-pubkey` and `decrypt --ssh-agent`, with `--user $(id -u):$(id -g)`, the agent socket at `/ssh-agent.sock`, `SSH_AUTH_SOCK`, and the mounted working and config directories
  - a `ruhnu` shell function
  - the note that only SSH recipients are supported in the image
  - the advice on dedicated agents and `ssh-add -c`
  - the local build steps from 1.1

  Verify: against ruhnu-demo (`./up.sh`), the documented `create` and `decrypt` commands run as written with the local image. The decrypted file matches, it is owned by the host user, and both servers log the release.

## 2. Publishing workflow

- [x] 2.1 Add `.github/ruhnu-deps.env`, with `CDOC2_OPENAPI_REF` and `CDOC2_AUTH_REF` set to `feat/add-ssh-pk-recipient` and the shares-server comment (D5). Verify: a clean checkout of both refs builds, and `mvn -pl cdoc2-cli -am install -DskipTests` then builds the CLI against them in an empty local Maven repo (`-Dmaven.repo.local=<tmp>`).
- [x] 2.2 Add `.github/workflows/ruhnu-images.yml` (D3, D6):
  - triggered by `v*` and `dev-*` tags
  - a `meta` job with the sanitised tag, `latest` only for `v*`, and one build time
  - build the siblings, then the CLI, then stage the jar
  - build an amd64 image with `load: true` and run `smoke-test.sh`
  - build and push linux/amd64 and linux/arm64 with OCI labels and `index:` annotations, including the sibling commits read from the checkouts (`io.github.ruhnu.cdoc2-auth.revision`, `io.github.ruhnu.cdoc2-openapi.revision`)

  Verify: `actionlint`, if available, reports nothing, and a review against the shares-server workflow finds no unexplained differences. (2026-11-10: actionlint was not available in the sandbox. The workflow was parsed with SnakeYAML and all step and job output references resolve; the diff review found only the D1, D3 and D6 differences and `-DskipTests`.)
- [x] 2.3 With the user's go-ahead, push the branch and a tag `dev-cli-image-1` to `ruhnu/cdoc2-java-ref-impl`. Verify:
  - the workflow passes
  - `docker buildx imagetools inspect ghcr.io/ruhnu/ruhnu-cli:dev-cli-image-1` lists linux/amd64 and linux/arm64, with version `dev-cli-image-1` and the tagged commit as revision
  - the sibling revision labels and annotations equal `git ls-remote` of `feat/add-ssh-pk-recipient` in ruhnu/cdoc2-auth and ruhnu/cdoc2-openapi at the time of the run
  - `latest` does not exist or is unchanged
  - a branch push with no tag starts no run of this workflow
- [x] 2.4 Check that a broken image is not published: on a throwaway tag `dev-cli-image-broken`, make the smoke test fail, for example with a wrong expected message in a commit on a scratch branch. Verify: the run fails, and no `dev-cli-image-broken` image exists on ghcr.io. Afterwards, delete the tag and the scratch branch.

## 3. Integration checks with the published image

- [x] 3.1 Make the `ruhnu-cli` package public on ghcr.io (by hand, done by the user). Verify: an anonymous `docker pull ghcr.io/ruhnu/ruhnu-cli:dev-cli-image-1` succeeds after `docker logout ghcr.io`.
- [x] 3.2 Run the README commands from 1.4 against ruhnu-demo with the published image instead of the local one. Verify: encryption, decryption through the host agent, the audit lines and the "agent socket not mounted" error all behave as in 1.4.
- [x] 3.3 On an arm64 host, if one is available, run `smoke-test.sh` against the published tag. Verify: the script passes, and `docker image inspect` shows `arm64`. If no arm64 host is available, record that in this task and leave it unchecked. (2026-11-10: postponed by the user until after the demo.) Done 2026-11-11 by the user on an on-premises arm64 (`aarch64`) server, with `smoke-test.sh` from branch `ruhnu`, for `ghcr.io/ruhnu/ruhnu-cli:dev-cli-image-1` and `:dev-did-1`. Both images inspect as `linux/arm64`, print `cdoc2-cli version: 1.10.0-SNAPSHOT`, report "No matching key found in ssh-agent" with a mounted agent and "ssh-agent is not available" without one, and pass the smoke test.

## Workflow follow-up

- Decide the fork release tag scheme for `v*` tags (design, Open Questions) before the first `v*` tag.
- When the siblings' `feat/add-ssh-pk-recipient` branches are merged or released, update `.github/ruhnu-deps.env` in cdoc2-java-ref-impl and cdoc2-shares-server together. Do it on the feature branches before their PRs merge into `ruhnu`, after the cdoc2-openapi and cdoc2-auth PRs have merged, so that `ruhnu` never carries feature-branch refs.
- Decide how to pin the build inputs of both image workflows (ruhnu-cli and ruhnu-shares-server) properly, as a separate change. The quick threat assessment of 2026-11-10 rated unpinned inputs high risk. Options discussed on 2026-11-10:
  - **Sibling repos** (cdoc2-auth, cdoc2-openapi), from simplest to strongest:
    - Branch plus SHA in `ruhnu-deps.env`, rewritten by a script that runs `git ls-remote` on each branch before tagging. CI checks out the SHA and checks that it is on the named branch. Recommended first step.
    - Matching tags in all three repos, protected by an org tag ruleset.
    - Git submodules with `branch =`. Not recommended: friction with upstream merges.
    - Publish the siblings to GitHub Packages (Maven) with GITHUB_TOKEN and depend on released versions. The long-term answer once the siblings stabilise.
  - **GitHub actions:** pin `actions/*` and `docker/*` by commit SHA instead of `@vN`.
  - **Base images:** pin by digest, with an update mechanism (Dependabot or Renovate) so JRE security fixes still arrive.
  - **Verifiability:** signed provenance (`actions/attest-build-provenance`), including the sibling commits, plus an SBOM. Optionally, split build and push into jobs with different token permissions.
- Archive the change after review.
