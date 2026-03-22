# Capability: client-sdk

@sombra/sdk standalone TypeScript package — REST client, WebSocket manager, state store, message cache, MLS session management, offline outbox queue, crypto bridge to Rust core.

## ADDED Requirements

### Requirement: Standalone Public Package
The SDK SHALL be published as `@sombra/sdk`, a standalone TypeScript package with its own package.json, independent of the client-ui or any other consumer. The package MUST be usable by the official Sombra client, third-party clients, and bot developers without importing internal modules from other workspace packages.

#### Scenario: SDK is independently installable
- **WHEN** a third-party developer runs `npm install @sombra/sdk`
- **THEN** the package installs with all necessary dependencies and exports a fully functional SDK without requiring any other Sombra workspace packages

#### Scenario: SDK API surface is identical for all consumers
- **WHEN** the official Sombra client, a third-party client, and a bot developer each import `@sombra/sdk`
- **THEN** all three consumers receive the same exported types, classes, and functions with no hidden or privileged APIs

---

### Requirement: REST Client for All API Endpoints
The SDK SHALL provide a typed REST client that covers all `/api/v1/` endpoints exposed by the Sombra server. Every endpoint MUST have a corresponding SDK method with full TypeScript type definitions for request parameters and response bodies. The REST client MUST handle authentication headers, error parsing, and rate-limit retry automatically.

#### Scenario: SDK method maps to REST endpoint
- **WHEN** a consumer calls `sdk.channels.getMessages(channelId, { before, limit })`
- **THEN** the SDK sends a GET request to `/api/v1/channels/:id/messages` with the appropriate query parameters and returns a typed response

#### Scenario: Authentication header injected automatically
- **WHEN** a consumer calls any authenticated SDK method after establishing a session
- **THEN** the SDK automatically attaches the session token in the Authorization header without the consumer specifying it

#### Scenario: Rate-limit retry
- **WHEN** the server responds with HTTP 429 Too Many Requests including a Retry-After header
- **THEN** the SDK waits for the specified duration and retries the request automatically, up to a configurable maximum number of retries

---

### Requirement: WebSocket Manager with Full Lifecycle
The SDK SHALL provide a WebSocket manager that implements the HELLO/IDENTIFY/READY/RESUME lifecycle. The manager MUST handle connection establishment, heartbeat maintenance, sequence tracking for resume, automatic reconnection with exponential backoff, and event dispatching to the reactive state store.

#### Scenario: Initial connection handshake
- **WHEN** a consumer calls `sdk.gateway.connect()`
- **THEN** the WebSocket manager connects to `/api/v1/gateway`, receives the HELLO event with heartbeat interval, sends IDENTIFY with the session token, and waits for the READY event before resolving the connection promise

#### Scenario: Heartbeat maintenance
- **WHEN** the WebSocket connection is established and READY has been received
- **THEN** the manager sends heartbeat messages at the interval specified by the server's HELLO payload, and if a heartbeat ACK is not received within the expected window, the manager initiates reconnection

#### Scenario: Resume after disconnect
- **WHEN** the WebSocket connection drops unexpectedly
- **THEN** the manager reconnects with exponential backoff, sends RESUME with the last received sequence number, and the server replays missed events from the 60-second JetStream buffer

#### Scenario: Resume failure falls back to fresh IDENTIFY
- **WHEN** the resume attempt fails (e.g., sequence number is too old or session is invalidated)
- **THEN** the manager falls back to a fresh IDENTIFY handshake and re-populates the state store from the READY payload

---

### Requirement: Reactive State Store
The SDK SHALL maintain a reactive state store that holds normalized data for users, servers, channels, messages, and presence. The store MUST update reactively when WebSocket events arrive, and consumers MUST be able to subscribe to fine-grained state changes without polling.

#### Scenario: State populated from READY payload
- **WHEN** the WebSocket manager receives the READY event
- **THEN** the state store is populated with the user's servers, channels, initial presence data, and user records included in the payload

#### Scenario: State updated on WebSocket event
- **WHEN** the WebSocket manager receives a `MESSAGE_CREATE` event
- **THEN** the state store inserts the new message into the correct channel's message list and notifies any subscribers watching that channel

#### Scenario: Fine-grained subscription
- **WHEN** a consumer subscribes to presence changes for a specific user ID
- **THEN** the subscriber callback fires only when that specific user's presence changes, not when unrelated state updates occur

#### Scenario: State store exposes read-only accessors
- **WHEN** a consumer accesses state store data (e.g., `sdk.stores.channels.get(channelId)`)
- **THEN** the returned data is a read-only view that cannot be mutated directly — mutations only occur through SDK methods or incoming WebSocket events

---

### Requirement: Message Cache with Local Storage
The SDK SHALL cache messages locally to reduce redundant server fetches. The cache MUST persist across page reloads (using localStorage or IndexedDB on web, filesystem on Tauri). Cache entries MUST be keyed by channel ID and ULID range, and the SDK MUST serve cached messages for previously loaded ranges without re-fetching.

#### Scenario: Messages served from cache
- **WHEN** a consumer requests messages for a channel and those messages have been previously fetched and cached
- **THEN** the SDK returns the cached messages immediately without making a network request

#### Scenario: Cache persists across reloads
- **WHEN** the web page is reloaded or the Tauri app is restarted
- **THEN** previously cached messages are available from local storage and can be displayed before any network request completes

#### Scenario: Cache invalidation on message edit or delete
- **WHEN** the SDK receives a `MESSAGE_UPDATE` or `MESSAGE_DELETE` WebSocket event
- **THEN** the corresponding cache entry is updated or removed so that stale data is never served

---

### Requirement: MLS Session Management via Crypto Bridge
The SDK SHALL manage MLS session lifecycle by delegating all cryptographic operations to the Rust crypto core. The SDK MUST coordinate MLS group joins, commits, proposals, and welcome processing by passing opaque MLS messages between the server and the crypto core. The SDK SHALL NOT perform any MLS cryptographic computation itself.

#### Scenario: Join an MLS group
- **WHEN** the SDK receives an MLS Welcome message from the server for a new channel
- **THEN** the SDK passes the Welcome message to the Rust crypto core for processing and, upon success, stores the group state reference for future operations

#### Scenario: Encrypt outgoing message
- **WHEN** a consumer calls `sdk.messages.send(channelId, plaintext)`
- **THEN** the SDK passes the plaintext to the Rust crypto core for MLS encryption, receives the ciphertext blob, and sends it to the server via the REST API

#### Scenario: Decrypt incoming message
- **WHEN** the SDK receives a `MESSAGE_CREATE` event with a ciphertext blob
- **THEN** the SDK passes the ciphertext to the Rust crypto core for MLS decryption and stores the resulting plaintext in the state store

#### Scenario: SDK never performs crypto directly
- **WHEN** any operation requires cryptographic processing (encryption, decryption, key generation, signature verification)
- **THEN** the SDK delegates to the Rust crypto core and MUST NOT import or use any JavaScript cryptography libraries for MLS operations

---

### Requirement: Offline Outbox Queue
The SDK SHALL maintain an offline outbox queue that persists unsent messages when the client has no network connectivity. The outbox MUST use IndexedDB on web targets and SQLite on Tauri targets. When connectivity is restored, the SDK MUST flush the outbox in FIFO order, sending each queued message to the server.

#### Scenario: Message queued while offline
- **WHEN** a consumer calls `sdk.messages.send(channelId, plaintext)` while the client is offline
- **THEN** the SDK encrypts the message via the crypto core, stores the encrypted payload in the outbox queue, and returns a local outbox entry with a temporary client-generated ID

#### Scenario: Outbox persists across restarts
- **WHEN** the client application is closed and reopened while messages remain in the outbox
- **THEN** the persisted outbox entries are loaded from IndexedDB (web) or SQLite (Tauri) and are available for flushing when connectivity returns

#### Scenario: Outbox flushes on reconnection
- **WHEN** network connectivity is restored and the WebSocket connection is re-established
- **THEN** the SDK flushes all outbox entries in FIFO order, sending each to the server, and removes successfully sent entries from the queue

#### Scenario: Failed flush retries with backoff
- **WHEN** an outbox entry fails to send during flush (e.g., server returns 500)
- **THEN** the SDK retries that entry with exponential backoff without blocking subsequent entries in the queue

---

### Requirement: Optimistic UI Support
The SDK MUST support optimistic UI rendering by exposing outbox messages as part of the channel message list with a "sending" status indicator. When the server confirms the message and assigns a ULID, the SDK MUST replace the temporary client-generated ID with the server ULID and update the status to "sent". If sending fails permanently, the status MUST change to "failed" with a retry option.

#### Scenario: Outbox message appears immediately
- **WHEN** a consumer sends a message while online or offline
- **THEN** the SDK immediately inserts the message into the channel's message list in the state store with a `status: "sending"` property and a temporary client-generated ID

#### Scenario: Server ULID replaces temporary ID
- **WHEN** the server confirms the message and returns the assigned ULID
- **THEN** the SDK replaces the temporary client-generated ID with the server ULID, updates the status to `"sent"`, and the message retains its position in the chronological list

#### Scenario: Permanently failed message
- **WHEN** a message fails to send after all retry attempts are exhausted
- **THEN** the SDK updates the message status to `"failed"` and exposes a `retry()` method on the outbox entry for the consumer to re-attempt sending

---

### Requirement: Crypto Bridge — No Direct Crypto in SDK
The SDK MUST NOT import, bundle, or execute any cryptographic libraries directly. All cryptographic operations — including MLS key management, message encryption/decryption, SFrame operations, and KeyPackage generation — SHALL be performed exclusively by the Rust crypto core. The SDK communicates with the crypto core through a defined bridge interface (native FFI on Tauri, postMessage to Web Worker on web).

#### Scenario: SDK has no crypto dependencies
- **WHEN** the `@sombra/sdk` package.json is inspected
- **THEN** there are no dependencies on cryptographic libraries (e.g., no `openmls`, `@noble/ciphers`, `tweetnacl`, or similar packages)

#### Scenario: Bridge interface abstracts platform differences
- **WHEN** the SDK is running on a Tauri target
- **THEN** it calls the Rust crypto core via Tauri's native FFI invoke mechanism

#### Scenario: Bridge interface on web target
- **WHEN** the SDK is running in a web browser without Tauri
- **THEN** it communicates with the Rust crypto core WASM module running in a Web Worker via postMessage

---

### Requirement: Same SDK for All Consumers
The `@sombra/sdk` package SHALL be the sole client-side interface for interacting with the Sombra server. The official Sombra client, third-party clients, and bot developers MUST all use this same package. There SHALL NOT be separate "internal" and "external" SDK variants.

#### Scenario: Official client uses @sombra/sdk
- **WHEN** the official Sombra client application imports its server communication layer
- **THEN** it imports from `@sombra/sdk` and does not use any private or internal modules

#### Scenario: Bot developer uses @sombra/sdk
- **WHEN** a bot developer builds a Sombra bot in TypeScript
- **THEN** they install and import `@sombra/sdk` and have access to the same REST client, WebSocket manager, and state store as the official client

#### Scenario: No hidden privileged APIs
- **WHEN** the SDK's public exports are compared between the version used by the official client and the version published to npm
- **THEN** the exports are identical — there are no additional methods, classes, or types available only to the official client
