# Proposal

## Why

To use the SSH public key recipient (`ssh-key-recipient`), a user today has to build `cdoc2-cli` from the fork and run it with a local Java runtime. A published container image lets anyone with Docker encrypt and decrypt with one `docker run`, with the host's ssh-agent. The shares server is already published this way (`ghcr.io/ruhnu/ruhnu-shares-server`), so the client should be too.

An investigation on 2026-11-10 showed that this works without changes to the CLI. The shaded `cdoc2-cli` jar on `bellsoft/liberica-runtime-container:jre-21-slim-musl` encrypted and decrypted end to end against two shares servers, and signed through a host ssh-agent socket mounted into the container. The image was about 72 MB to download and 160 MB on disk.

## What Changes

- Pushing a tag to `ruhnu/cdoc2-java-ref-impl` builds and publishes the container image `ghcr.io/ruhnu/ruhnu-cli`. A `v*` tag publishes `:<tag>` and `:latest`; a `dev-*` tag publishes `:<tag>` only. This follows the shares-server images.
- The image is multi-arch (linux/amd64 and linux/arm64) under one tag.
- The image records the exact commits of the sibling repos built into it (`cdoc2-auth`, `cdoc2-openapi`). The siblings are built from branch names, so the tag alone doesn't identify them. Pinning those inputs is a follow-up.
- The image runs `cdoc2-cli` as its entrypoint, so `docker run ghcr.io/ruhnu/ruhnu-cli <args>` takes the same arguments as `java -jar cdoc2-cli.jar <args>`.
- The supported use is the SSH public key recipient: encrypt to an SSH key, and decrypt through the host ssh-agent with its socket mounted into the container.
- `cdoc2-cli/README-ruhnu.md` documents how to run the image, including the agent socket, the container user and a `ruhnu` shell function. The fork-only CLI documentation (SSH recipients and the image) lives in that file, and `cdoc2-cli/README.md` only links to it, to keep upstream merges cheap.
- **Not in scope:**
  - Smart-ID, Mobile-ID and PKCS#11/ID-card in the image. The jar still contains that code, but the image doesn't have to support it, and it isn't tested there.
  - A GraalVM native image.
  - Clearer ssh-agent error messages. When ssh-agent itself refuses a foreign UID, the CLI says only `ssh-agent communication failed`; that is a separate fix in `SshAgentClient`.
  - Publishing the cdoc2 jars to a Maven repository.

## Capabilities

### New Capabilities

- `cli-container-image`: the published `ruhnu-cli` container image. Covers when it is published and under which tags, the supported platforms, how arguments, files and the ssh-agent reach the CLI in the container, and which recipient types the image supports.

### Modified Capabilities

None. `ssh-key-recipient` describes what the CLI does, and that is unchanged; the image only packages it.

## Impact

- **cdoc2-java-ref-impl** (fork only, nothing for upstream):
  - a new Dockerfile for `cdoc2-cli`
  - a new GitHub Actions workflow triggered by `v*` and `dev-*` tags, with a pinned-refs file for the sibling repos it builds first (`cdoc2-openapi`, `cdoc2-auth`), as in cdoc2-shares-server
  - a test fixture for the image smoke test
  - `cdoc2-cli/README-ruhnu.md`, with the SSH recipient documentation moved there from `cdoc2-cli/README.md`
- **GitHub:** a new package `ruhnu-cli` in the `ruhnu` organisation on ghcr.io. Its visibility is set once by hand, as for the shares-server packages.
- **CLI code and other repos:** no change.
