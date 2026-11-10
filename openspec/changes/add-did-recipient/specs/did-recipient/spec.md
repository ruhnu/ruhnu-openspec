# Spec Delta

## Purpose

Lets a CDOC2 container be encrypted to an SSH key that a person's `did:web` DID document lists, so that the holder can decrypt it through ssh-agent only while the DID document still lists the key, and a compromised DID host cannot add keys that open existing containers.

## ADDED Requirements

### Requirement: DID recipient identifier
A DID recipient SHALL be stored as a key-shares recipient with the recipient type `RUHNU` and a `recipient_id` of the form `ruhnu/did/<did>/hash/<fingerprint>`. `<did>` SHALL be a `did:web` DID. `<fingerprint>` SHALL be the base64url encoding, without padding, of the SHA-256 hash of the key's OpenSSH public key wire blob. The same DID and key SHALL always produce the same identifier.

#### Scenario: Identifier built from a DID and a key
- **WHEN** the DID document of `did:web:example.org:p:alice` lists a P-256 key whose `ssh-keygen -l` fingerprint is `SHA256:NzbLsXh8uDCcd+6MNwXF4W/7noWXFZAfHkxZsRGC9Xs`
- **THEN** the recipient for that key is stored with the recipient type `RUHNU` and `recipient_id` `ruhnu/did/did:web:example.org:p:alice/hash/NzbLsXh8uDCcd-6MNwXF4W_7noWXFZAfHkxZsRGC9Xs`

#### Scenario: Identifier is independent of the verification method id
- **WHEN** the same key is listed in the DID document under the id `#key-1`, and later under `#laptop`
- **THEN** both produce the same `recipient_id`

#### Scenario: DID method other than did:web
- **WHEN** the user encrypts to a DID whose method is not `web`, such as `did:key:zDn...`
- **THEN** encryption fails before any share is uploaded, with an error stating that only `did:web` DIDs are supported

#### Scenario: did:web with an IP address
- **WHEN** the user encrypts to a `did:web` DID whose host is an IP address, such as `did:web:192.0.2.1`
- **THEN** encryption fails before any share is uploaded, with an error stating the DID is not a valid `did:web` DID

#### Scenario: Identifier too long
- **WHEN** the user encrypts to a DID for which the `recipient_id` would be longer than 255 characters
- **THEN** encryption fails before any share is uploaded, with an error stating the DID is too long

#### Scenario: Malformed identifier in a container
- **WHEN** a container has a `RUHNU` recipient whose `recipient_id` starts with `ruhnu/did/` but whose DID is not a valid `did:web` DID, or whose fingerprint does not decode to 32 bytes of canonical unpadded base64url
- **THEN** decryption with that recipient fails with an error stating the recipient identifier is invalid, and no request is sent to any shares server

### Requirement: Resolving a did:web DID
The client and the shares server SHALL resolve a `did:web` DID with one HTTPS GET of the URL the `did:web` method defines. They SHALL validate TLS against the system's trusted CAs and SHALL NOT follow redirects. Resolution SHALL succeed only with a 200 response whose body is a JSON DID document whose `id` equals the DID. A connection failure, timeout, TLS failure, 5xx or 429 response SHALL mean the DID host is unavailable; any other failure SHALL mean the DID is not found.

#### Scenario: DID with a path
- **WHEN** `did:web:example.org:p:alice` is resolved
- **THEN** the document is fetched from `https://example.org/p/alice/did.json`

#### Scenario: DID without a path
- **WHEN** `did:web:example.org` is resolved
- **THEN** the document is fetched from `https://example.org/.well-known/did.json`

#### Scenario: DID with a port
- **WHEN** `did:web:example.org%3A8443:p:alice` is resolved
- **THEN** the document is fetched from `https://example.org:8443/p/alice/did.json`

#### Scenario: Document missing
- **WHEN** the DID host responds 404 or 410
- **THEN** the DID is not found

#### Scenario: Redirect
- **WHEN** the DID host responds with a redirect
- **THEN** the redirect is not followed, no request is sent to its target, and the DID is not found

#### Scenario: Document for another DID
- **WHEN** the DID host returns a DID document whose `id` is not the DID being resolved
- **THEN** the DID is not found

#### Scenario: DID host down
- **WHEN** the DID host cannot be connected to, does not answer in time, fails TLS validation, or responds with a 5xx or 429 status
- **THEN** the DID host is unavailable

### Requirement: Keys listed by a DID document
A DID document SHALL list a key if and only if its `authentication` contains a verification method, embedded or as a reference to an entry in `verificationMethod`, whose `controller` equals the DID and whose `publicKeyJwk` is an EC P-256 public key. Fingerprints SHALL be computed from the key's OpenSSH public key wire blob. Other verification methods and key types SHALL be ignored.

#### Scenario: Key referenced from authentication
- **WHEN** `verificationMethod` contains `did:web:example.org:p:alice#key-1` with a P-256 `publicKeyJwk`, and `authentication` contains the reference `did:web:example.org:p:alice#key-1` or `#key-1`
- **THEN** the document lists that key

#### Scenario: Key embedded in authentication
- **WHEN** `authentication` contains an embedded verification method with a P-256 `publicKeyJwk` and the DID as `controller`
- **THEN** the document lists that key

#### Scenario: Key not used for authentication
- **WHEN** a P-256 key appears only in `verificationMethod`, or only in other relationships such as `assertionMethod`
- **THEN** the document does not list that key

#### Scenario: Key controlled by another DID
- **WHEN** a verification method in `authentication` has a `controller` other than the DID
- **THEN** the document does not list that key

#### Scenario: Unsupported key types ignored
- **WHEN** `authentication` contains a P-256 key and an Ed25519 key
- **THEN** the document lists only the P-256 key

### Requirement: Encrypt to a DID
The CLI SHALL let the user encrypt a container to one or more DIDs. For each DID it SHALL resolve the DID document and add one DID recipient per key the document lists, with shares on every configured shares server. It SHALL print each pinned `recipient_id` and the key's `SHA256:` fingerprint as `ssh-keygen -l` shows it. A requested share lifetime SHALL apply to DID recipients too.

#### Scenario: DID with two keys
- **WHEN** the user creates a container for a DID whose document lists two P-256 keys
- **THEN** the container has two DID recipients, one per key, each with shares on every configured shares server, and the CLI prints both pinned recipients and fingerprints

#### Scenario: Unsupported key skipped
- **WHEN** the user creates a container for a DID whose document lists a P-256 key and an Ed25519 key under `authentication`
- **THEN** the container has one DID recipient for the P-256 key, and the CLI reports that the Ed25519 key was skipped as unsupported

#### Scenario: No supported key
- **WHEN** the user creates a container for a DID whose document lists no P-256 key
- **THEN** encryption fails before any share is uploaded, with an error stating that the DID lists no supported key

#### Scenario: DID not found
- **WHEN** the user creates a container for a DID that is not found
- **THEN** encryption fails before any share is uploaded, with an error stating that the DID document was not found

#### Scenario: DID host unavailable when encrypting
- **WHEN** the user creates a container for a DID whose host is unavailable
- **THEN** encryption fails before any share is uploaded, with an error stating that the DID host could not be reached

#### Scenario: Requested lifetime
- **WHEN** the user creates a container for a DID and requests a 3-day share lifetime
- **THEN** every share upload for the DID's recipients asks the shares server for an expiry 3 days from now

### Requirement: Decrypt a DID recipient with ssh-agent
The client SHALL decrypt a DID recipient through the ssh-agent at `$SSH_AUTH_SOCK`, using the agent key whose fingerprint equals the recipient's `<fingerprint>`. The client SHALL NOT resolve the DID to decrypt. Decryption SHALL NOT require a private key file, Smart-ID, Mobile-ID, an RP server or an auth server.

#### Scenario: Successful decryption
- **WHEN** the agent holds a key whose fingerprint matches a DID recipient of the container, the DID document still lists that key, and the user decrypts the container
- **THEN** the client fetches every share, reconstructs the key and decrypts the container

#### Scenario: Agent holds no matching key
- **WHEN** the agent is reachable but holds no key whose fingerprint matches any SSH public key or DID recipient of the container
- **THEN** decryption fails with an error stating that no matching key was found in ssh-agent and naming the DID and fingerprint of each DID recipient, and no auth token is sent to any shares server

#### Scenario: No agent available for a DID recipient
- **WHEN** the container's DID recipients are its only SSH key recipients, and `$SSH_AUTH_SOCK` is unset or the socket cannot be connected
- **THEN** decryption fails with an error stating that ssh-agent is not available, and no request is sent to any shares server

#### Scenario: Agent refuses to sign for a DID recipient
- **WHEN** the agent holds the key of a DID recipient but refuses the sign request, for example because the user declined an `ssh-add -c` confirmation
- **THEN** decryption fails with an error stating that ssh-agent refused to sign, and no auth token is sent to any shares server

#### Scenario: Container with SSH public key and DID recipients
- **WHEN** a container has a `ruhnu/pk/` recipient for key A and a `ruhnu/did/` recipient for key B, and the agent holds only key B
- **THEN** the client decrypts the container with the DID recipient

#### Scenario: Key no longer listed
- **WHEN** a shares server responds 404 to a share request for a DID recipient
- **THEN** decryption fails with an error stating that the share was not found, has expired, or the key is no longer listed in the recipient's DID document

#### Scenario: DID host unavailable when decrypting
- **WHEN** a shares server responds 503 to a share request for a DID recipient
- **THEN** decryption fails with an error stating that the shares server could not reach the DID host and that the user can try again later

### Requirement: Auth token for a DID recipient
For each share of a DID recipient, the client SHALL send an SD-JWT auth token whose JWS uses `alg` `ES256`, whose header carries the signing key's public JWK in `jwk`, whose `iss` equals the `recipient_id`, and whose `aud` disclosure names the share's URL and server nonce. The `jwk` key's fingerprint SHALL equal the recipient's `<fingerprint>`, and the JWS SHALL verify with it.

#### Scenario: Token verifies against the header key
- **WHEN** the client creates an auth token for a share of a DID recipient
- **THEN** the token's JWS header has `alg` `ES256` and a `jwk` with only the public P-256 key members, `iss` equals the `recipient_id`, the SHA-256 fingerprint of the `jwk` key equals the `<fingerprint>` in `iss`, and the signature verifies with that key
