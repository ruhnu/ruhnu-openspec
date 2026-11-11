# Design

## Context

See `proposal.md` for motivation. Current state, read from local checkouts on 2026-11-10:

- **cdoc2-java-ref-impl** builds `cdoc2-cli` as a shaded jar (`maven-shade-plugin`, about 27 MB). Its only workflows are the upstream ones (`maven.yml`, `maven-release.yml`), which trigger on `master`. The fork's default branch is `ruhnu`, so neither runs for fork work.
- **Unreleased dependencies:** cdoc2-lib depends on `cdoc2-auth-token` 0.8.0-SNAPSHOT (for `SshPublicKeyParser`). cdoc2-client needs the key-shares OpenAPI 1.4.0-draft, and the key-capsules, auth-server and RP-server OpenAPI documents. All of these are built from the `feat/add-ssh-pk-recipient` branches of `cdoc2-auth` and `cdoc2-openapi`, which are not published to a Maven repository CI can reach.
- **The shares-server images** are published by `cdoc2-shares-server/.github/workflows/ruhnu-images.yml`:
  - It runs on `v*` and `dev-*` tags. Docker tags are the sanitised git tag, plus `latest` for `v*`.
  - It builds the sibling repos at refs pinned in `.github/ruhnu-deps.env`.
  - It sets OCI labels and index annotations.
  - The server image is built with Paketo buildpacks on one runner per architecture. The liquibase image is a Dockerfile with no `RUN` steps, built for both architectures on one runner.
- **What was tested on 2026-11-10** (Docker 29, Linux), with the CLI image `FROM bellsoft/liberica-runtime-container:jre-21-slim-musl` plus the jar:
  - The image is about 72 MB to download and 160 MB on disk. On `eclipse-temurin:21-jre` it was about 137 MB to download and about 350 MB on disk.
  - Against two shares servers, `create --ssh-pubkey`, `info` and `decrypt --ssh-agent` with the host agent socket mounted all worked as the host UID, including TLS with a mounted truststore.
  - A foreign UID is refused by the socket's file mode and by ssh-agent's own check of the caller's UID.
  - `decrypt --ssh-agent` asks the agent for its keys before it reads `key-shares.properties` or uses the network. With `--network none` and no configuration, a container encrypted to another key fails with `No matching key found in ssh-agent`.

## Goals / Non-Goals

**Goals:**
- The image and its workflow are fork-only additions. No CLI or library code changes.
- The workflow looks like the shares-server one, so that one person can maintain both.
- CI checks that the image reaches a mounted ssh-agent before anything is pushed.

**Non-Goals:**
- Running the cdoc2-java-ref-impl test suite in this workflow. As in the shares-server workflow, tests belong to branch CI.
- A full end-to-end test with shares servers in CI. That stays in ruhnu-demo.
- A smaller custom runtime (`jlink`) or a GraalVM native image.

## Decisions

### D1. Dockerfile on the Liberica musl JRE, not buildpacks

`cdoc2-cli/docker/Dockerfile`:

```dockerfile
FROM bellsoft/liberica-runtime-container:jre-21-slim-musl
COPY cdoc2-cli.jar /opt/cdoc2/cdoc2-cli.jar
ENV LANG=C.UTF-8
WORKDIR /work
ENTRYPOINT ["java", "-jar", "/opt/cdoc2/cdoc2-cli.jar"]
```

- It is the smallest of the JRE images measured, and the end-to-end run above used this exact image.
- It has no `RUN` steps, so buildx builds both architectures on one amd64 runner without emulation, as in the liquibase job. The jar is architecture-independent and is built once.
- Java 21 runs the CLI, which is compiled for Java 17.

Alternatives considered:
- **Paketo buildpacks**, as for the shares server: Spring Boot's `build-image` only applies to Spring Boot applications. Plain jars would need the `pack` CLI and one runner per architecture.
- **`eclipse-temurin:21-jre`** (Ubuntu): about twice the size, for no gain here.
- **`eclipse-temurin:21-jre-alpine`**: 74 MB compressed, against 50 MB for the Liberica image.
- **`jlink`**: smaller still, but it needs a list of JDK modules to maintain. It can come later if size matters.

The base is referenced by its floating tag `jre-21-slim-musl`, so each tag build picks up the current JRE 21 patch release.

### D2. The build context is a staging directory with a fixed jar name

The workflow copies `cdoc2-cli/target/cdoc2-cli-<version>.jar` to `cdoc2-cli/target/docker/cdoc2-cli.jar`, with the Dockerfile next to it, and builds from that directory. A glob such as `COPY target/cdoc2-cli-*.jar` would match several jars in a local `target/` that holds an older version. The local build documented in the README uses the same staging steps.

No `USER` is set. The user picks the UID with `--user $(id -u):$(id -g)`: it has to be the agent owner's UID anyway, and with that UID the files written to mounted directories belong to the user. `WORKDIR /work` gives the documented mount point.

### D3. Workflow `ruhnu-images.yml` in cdoc2-java-ref-impl

`.github/workflows/ruhnu-images.yml`, triggered by `push: tags: [ 'v*', 'dev-*' ]`. Like the shares-server workflow it uses `contents: read` and `packages: write`. It runs these jobs:

1. **`meta`:** the sanitised tag (`[^A-Za-z0-9_.-]` → `-`), `latest` only for `v*`, and one build time. This is the shares-server job, copied.
2. **`cli`**, on `ubuntu-latest`:
   - Check out the repo, and the siblings at the refs in `.github/ruhnu-deps.env`.
   - Build the siblings with `mvn install -Dmaven.test.skip=true -Dspotbugs.skip=true -Dcheckstyle.skip=true`.
   - Build the CLI with `mvn -pl cdoc2-cli -am install -DskipTests`, using JDK 17 like the repo's own CI. It has to be `-DskipTests`, not `-Dmaven.test.skip=true`: cdoc2-cli has a test-scoped dependency on cdoc2-lib's test jar, which `maven.test.skip` doesn't build, so dependency resolution fails in an empty Maven repo (found in task 2.1).
   - Stage the jar as in D2.
   - Build the amd64 image with `load: true`, then run the smoke test (D4).
   - Build and push both platforms with `docker/build-push-action`, with OCI `labels` and `index:` `annotations` as in the liquibase job. Buildx reuses the layers it built for amd64.

GitHub reads the workflow from the tagged commit. Upstream tags such as `v3.5.2` point at commits without this file, so mirroring upstream tags into the fork publishes nothing.

### D4. Smoke test against a CI-started ssh-agent, without shares servers

`cdoc2-cli/docker/smoke-test.sh <image>` runs in CI and locally:

1. `docker run --rm <image> --version` must print the `cdoc2-cli` version from the pom and exit 0.
2. Start a throwaway `ssh-agent` with a fresh P-256 key. Run `decrypt --ssh-agent` on the fixture `cdoc2-cli/docker/smoke-test/ssh-recipient.cdoc2`, as the runner's UID, with the socket mounted and `--network none`. It must exit non-zero with `No matching key found in ssh-agent`. That proves the CLI in the image connected to the agent and read its key list (verified on 2026-11-10).
3. Run the same without the socket. It must fail with `ssh-agent is not available`.

The fixture is a small container encrypted once to a throwaway SSH key using ruhnu-demo. The private key is discarded, so the fixture can never be decrypted and never needs a shares server.

A successful decryption is not tested in CI. It needs two shares servers and is covered by ruhnu-demo (task 3.2).

Alternative considered: run the ruhnu-demo stack in CI. That would make one workflow depend on three repos' server builds, a database and TLS material, just to repeat what step 2 shows about the image.

### D5. Pinned sibling refs, as in the shares server

`.github/ruhnu-deps.env` holds `CDOC2_OPENAPI_REF` and `CDOC2_AUTH_REF`. On the feature branch both are `feat/add-ssh-pk-recipient`. They must produce the versions the poms depend on. This is the shares-server mechanism and comment, copied, so both repos are updated together when the siblings are released.

### D6. Labels

These are the same keys as the shares-server images:
- `title=ruhnu-cli`
- `description`, for example: "cdoc2-cli for Ruhnu: encrypt secrets to SSH keys and decrypt them through ssh-agent. Fork of open-eid/cdoc2-java-ref-impl."
- `version=<image tag>`
- `revision=<commit>`
- `source=<repo URL>`
- `created`
- `documentation=https://github.com/ruhnu`

The CLI's own version is shown by `--version`; no separate label is added.

**Sibling commits.** The siblings are built from branch names (D5), so the image tag and `revision` alone don't say which cdoc2-auth and cdoc2-openapi code went in. After checking them out, the workflow reads `git rev-parse HEAD` in each checkout and adds:
- `io.github.ruhnu.cdoc2-auth.revision=<full sha>`
- `io.github.ruhnu.cdoc2-openapi.revision=<full sha>`

These are added both as labels and as `index:` annotations. The `io.github.ruhnu` prefix is the reverse DNS of the org's GitHub Pages domain, because custom keys need a namespace and the org has no domain of its own.

This records what was built. It doesn't control what gets built: a push to a sibling branch still changes the next tag's build (see Risks, and the follow-up in tasks.md).

Alternatives considered:
- **An SBOM:** it lists the siblings as `0.8.0-SNAPSHOT`, which doesn't identify a commit.
- **Signed provenance (`actions/attest-build-provenance`):** it can be verified, unlike labels. It's covered by the dependency-hardening follow-up.

## Risks / Trade-offs

- **[Siblings built from branch names]** → Anyone who can push to the sibling branches decides what the next tag publishes, and the token in that job can push packages. The quick threat assessment of 2026-11-10 rated this high, together with GitHub actions pinned by version tag and the floating base tag. This change only records the sibling commits (D6). Pinning is a follow-up for both image workflows (tasks.md).
- **[Floating base tag]** → Two builds of the same commit can differ in JRE patch level. That's accepted in exchange for security updates on every tag. The revision label plus `--version` identify the CLI, and the base digest is recorded in the image config.
- **[arm64 is not run in CI]** → Only the amd64 image is smoke-tested. The arm64 image differs only in the base image layers that BellSoft publishes, and the jar layer is identical. Check it once by hand on an arm64 host (task 3.3).
- **[Tests are skipped]** → A tag can publish code that fails its tests. This is the same trade-off as the shares-server workflow. Tag only commits whose branch CI passed.
- **[Mounting the agent socket]** → While the container runs, it can ask the agent to sign with any of its keys. The README recommends a dedicated agent or `ssh-add -c`, as `ssh-key-recipient` already notes for forwarded agents.
- **[Upstream tag names]** → Fork `v*` tags must not reuse upstream names such as `v3.5.2`, which already exist in the fork. See Open Questions.

## Migration Plan

- Nothing to migrate. The first `dev-*` tag creates the `ruhnu-cli` package on ghcr.io, private by default. Make it public once by hand, as was done for the shares-server packages.
- Rollback: delete the package version on ghcr.io, and delete the tag if needed.

## Open Questions

- The naming scheme for fork release tags (`v*`) in cdoc2-java-ref-impl, for example `v1.10.0-ruhnu.1`, as long as it doesn't collide with upstream `v*` tags. Until it's decided, use `dev-*` tags. The workflow and spec don't depend on the scheme.
