# ssh-key-recipient Specification

## Purpose

Lets a CDOC2 container be encrypted to an SSH public key through the key-shares scheme, and decrypted by whoever holds the matching private key in an ssh-agent, without Smart-ID, Mobile-ID, an RP server or an auth server.

## Requirements

### Requirement: SSH public key recipient identifier
An SSH public key recipient SHALL be stored as a key-shares recipient with the recipient type `RUHNU` and a `recipient_id` of the form `ruhnu/pk/<blob>`. `RUHNU` covers all `ruhnu/<kind>/...` identifiers, and the kind SHALL be determined by the `recipient_id` prefix. `<blob>` is the standard base64 (with padding) of the OpenSSH public key wire blob, as in the second field of an OpenSSH `.pub` line. The same key SHALL always produce the same identifier.

#### Scenario: Identifier built from an OpenSSH public key line
- **WHEN** the user supplies `ecdsa-sha2-nistp256 AAAAE2VjZHNh... user@host` as the recipient key
- **THEN** the recipient is stored with the recipient type `RUHNU` and `recipient_id` `ruhnu/pk/AAAAE2VjZHNh...`, without the key type prefix or comment

#### Scenario: Identifier is independent of the comment
- **WHEN** the same public key is supplied twice with different comments
- **THEN** both produce the same `recipient_id`

### Requirement: Supported SSH key types
The SSH public key recipient SHALL support ECDSA P-256 (`ecdsa-sha2-nistp256`) keys. Any other key type SHALL be rejected when encrypting, with an error naming the key type as unsupported.

#### Scenario: P-256 key accepted
- **WHEN** the user encrypts to an `ecdsa-sha2-nistp256` public key
- **THEN** the container is created with an SSH public key recipient for that key

#### Scenario: Ed25519 key rejected
- **WHEN** the user encrypts to an `ssh-ed25519` public key
- **THEN** encryption fails before any share is uploaded, with an error stating that `ssh-ed25519` is not supported

#### Scenario: Malformed key rejected
- **WHEN** the user supplies a recipient key whose blob is not valid base64 or does not parse as an OpenSSH public key
- **THEN** encryption fails before any share is uploaded, with an error stating the key could not be parsed

### Requirement: Encrypt to an SSH public key
The CLI SHALL let the user encrypt a container to one or more SSH public keys given as OpenSSH public key lines. For each key it SHALL split the key encryption key into shares, one per configured shares server, each with that key's `recipient_id` as recipient.

#### Scenario: Encrypt to a key from a file
- **WHEN** the user creates a container with an SSH recipient given as a path to a `.pub` file
- **THEN** shares are uploaded to every configured shares server with recipient `ruhnu/pk/<blob>`, and the container's key-shares recipient lists those shares

#### Scenario: Encrypt to a key given as text
- **WHEN** the user creates a container with an SSH recipient given as an OpenSSH public key line, such as the output of `ssh-add -L`
- **THEN** the result is the same as when the key is given as a file

### Requirement: Decrypt with ssh-agent
The client SHALL decrypt an SSH public key recipient by signing each per-share auth token through the ssh-agent at `$SSH_AUTH_SOCK`, using the key named in the recipient. Decryption SHALL NOT require a private key file, Smart-ID, Mobile-ID, an RP server or an auth server.

#### Scenario: Successful decryption through the agent
- **WHEN** the agent holds the private key matching a container's SSH recipient and the user decrypts the container
- **THEN** the client fetches every share, reconstructs the key and decrypts the container

#### Scenario: Agent does not hold the key
- **WHEN** the agent is reachable but does not list the public key named in any SSH recipient of the container
- **THEN** decryption fails with an error stating that no matching key was found in ssh-agent, and no auth token is sent to any shares server

#### Scenario: No agent available
- **WHEN** `$SSH_AUTH_SOCK` is unset or the socket cannot be connected
- **THEN** decryption fails with an error stating that ssh-agent is not available

#### Scenario: Unknown ruhnu recipient kind
- **WHEN** the container has a `RUHNU` recipient whose `recipient_id` starts with `ruhnu/` but not with a kind this client supports, such as `ruhnu/xyz/...`
- **THEN** decryption with that recipient fails with an error stating the recipient kind is unsupported, and no request is sent to any shares server

#### Scenario: Agent refuses to sign
- **WHEN** the agent refuses a sign request, for example because the user declined an `ssh-add -c` confirmation
- **THEN** decryption fails with an error stating that ssh-agent refused to sign

### Requirement: Auth token signed through ssh-agent
For each share, the client SHALL send an SD-JWT auth token whose JWS uses `alg` `ES256`, whose `iss` equals the recipient's `recipient_id`, and whose `aud` disclosure names that share's URL and server nonce. The JWS SHALL verify with the recipient's public key under standard JWS rules.

#### Scenario: Token verifies against the recipient key
- **WHEN** the client creates an auth token for a share of an SSH recipient
- **THEN** the token's JWS header has `alg` `ES256`, `iss` equals the `recipient_id`, and the signature verifies with the P-256 public key decoded from that `recipient_id`

### Requirement: Requested share lifetime
When encrypting to an SSH public key, the CLI SHALL let the user request a share lifetime. If the user gives none, the CLI SHALL leave the lifetime to the shares server.

#### Scenario: User requests a lifetime
- **WHEN** the user encrypts to an SSH recipient and requests a 3-day share lifetime
- **THEN** every share upload for that recipient asks the shares server for an expiry 3 days from now

#### Scenario: No lifetime requested
- **WHEN** the user encrypts to an SSH recipient without requesting a lifetime
- **THEN** share uploads do not request an expiry, and the server default for that recipient type applies
