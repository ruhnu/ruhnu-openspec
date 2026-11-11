# Spec Delta

## ADDED Requirements

### Requirement: Requested share lifetime is checked and reported
When the user requests a share lifetime for an SSH recipient, the client SHALL refuse a zero or negative lifetime before contacting any shares server. When a shares server reports that it shortened the requested lifetime, the client SHALL warn the user, naming that server and the expiry time it applied.

#### Scenario: Zero or negative lifetime refused
- **WHEN** the user encrypts to an SSH recipient and requests a share lifetime of zero or a negative duration
- **THEN** the client fails with an error stating that the lifetime must be positive, and no share is uploaded

#### Scenario: Shortened lifetime reported
- **WHEN** the user requests a 30-day share lifetime and a shares server with a 14-day maximum responds that it adjusted the expiry
- **THEN** the container is created, and the client warns that this server shortened the lifetime, with the expiry time the server reported

#### Scenario: Lifetime accepted as requested
- **WHEN** the user requests a share lifetime that every shares server accepts unchanged
- **THEN** no warning about the lifetime is shown
