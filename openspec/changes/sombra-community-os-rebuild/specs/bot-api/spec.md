# Capability: bot-api

Bots as first-class API citizens -- same REST + WebSocket API, API tokens with scoped permissions, E2EE bots (full MLS members) and plaintext bots (stateless), SDKs for TypeScript/Rust/Python.

## ADDED Requirements

### Requirement: Bots as User Accounts with Bot Flag
Bots SHALL be represented as user accounts with a `bot: true` flag and authenticated via API tokens instead of session cookies. Bot user accounts MUST appear in member lists and MUST be distinguishable from human users by the `bot` flag.

#### Scenario: Bot account has bot flag
- **WHEN** a bot account is retrieved via `GET /api/v1/users/:id`
- **THEN** the response SHALL include `"bot": true` in the user object

#### Scenario: Bot authenticates with API token
- **WHEN** a bot sends a request with an `Authorization: Bot <token>` header
- **THEN** the server SHALL authenticate the bot using the API token and inject the bot's user identity into the request context

#### Scenario: Bot appears in server member list
- **WHEN** a client fetches the member list for a server that includes a bot
- **THEN** the bot SHALL appear in the member list with its `bot: true` flag visible

---

### Requirement: Same REST and WebSocket API as Human Clients
Bots MUST use the same REST API endpoints and WebSocket gateway as human clients. There SHALL NOT be a separate bot-only API surface. All endpoints accessible to human users MUST be accessible to bots, subject to their scoped permissions.

#### Scenario: Bot sends a message via REST API
- **WHEN** a bot sends `POST /api/v1/channels/:id/messages` with a valid ciphertext payload
- **THEN** the server SHALL create the message using the same handler and logic as for a human client

#### Scenario: Bot receives events via WebSocket
- **WHEN** a bot establishes a WebSocket connection to the gateway
- **THEN** the bot SHALL receive the same event types (MESSAGE_CREATE, CHANNEL_UPDATE, etc.) as a human client connected to the same channels

#### Scenario: No separate bot API surface
- **WHEN** a developer inspects the API route definitions
- **THEN** there SHALL be no `/api/v1/bot/` or equivalent bot-only namespace; bots use the same `/api/v1/` routes as human clients

---

### Requirement: Scoped API Token Permissions
Bot API tokens MUST support scoped permissions. Scopes SHALL include at minimum: `read` (read-only access to channels and messages), `send` (ability to send messages), and `admin` (server administration capabilities). Scopes MUST be assignable per-server.

#### Scenario: Read-only bot cannot send messages
- **WHEN** a bot with only the `read` scope sends `POST /api/v1/channels/:id/messages`
- **THEN** the server SHALL reject the request with HTTP 403 indicating insufficient permissions

#### Scenario: Send-scoped bot can post messages
- **WHEN** a bot with the `send` scope sends `POST /api/v1/channels/:id/messages` with a valid payload
- **THEN** the server SHALL accept the request and create the message

#### Scenario: Admin-scoped bot can manage channels
- **WHEN** a bot with the `admin` scope sends a request to create or delete a channel
- **THEN** the server SHALL permit the operation subject to the bot's role within the server

#### Scenario: Scopes are per-server
- **WHEN** a bot has `send` scope for Server A but only `read` scope for Server B
- **THEN** the bot SHALL be able to send messages in Server A but MUST be rejected with HTTP 403 when attempting to send messages in Server B

---

### Requirement: E2EE Bots as Full MLS Group Members
E2EE bots SHALL be full MLS group members that participate in the group key schedule. E2EE bots MUST maintain local storage for MLS state (key packages, group state, epoch secrets). E2EE bots MUST bundle the OpenMLS crypto core to perform encryption and decryption locally.

#### Scenario: E2EE bot joins MLS group
- **WHEN** an E2EE bot is added to a channel
- **THEN** the bot SHALL be added to the channel's MLS group as a full member via an MLS Add proposal and commit, and the bot SHALL receive the group's current epoch secret

#### Scenario: E2EE bot decrypts messages locally
- **WHEN** the E2EE bot receives a MESSAGE_CREATE event with a ciphertext blob
- **THEN** the bot SHALL decrypt the message locally using its MLS group state and the OpenMLS crypto core

#### Scenario: E2EE bot requires local storage
- **WHEN** an E2EE bot is initialized
- **THEN** the bot SDK SHALL create and maintain a local storage directory for MLS key packages, group state, and epoch secrets

#### Scenario: E2EE bot encrypts outgoing messages
- **WHEN** an E2EE bot sends a message to a channel
- **THEN** the bot SHALL encrypt the message locally using MLS application message encryption before sending the ciphertext blob to the server

---

### Requirement: Plaintext Bots as Stateless API Wrappers
Plaintext bots SHALL operate as simple stateless API wrappers that do not participate in MLS groups. Plaintext bots MUST NOT have access to encrypted message content. Plaintext bots SHALL be suitable for running on serverless platforms such as AWS Lambda.

#### Scenario: Plaintext bot does not join MLS group
- **WHEN** a plaintext bot is added to a channel
- **THEN** the bot SHALL NOT be added to the channel's MLS group and SHALL NOT receive MLS key packages or epoch secrets

#### Scenario: Plaintext bot cannot decrypt messages
- **WHEN** a plaintext bot receives a MESSAGE_CREATE event for a channel with E2EE enabled
- **THEN** the bot SHALL receive only the message metadata (sender, timestamp, channel) and the opaque ciphertext blob, which it cannot decrypt

#### Scenario: Plaintext bot runs on serverless
- **WHEN** a plaintext bot is deployed as an AWS Lambda function triggered by a webhook
- **THEN** the bot SHALL process the incoming event, make any necessary API calls, and return without maintaining any local state between invocations

#### Scenario: Plaintext bot sends messages to non-E2EE contexts
- **WHEN** a plaintext bot sends a response in a channel or via a slash command response
- **THEN** the bot SHALL send the message as a plaintext payload and the server SHALL handle it according to channel encryption settings

---

### Requirement: Bot Creation Endpoint
The server SHALL expose `POST /api/v1/servers/:id/bots` for creating a new bot. The endpoint MUST return the bot's user object and a one-time-visible API token. Only users with server admin permissions SHALL be authorized to create bots.

#### Scenario: Admin creates a bot
- **WHEN** a server admin sends `POST /api/v1/servers/:id/bots` with `{"name": "my-bot", "type": "plaintext", "scopes": ["read", "send"]}`
- **THEN** the server SHALL create a bot user account with `bot: true`, generate a scoped API token, and return both the user object and the token in the response

#### Scenario: API token shown only once
- **WHEN** a bot is successfully created via `POST /api/v1/servers/:id/bots`
- **THEN** the API token SHALL be included in the creation response and MUST NOT be retrievable again through any subsequent API call

#### Scenario: Non-admin cannot create bots
- **WHEN** a user without server admin permissions sends `POST /api/v1/servers/:id/bots`
- **THEN** the server SHALL reject the request with HTTP 403

#### Scenario: Bot type must be specified
- **WHEN** a request to `POST /api/v1/servers/:id/bots` includes `"type": "e2ee"` or `"type": "plaintext"`
- **THEN** the server SHALL create the bot with the specified type, which determines whether the bot will be enrolled in MLS groups

---

### Requirement: Bot SDKs for TypeScript, Rust, and Python
The project SHALL provide official Bot SDKs for three languages: `@sombra/bot-sdk` for TypeScript (npm), `sombra-bot-sdk` for Rust (crates.io), and `sombra-bot-sdk` for Python (PyPI). Each SDK MUST provide connection management, event handling, and REST API wrappers.

#### Scenario: TypeScript SDK connects and receives events
- **WHEN** a developer uses `@sombra/bot-sdk` to create a bot client with a valid API token
- **THEN** the SDK SHALL establish a WebSocket connection, authenticate using the token, and deliver parsed events via an event emitter or callback interface

#### Scenario: Rust SDK provides typed API wrappers
- **WHEN** a developer uses the `sombra-bot-sdk` Rust crate to send a message
- **THEN** the SDK SHALL provide a strongly-typed method that constructs and sends the `POST /api/v1/channels/:id/messages` request and returns a typed response

#### Scenario: Python SDK supports async event handling
- **WHEN** a developer uses the `sombra-bot-sdk` Python package to listen for events
- **THEN** the SDK SHALL support async/await event handling via asyncio and deliver parsed events to registered handler functions

#### Scenario: E2EE SDKs bundle OpenMLS
- **WHEN** a developer creates an E2EE bot using any of the three SDKs
- **THEN** the SDK SHALL include or depend on the OpenMLS crypto core for MLS group operations, key package management, and message encryption/decryption

---

### Requirement: Bots Work on All Tiers Including Tier 1
Bots MUST function on all hosting tiers, including Tier 1 localhost deployments. The bot API endpoints, WebSocket event delivery, and SDK connectivity SHALL operate identically regardless of the hosting tier.

#### Scenario: Bot operates on Tier 1 localhost
- **WHEN** a bot connects to a Tier 1 Sombra instance at `http://localhost:8080` using a valid API token
- **THEN** the bot SHALL authenticate, receive WebSocket events, and interact with the REST API exactly as it would on any other tier

#### Scenario: Bot SDK works without internet access
- **WHEN** a bot SDK is configured to connect to a Tier 1 instance on a machine with no internet access
- **THEN** the SDK SHALL connect to the local server and function without requiring any external service dependencies
