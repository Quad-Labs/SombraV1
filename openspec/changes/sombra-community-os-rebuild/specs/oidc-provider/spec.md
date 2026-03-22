## ADDED Requirements

### Requirement: Embedded OAuth2/OIDC Provider
The server SHALL embed a fully functional OAuth2/OIDC provider using the ory/fosite library, compiled directly into the single Go binary. No external identity provider service SHALL be required. The embedded provider MUST support the Authorization Code Grant with PKCE, Client Credentials Grant, and Refresh Token Grant flows.

#### Scenario: OIDC provider starts with the server
- **WHEN** the Sombra server binary starts
- **THEN** the embedded OIDC provider is initialized and ready to handle OAuth2/OIDC requests without any external service dependency

#### Scenario: Authorization Code Grant with PKCE
- **WHEN** a client application initiates an Authorization Code Grant flow with PKCE
- **THEN** the embedded provider issues an authorization code, validates the PKCE code verifier on token exchange, and returns an ID token, access token, and refresh token

#### Scenario: Client Credentials Grant
- **WHEN** a registered OAuth2 client sends a Client Credentials Grant request with valid client ID and secret
- **THEN** the embedded provider issues an access token scoped to the client's registered permissions

### Requirement: Log in with Sombra Flow
The OIDC provider SHALL support a "Log in with Sombra" flow that allows community apps (installed via the community app orchestrator) to authenticate users against the Sombra server's user database. The user MUST be presented with a consent screen showing the requested scopes before granting access to a third-party application.

#### Scenario: Community app redirects user to Sombra login
- **WHEN** a user clicks "Log in with Sombra" in a community app
- **THEN** the user is redirected to the Sombra server's authorization endpoint, prompted to log in (if not already authenticated), shown a consent screen with requested scopes, and upon approval redirected back to the app with an authorization code

#### Scenario: User denies consent
- **WHEN** a user clicks "Deny" on the consent screen
- **THEN** the user is redirected back to the community app with an `access_denied` error parameter and no authorization code is issued

#### Scenario: Already-authenticated user skips login prompt
- **WHEN** a user who is already logged into Sombra clicks "Log in with Sombra" in a community app
- **THEN** the user is shown the consent screen directly without being prompted to re-enter credentials

### Requirement: Bot OAuth Flows
The OIDC provider SHALL support OAuth2 flows for bot applications. Bots MUST be able to authenticate using the Client Credentials Grant to obtain access tokens. Bot tokens SHALL carry scoped permissions that restrict API access to only the capabilities granted by the server owner during bot registration.

#### Scenario: Bot authenticates via Client Credentials
- **WHEN** a registered bot sends a Client Credentials Grant request with its client ID and client secret
- **THEN** the OIDC provider issues an access token with scopes matching the bot's registered permissions

#### Scenario: Bot token respects scoped permissions
- **WHEN** a bot with only `messages:read` scope attempts to call `POST /api/v1/channels/:id/messages`
- **THEN** the server rejects the request with a 403 Forbidden status because the bot's token lacks the `messages:write` scope

#### Scenario: Bot registration by server owner
- **WHEN** a server owner registers a new bot application via the server settings
- **THEN** the OIDC provider creates a client ID and client secret for the bot, and the owner can configure the permitted scopes

### Requirement: User Identity and Permissions as OIDC Claims
ID tokens and the UserInfo endpoint MUST include the authenticated user's identity, server-level roles, and channel-level permissions as OIDC claims. Claims SHALL include at minimum: `sub` (user ID), `preferred_username`, `email` (if consented), `sombra:server_roles` (list of server role IDs), and `sombra:channel_permissions` (map of channel ID to permission bitmask for channels the app is authorized to access).

#### Scenario: ID token contains server roles
- **WHEN** a user with roles "Admin" and "Moderator" completes the OIDC flow for a community app requesting the `roles` scope
- **THEN** the issued ID token contains a `sombra:server_roles` claim listing the user's server role IDs

#### Scenario: ID token contains channel permissions
- **WHEN** a user completes the OIDC flow for a community app requesting the `channels` scope
- **THEN** the issued ID token contains a `sombra:channel_permissions` claim mapping authorized channel IDs to their resolved permission bitmasks for that user

#### Scenario: UserInfo endpoint returns claims
- **WHEN** a community app calls the UserInfo endpoint with a valid access token
- **THEN** the response includes the same identity, role, and permission claims as the ID token

#### Scenario: Claims reflect current state
- **WHEN** a user's roles are changed after an ID token was issued, and the community app calls the UserInfo endpoint
- **THEN** the UserInfo response reflects the user's current roles and permissions, not the stale values from the original token

### Requirement: No Separate Account Creation for Integrated Apps
Community apps that authenticate via "Log in with Sombra" SHALL NOT require users to create separate accounts. The user's Sombra identity MUST be the sole identity used for authentication and authorization within integrated apps. Apps SHALL receive user identity via OIDC claims and MUST NOT maintain their own user registration flows for Sombra users.

#### Scenario: User accesses community app without separate signup
- **WHEN** a user who has never used a community app before clicks "Log in with Sombra"
- **THEN** the user is authenticated via the OIDC flow and gains access to the community app using their Sombra identity, without any additional registration step

#### Scenario: User identity is consistent across apps
- **WHEN** a user logs into two different community apps via "Log in with Sombra"
- **THEN** both apps receive the same `sub` claim (user ID) and `preferred_username`, identifying the user consistently

### Requirement: OIDC Discovery Endpoint
The server SHALL expose a standard OIDC discovery document at `/.well-known/openid-configuration`. The discovery document MUST include all required metadata fields per the OpenID Connect Discovery 1.0 specification, including `issuer`, `authorization_endpoint`, `token_endpoint`, `userinfo_endpoint`, `jwks_uri`, and `scopes_supported`.

#### Scenario: Discovery document is publicly accessible
- **WHEN** a client sends a `GET` request to `/.well-known/openid-configuration`
- **THEN** the server responds with a 200 OK status and a JSON document containing valid OIDC discovery metadata

#### Scenario: Discovery document does not require authentication
- **WHEN** an unauthenticated client sends a `GET` request to `/.well-known/openid-configuration`
- **THEN** the server responds with a 200 OK status without requiring a session token or API key

#### Scenario: JWKS endpoint is functional
- **WHEN** a client fetches the URL specified in the `jwks_uri` field of the discovery document
- **THEN** the server responds with a JSON Web Key Set containing the public keys used to sign ID tokens

#### Scenario: Issuer matches server URL
- **WHEN** the server is running at `https://sombra.example.com`
- **THEN** the `issuer` field in the discovery document is `https://sombra.example.com` and all endpoint URLs in the document use the same base URL

### Requirement: Token Introspection and Refresh
The OIDC provider SHALL support token introspection (RFC 7662) and token refresh (RFC 6749 Section 6). The introspection endpoint MUST allow resource servers and community apps to validate access tokens and retrieve their associated metadata. Refresh tokens MUST be rotatable — issuing a new refresh token on each use and invalidating the previous one.

#### Scenario: Token introspection returns active token metadata
- **WHEN** a community app sends a `POST` request to the introspection endpoint with a valid, non-expired access token
- **THEN** the server responds with `{"active": true}` and includes metadata fields: `sub`, `scope`, `exp`, `iat`, and `client_id`

#### Scenario: Token introspection detects expired tokens
- **WHEN** a community app sends a `POST` request to the introspection endpoint with an expired access token
- **THEN** the server responds with `{"active": false}`

#### Scenario: Refresh token issues new token pair
- **WHEN** a client sends a refresh token grant request with a valid refresh token
- **THEN** the server issues a new access token and a new refresh token, and the previous refresh token is invalidated

#### Scenario: Reuse of invalidated refresh token revokes token family
- **WHEN** a client attempts to use a refresh token that has already been exchanged for a new token pair
- **THEN** the server rejects the request, detects potential token theft, and revokes all tokens in the refresh token family for that client-user pair
