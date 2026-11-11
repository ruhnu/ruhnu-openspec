# cli-container-image Specification

## Purpose

Publishes `cdoc2-cli` as the container image `ghcr.io/ruhnu/ruhnu-cli`, so that users can encrypt to and decrypt with SSH public key recipients through Docker, with the host's ssh-agent, without installing Java or building the fork.

## Requirements

### Requirement: Image published on tag
Pushing a tag `v*` or `dev-*` to `ruhnu/cdoc2-java-ref-impl` SHALL publish `ghcr.io/ruhnu/ruhnu-cli` built from the tagged commit, with the image tag equal to the git tag. Characters not allowed in image tags SHALL be replaced with `-`. A `v*` tag SHALL also move `latest`; a `dev-*` tag SHALL NOT. Branch pushes and other tags SHALL NOT publish an image.

#### Scenario: Release tag
- **WHEN** tag `v1.0.0` is pushed
- **THEN** `ghcr.io/ruhnu/ruhnu-cli:v1.0.0` is published, and `ghcr.io/ruhnu/ruhnu-cli:latest` points to the same image

#### Scenario: Development tag
- **WHEN** tag `dev-ssh-pk-1` is pushed
- **THEN** `ghcr.io/ruhnu/ruhnu-cli:dev-ssh-pk-1` is published, and `latest` is unchanged

#### Scenario: Tag with a slash
- **WHEN** tag `dev-feat/x` is pushed
- **THEN** the image is published as `ghcr.io/ruhnu/ruhnu-cli:dev-feat-x`

#### Scenario: Branch push or other tag
- **WHEN** a branch is pushed, or a tag that starts with neither `v` nor `dev-`
- **THEN** no image is published

### Requirement: Multi-arch image
Each published image tag SHALL provide linux/amd64 and linux/arm64 images, so that `docker run` picks the host's architecture without the user naming a platform.

#### Scenario: Pull on arm64
- **WHEN** a user on a linux/arm64 host runs `docker run --rm ghcr.io/ruhnu/ruhnu-cli:<tag> --version`
- **THEN** the arm64 image runs natively and prints the CLI version

#### Scenario: Both platforms listed
- **WHEN** the manifest of a published tag is inspected
- **THEN** it lists linux/amd64 and linux/arm64

### Requirement: CLI is the entrypoint
The image SHALL run `cdoc2-cli` with the arguments given to `docker run`, unchanged, and the container SHALL exit with the CLI's exit code. `-D<property>=<value>` arguments SHALL be accepted as they are by `java -jar cdoc2-cli.jar`.

#### Scenario: Version
- **WHEN** a user runs `docker run --rm ghcr.io/ruhnu/ruhnu-cli:<tag> --version`
- **THEN** the output shows the `cdoc2-cli` and `cdoc2-lib` versions built from the tagged commit, and the exit code is 0

#### Scenario: Failing command
- **WHEN** a user runs `decrypt` on a file that does not exist
- **THEN** the container exits with a non-zero exit code

### Requirement: Image identifies its source
Each published image SHALL carry OCI labels and index annotations naming the title `ruhnu-cli`, a description, the image tag as version, the git commit as revision, the source repository URL and the build time. It SHALL also name the exact commit of each sibling repository built into it (`cdoc2-auth`, `cdoc2-openapi`).

#### Scenario: Inspect a published image
- **WHEN** a user inspects `ghcr.io/ruhnu/ruhnu-cli:<tag>`
- **THEN** the version is `<tag>` and the revision is the commit the tag points to

#### Scenario: Sibling commits recorded
- **WHEN** a user inspects `ghcr.io/ruhnu/ruhnu-cli:<tag>`
- **THEN** it names the full commit SHA of `cdoc2-auth` and of `cdoc2-openapi` that the build checked out, and checking out those commits with the tagged commit rebuilds the same CLI jar contents

### Requirement: Encrypt to an SSH key in the container
With the user's files and the key-shares configuration mounted, `create --ssh-pubkey` in the container SHALL behave as `ssh-key-recipient` specifies for the CLI. Paths in the arguments and in the configuration refer to the container's file system. No ssh-agent is needed to encrypt.

#### Scenario: Encrypt with mounted configuration
- **WHEN** a user mounts a directory with `key-shares.properties` and the truststore it names, mounts the working directory, and runs `create` with `--ssh-pubkey` and `-Dkey-shares.properties=<container path>`
- **THEN** shares are uploaded to every configured shares server and the container file is written to the mounted directory

### Requirement: Decrypt through the host ssh-agent
When the host's ssh-agent socket is mounted into the container and `SSH_AUTH_SOCK` names it, `decrypt --ssh-agent` in the container SHALL sign through that agent and decrypt as `ssh-key-recipient` specifies for the CLI. The agent's private keys SHALL NOT be needed in the container.

#### Scenario: Decrypt with the mounted agent
- **WHEN** a user runs the image as their own UID with the agent socket mounted and `SSH_AUTH_SOCK` set, and the agent holds the key of the container's SSH recipient
- **THEN** the CLI fetches every share and writes the decrypted files to the mounted output directory

#### Scenario: Agent socket not mounted
- **WHEN** a user runs `decrypt --ssh-agent` in the image without `SSH_AUTH_SOCK` set
- **THEN** the CLI fails with an error stating that ssh-agent is not available, and no request is sent to any shares server

#### Scenario: Agent does not hold the key
- **WHEN** the mounted agent does not hold the key of any SSH recipient of the container
- **THEN** the CLI fails with an error stating that no matching key was found in ssh-agent, and no request is sent to any shares server

#### Scenario: Container runs as another user
- **WHEN** the container runs as a UID other than the agent owner's and other than root
- **THEN** the CLI fails with an ssh-agent error and no auth token is sent to any shares server

### Requirement: Files written as the invoking user
The image SHALL run correctly as an arbitrary non-root UID and GID given with `docker run --user`, so that files the CLI writes to a mounted directory are owned by that user.

#### Scenario: Output ownership
- **WHEN** a user runs `decrypt` with `--user $(id -u):$(id -g)` and the output directory mounted
- **THEN** the decrypted files on the host are owned by that user

### Requirement: Broken images are not published
A tag SHALL NOT be published if the image built for it cannot print the CLI version, or cannot reach a mounted ssh-agent from `decrypt --ssh-agent`.

#### Scenario: Smoke test fails
- **WHEN** the image built for a tag fails to print the CLI version, or `decrypt --ssh-agent` with a mounted agent does not reach the agent
- **THEN** no image is published for that tag, and the CI run fails
