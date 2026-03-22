## ADDED Requirements

### Requirement: User Registration
The server SHALL allow new users to register by providing a username, email, and password via POST `/api/v1/auth/register`. The server MUST validate that the username and email are unique, hash the password using a secure algorithm (e.g., bcrypt or argon2), and return the created user record with a session token.

#### Scenario: Successful registration
- **WHEN** a client sends a POST request to `/api/v1/auth/register` with a valid username, email, and password
- **THEN** the server creates a new user record, returns a 201 Created response with the user object and a session token

#### Scenario: Duplicate username rejected
- **WHEN** a client sends a POST request to `/api/v1/auth/register` with a username that already exists
- **THEN** the server responds with a 409 Conflict status and an error message indicating the username is taken

#### Scenario: Duplicate email rejected
- **WHEN** a client sends a POST request to `/api/v1/auth/register` with an email that is already associated with an account
- **THEN** the server responds with a 409 Conflict status and an error message indicating the email is already in use

#### Scenario: Invalid input rejected
- **WHEN** a client sends a POST request to `/api/v1/auth/register` with missing or malformed fields (e.g., empty username, invalid email format, password too short)
- **THEN** the server responds with a 400 Bad Request status and validation error details

### Requirement: Login Returns Session Token
The server SHALL authenticate a user via POST `/api/v1/auth/login` using email and password. On successful authentication, the server MUST return a session token that can be used for subsequent authenticated requests.

#### Scenario: Successful login
- **WHEN** a client sends a POST request to `/api/v1/auth/login` with a valid email and correct password
- **THEN** the server responds with a 200 OK status and a response body containing a session token and the user object

#### Scenario: Invalid credentials rejected
- **WHEN** a client sends a POST request to `/api/v1/auth/login` with an incorrect password or non-existent email
- **THEN** the server responds with a 401 Unauthorized status without revealing whether the email or password was incorrect

### Requirement: Multi-Device Session Management
The server SHALL support multiple concurrent sessions per user (one per device). The server MUST provide endpoints to list all active sessions, revoke an individual session, and revoke all sessions except the current one.

#### Scenario: List active sessions
- **WHEN** an authenticated client sends a GET request to the sessions endpoint
- **THEN** the server returns a list of all active sessions for the user, including device metadata and creation timestamps

#### Scenario: Revoke individual session
- **WHEN** an authenticated client sends a DELETE request targeting a specific session ID
- **THEN** that session is invalidated, and any subsequent requests using that session's token are rejected with 401 Unauthorized

#### Scenario: Revoke all other sessions
- **WHEN** an authenticated client sends a request to revoke all sessions except the current one
- **THEN** all other sessions for the user are invalidated, while the current session remains valid

### Requirement: Session Token Validation on Every Authenticated Request
The Authenticator middleware MUST validate the session token on every request to an authenticated endpoint. If the token is missing, expired, or revoked, the server SHALL reject the request with a 401 Unauthorized response.

#### Scenario: Valid token grants access
- **WHEN** a client sends a request with a valid, non-expired session token in the Authorization header
- **THEN** the server identifies the user, injects the user identity into the request context, and allows the request to proceed

#### Scenario: Missing token rejected
- **WHEN** a client sends a request to an authenticated endpoint without an Authorization header
- **THEN** the server responds with a 401 Unauthorized status

#### Scenario: Expired or revoked token rejected
- **WHEN** a client sends a request with a session token that has been revoked or has expired
- **THEN** the server responds with a 401 Unauthorized status

### Requirement: Account Recovery Key Generation
The server SHALL generate a BIP-39 mnemonic recovery key for each user at signup. This recovery key MUST be derived from the user's identity keypair and displayed to the user exactly once during the registration flow. The server SHALL NOT store the recovery key in plaintext.

#### Scenario: Recovery key generated at registration
- **WHEN** a user successfully registers a new account
- **THEN** the server generates a BIP-39 mnemonic recovery key derived from the user's identity keypair and includes it in the registration response

#### Scenario: Recovery key displayed only once
- **WHEN** the registration response includes the recovery key
- **THEN** there is no subsequent API endpoint that returns the recovery key again — it is the user's responsibility to store it securely

### Requirement: Recovery Key Device Bootstrap
A user SHALL be able to use their BIP-39 recovery key to bootstrap a new device, re-establish their identity, and rejoin MLS groups. The recovery flow MUST regenerate the identity keypair from the mnemonic, create a new session on the new device, and upload new MLS KeyPackages for group re-admission.

#### Scenario: Recovery key bootstraps new device
- **WHEN** a user submits their BIP-39 mnemonic to the account recovery endpoint from a new device
- **THEN** the server verifies the mnemonic against the stored identity public key, creates a new session for the device, and the client uses the regenerated keypair to upload new MLS KeyPackages

#### Scenario: Re-joining MLS groups after recovery
- **WHEN** a user has recovered their account on a new device
- **THEN** the client generates new MLS KeyPackages and existing group members can add the recovered device as a new leaf node via MLS Commit

### Requirement: Recovery Key Preserves Forward Secrecy
The recovery key MUST NOT enable decryption of historical ephemeral messages. Forward secrecy SHALL be preserved: the recovery key restores identity but not past MLS epoch keys.

#### Scenario: Historical messages not decryptable after recovery
- **WHEN** a user recovers their account using the BIP-39 mnemonic and rejoins an MLS group
- **THEN** the user cannot decrypt messages sent before the recovery — only messages from the current MLS epoch onward are decryptable

### Requirement: Authentication REST Endpoints
The server SHALL expose the following authentication endpoints: POST `/api/v1/auth/register` for user registration, POST `/api/v1/auth/login` for user login, and POST `/api/v1/auth/logout` for session termination.

#### Scenario: Register endpoint accessible
- **WHEN** a client sends a POST request to `/api/v1/auth/register`
- **THEN** the request is routed to the registration handler

#### Scenario: Login endpoint accessible
- **WHEN** a client sends a POST request to `/api/v1/auth/login`
- **THEN** the request is routed to the login handler

#### Scenario: Logout endpoint terminates session
- **WHEN** an authenticated client sends a POST request to `/api/v1/auth/logout`
- **THEN** the current session token is invalidated and the server responds with a 200 OK status

#### Scenario: Logout without valid token
- **WHEN** a client sends a POST request to `/api/v1/auth/logout` without a valid session token
- **THEN** the server responds with a 401 Unauthorized status

### Requirement: Current User Profile Endpoints
The server SHALL expose GET `/api/v1/users/@me` to retrieve the current authenticated user's profile and PATCH `/api/v1/users/@me` to update it. Only the authenticated user SHALL be able to access and modify their own profile via these endpoints.

#### Scenario: Get current user profile
- **WHEN** an authenticated client sends a GET request to `/api/v1/users/@me`
- **THEN** the server responds with a 200 OK status and the full profile of the authenticated user (username, email, avatar, etc.)

#### Scenario: Update current user profile
- **WHEN** an authenticated client sends a PATCH request to `/api/v1/users/@me` with updated fields (e.g., display name, avatar)
- **THEN** the server updates the specified fields and responds with a 200 OK status and the updated user profile

#### Scenario: Unauthenticated access denied
- **WHEN** a client sends a GET or PATCH request to `/api/v1/users/@me` without a valid session token
- **THEN** the server responds with a 401 Unauthorized status
