# Capability: messaging

Text channels, messages (ULID-ordered, ciphertext blobs), threads, reactions, message editing/deletion, pins, cursor-based pagination.

## ADDED Requirements

### Requirement: Opaque Ciphertext Storage

Messages SHALL be stored as opaque ciphertext blobs. The server MUST NOT attempt to decrypt, parse, or inspect message content. The server SHALL treat the `ciphertext` field as an arbitrary byte sequence and MUST store and return it verbatim.

#### Scenario: Server stores ciphertext without inspection
- **WHEN** a client sends a message with an encrypted ciphertext blob
- **THEN** the server SHALL persist the blob exactly as received and MUST NOT attempt decryption or content inspection

#### Scenario: Server returns ciphertext verbatim
- **WHEN** a client fetches messages from a channel
- **THEN** the server SHALL return the `ciphertext` field byte-for-byte identical to what was originally submitted

---

### Requirement: ULID Message Identifiers

All message IDs SHALL be ULIDs (Universally Unique Lexicographically Sortable Identifiers). Message IDs MUST be time-ordered such that lexicographic sorting produces chronological ordering. The server MUST generate the ULID at message creation time.

#### Scenario: Message ID is a valid ULID
- **WHEN** a message is created via `POST /api/v1/channels/:id/messages`
- **THEN** the server SHALL assign a ULID as the message ID that encodes the server-side creation timestamp

#### Scenario: Lexicographic sort equals chronological sort
- **WHEN** messages are sorted lexicographically by their ULID IDs
- **THEN** the resulting order SHALL match chronological creation order

---

### Requirement: Cursor-Based Pagination

Message listing endpoints SHALL support cursor-based pagination using `before` and `after` query parameters, both of which accept ULID values. A `limit` parameter SHALL control page size with a server-defined maximum.

#### Scenario: Paginate messages before a cursor
- **WHEN** a client sends `GET /api/v1/channels/:id/messages?before=<ULID>&limit=50`
- **THEN** the server SHALL return up to 50 messages with IDs lexicographically less than the provided ULID, ordered from newest to oldest

#### Scenario: Paginate messages after a cursor
- **WHEN** a client sends `GET /api/v1/channels/:id/messages?after=<ULID>&limit=50`
- **THEN** the server SHALL return up to 50 messages with IDs lexicographically greater than the provided ULID, ordered from oldest to newest

#### Scenario: Default page size and maximum limit
- **WHEN** a client omits the `limit` parameter
- **THEN** the server SHALL use a default page size and MUST NOT return more than the server-defined maximum (e.g., 100) messages per request

---

### Requirement: Text Channel CRUD

The server SHALL support creating, reading, updating, and deleting text channels. Each channel MUST belong to a server (community) and MUST have a unique name within that server. Channel deletion MUST cascade to all contained messages, threads, pins, and associated MLS group state.

#### Scenario: Create a text channel
- **WHEN** a user with sufficient permissions sends a channel creation request
- **THEN** the server SHALL create the channel with a ULID identifier and return the channel object

#### Scenario: Update a text channel
- **WHEN** a user with sufficient permissions updates a channel's name or topic
- **THEN** the server SHALL persist the changes and emit a `CHANNEL_UPDATE` WebSocket event

#### Scenario: Delete a text channel
- **WHEN** a user with sufficient permissions deletes a channel
- **THEN** the server SHALL remove the channel and all associated messages, threads, pins, and MLS group state, and emit a `CHANNEL_DELETE` WebSocket event

---

### Requirement: Message Creation

Clients SHALL create messages by sending `POST /api/v1/channels/:id/messages` with an encrypted ciphertext blob. The server MUST assign a ULID, persist the message, and emit a `MESSAGE_CREATE` WebSocket event to all channel members.

#### Scenario: Successful message creation
- **WHEN** a client sends `POST /api/v1/channels/:id/messages` with a valid ciphertext payload
- **THEN** the server SHALL assign a ULID, persist the message, return the message object with HTTP 201, and emit a `MESSAGE_CREATE` event via WebSocket

#### Scenario: Unauthorized message creation
- **WHEN** a client without send-message permission sends `POST /api/v1/channels/:id/messages`
- **THEN** the server SHALL reject the request with HTTP 403

---

### Requirement: Message Editing

Clients SHALL edit their own messages by sending `PATCH /api/v1/channels/:id/messages/:id` with an updated ciphertext blob. The server MUST record that the message has been edited and emit a `MESSAGE_UPDATE` WebSocket event. Users MUST NOT edit messages authored by other users.

#### Scenario: Author edits own message
- **WHEN** a message author sends `PATCH /api/v1/channels/:id/messages/:id` with a new ciphertext blob
- **THEN** the server SHALL replace the stored ciphertext, set an `edited_at` timestamp, and emit a `MESSAGE_UPDATE` WebSocket event

#### Scenario: Non-author attempts to edit a message
- **WHEN** a user who is not the message author sends `PATCH /api/v1/channels/:id/messages/:id`
- **THEN** the server SHALL reject the request with HTTP 403

---

### Requirement: Message Deletion

Clients SHALL delete messages via `DELETE /api/v1/channels/:id/messages/:id`. The message author MAY delete their own messages. Users with the manage-messages permission MAY delete any message in the channel. The server MUST emit a `MESSAGE_DELETE` WebSocket event.

#### Scenario: Author deletes own message
- **WHEN** a message author sends `DELETE /api/v1/channels/:id/messages/:id`
- **THEN** the server SHALL remove the message and emit a `MESSAGE_DELETE` WebSocket event

#### Scenario: Moderator deletes another user's message
- **WHEN** a user with manage-messages permission sends `DELETE /api/v1/channels/:id/messages/:id` for a message authored by another user
- **THEN** the server SHALL remove the message and emit a `MESSAGE_DELETE` WebSocket event

#### Scenario: Unauthorized deletion attempt
- **WHEN** a user without manage-messages permission attempts to delete another user's message
- **THEN** the server SHALL reject the request with HTTP 403

---

### Requirement: Threads as Child Channels

Threads SHALL be implemented as child channels spawned from a parent message. A thread MUST reference its parent channel and parent message. Threads SHALL support the same message operations (create, edit, delete, paginate) as regular text channels. Creating a thread MUST emit a `THREAD_CREATE` WebSocket event.

#### Scenario: Create a thread from a message
- **WHEN** a user creates a thread on a specific message
- **THEN** the server SHALL create a child channel linked to the parent channel and parent message, and emit a `THREAD_CREATE` WebSocket event

#### Scenario: Send a message in a thread
- **WHEN** a client sends `POST /api/v1/channels/:thread_id/messages` with a ciphertext payload
- **THEN** the server SHALL create the message in the thread using the same message creation semantics as a regular channel

---

### Requirement: Encrypted Reactions

Reactions SHALL be encrypted within MLS application message payloads. The server SHALL store reactions as opaque ciphertext blobs and MUST NOT interpret reaction content. The server SHALL emit `MESSAGE_REACTION_ADD` and `MESSAGE_REACTION_REMOVE` WebSocket events containing the encrypted payloads.

#### Scenario: Add a reaction to a message
- **WHEN** a client adds an encrypted reaction to a message
- **THEN** the server SHALL store the opaque reaction blob, associate it with the message and user, and emit a `MESSAGE_REACTION_ADD` WebSocket event

#### Scenario: Remove a reaction from a message
- **WHEN** a client removes their reaction from a message
- **THEN** the server SHALL delete the reaction record and emit a `MESSAGE_REACTION_REMOVE` WebSocket event

---

### Requirement: Channel Pins

Each channel SHALL support pinning messages. Pinned messages MUST be retrievable as an ordered list. A server-defined maximum number of pins per channel SHALL be enforced. Pin and unpin operations SHALL emit `CHANNEL_PINS_UPDATE` WebSocket events.

#### Scenario: Pin a message
- **WHEN** a user with manage-messages permission pins a message in a channel
- **THEN** the server SHALL mark the message as pinned, add it to the channel's pin list, and emit a `CHANNEL_PINS_UPDATE` WebSocket event

#### Scenario: Unpin a message
- **WHEN** a user with manage-messages permission unpins a message
- **THEN** the server SHALL remove the message from the pin list and emit a `CHANNEL_PINS_UPDATE` WebSocket event

#### Scenario: Pin limit exceeded
- **WHEN** a user attempts to pin a message but the channel has reached the maximum pin count
- **THEN** the server SHALL reject the request with an appropriate error indicating the pin limit has been reached

---

### Requirement: Message Listing Endpoint

The server SHALL expose `GET /api/v1/channels/:id/messages` for retrieving messages. This endpoint MUST support `before`, `after`, and `limit` query parameters for cursor-based pagination. Responses MUST include the message array and pagination metadata.

#### Scenario: Fetch messages with default parameters
- **WHEN** a client sends `GET /api/v1/channels/:id/messages` with no query parameters
- **THEN** the server SHALL return the most recent messages up to the default limit, ordered from newest to oldest

#### Scenario: Fetch messages with before and limit
- **WHEN** a client sends `GET /api/v1/channels/:id/messages?before=01HQXYZ&limit=25`
- **THEN** the server SHALL return up to 25 messages older than the specified ULID cursor

---

### Requirement: Message Mutation Endpoints

The server SHALL expose `PATCH /api/v1/channels/:id/messages/:id` for editing messages and `DELETE /api/v1/channels/:id/messages/:id` for deleting messages. Both endpoints MUST validate authorization before performing the operation.

#### Scenario: Edit a message via PATCH
- **WHEN** a client sends `PATCH /api/v1/channels/:id/messages/:id` with an updated ciphertext blob
- **THEN** the server SHALL update the message if the client is the author, and return the updated message object

#### Scenario: Delete a message via DELETE
- **WHEN** a client sends `DELETE /api/v1/channels/:id/messages/:id`
- **THEN** the server SHALL delete the message if the client is authorized, and return HTTP 204

---

### Requirement: Typing Indicator Endpoint

The server SHALL expose `POST /api/v1/channels/:id/typing` as an ephemeral endpoint. Typing indicators MUST NOT be persisted to the database. The server SHALL write the typing state to NATS KV with a short TTL and fan out a `TYPING_START` WebSocket event to channel members.

#### Scenario: Send typing indicator
- **WHEN** a client sends `POST /api/v1/channels/:id/typing`
- **THEN** the server SHALL write the user's typing state to NATS KV with a TTL (e.g., 8 seconds) and emit a `TYPING_START` WebSocket event to other channel members

#### Scenario: Typing indicator expires
- **WHEN** a typing indicator's TTL expires in NATS KV without renewal
- **THEN** the typing state SHALL be automatically removed and clients SHALL treat the user as no longer typing

#### Scenario: Rate limiting on typing endpoint
- **WHEN** a client sends `POST /api/v1/channels/:id/typing` more frequently than the rate limit allows
- **THEN** the server SHALL reject excess requests with HTTP 429
