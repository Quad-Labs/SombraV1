# Capability: presence-system

Online/idle/DND/invisible status and typing indicators via NATS KV with TTLs -- fully ephemeral, never hits disk.

## ADDED Requirements

### Requirement: Presence States

The system SHALL support four presence states: **online**, **idle**, **DND** (Do Not Disturb), and **invisible**. Each user MUST have exactly one active presence state at any time. The `invisible` state SHALL cause the user to appear as offline to other users while maintaining full functionality.

#### Scenario: User sets presence to online
- **WHEN** a user sets their presence to `online`
- **THEN** the server SHALL store the state in NATS KV and emit a `PRESENCE_UPDATE` WebSocket event to all users who share a server with the user

#### Scenario: User sets presence to idle
- **WHEN** a user sets their presence to `idle`
- **THEN** the server SHALL store the state in NATS KV and emit a `PRESENCE_UPDATE` WebSocket event to relevant users

#### Scenario: User sets presence to DND
- **WHEN** a user sets their presence to `dnd`
- **THEN** the server SHALL store the state in NATS KV and emit a `PRESENCE_UPDATE` WebSocket event to relevant users

#### Scenario: User sets presence to invisible
- **WHEN** a user sets their presence to `invisible`
- **THEN** the server SHALL store the state internally but MUST emit a `PRESENCE_UPDATE` WebSocket event indicating the user is `offline` to all other users

---

### Requirement: Presence Heartbeat Endpoint

The server SHALL expose `PUT /api/v1/users/@me/presence` for clients to set and maintain their presence state. Each request SHALL act as a heartbeat that refreshes the TTL on the presence entry in NATS KV. The server SHALL write the presence state to NATS KV and trigger a WebSocket fan-out via `PRESENCE_UPDATE` events when the state changes.

#### Scenario: Client sends presence heartbeat
- **WHEN** a client sends `PUT /api/v1/users/@me/presence` with `{"status": "online"}`
- **THEN** the server SHALL update the NATS KV entry for that user with the `online` status and refresh the TTL

#### Scenario: Presence state change triggers WebSocket event
- **WHEN** a client changes their presence from `online` to `dnd` via `PUT /api/v1/users/@me/presence`
- **THEN** the server SHALL emit a `PRESENCE_UPDATE` WebSocket event to all relevant users reflecting the new state

#### Scenario: Heartbeat without state change refreshes TTL only
- **WHEN** a client sends a heartbeat with the same status as currently stored
- **THEN** the server SHALL refresh the NATS KV TTL but MUST NOT emit a redundant `PRESENCE_UPDATE` WebSocket event

---

### Requirement: Typing Indicators

Typing indicators SHALL be triggered via `POST /api/v1/channels/:id/typing`. The server SHALL write the typing state to NATS KV with a short TTL (e.g., 8 seconds) and emit a `TYPING_START` WebSocket event to other members of the channel. Typing indicators MUST NOT be persisted to the database.

#### Scenario: User starts typing
- **WHEN** a client sends `POST /api/v1/channels/:id/typing`
- **THEN** the server SHALL write a typing entry to NATS KV with a short TTL and emit a `TYPING_START` WebSocket event to other members of that channel

#### Scenario: Typing state expires naturally
- **WHEN** a user stops typing and the NATS KV TTL for their typing entry expires
- **THEN** the typing state SHALL be automatically removed without any explicit cleanup action

---

### Requirement: NATS KV Storage with TTLs

All presence data (status and typing indicators) SHALL be stored exclusively in NATS KV with time-to-live (TTL) values. Presence status entries MUST have a TTL that requires periodic heartbeat renewal (e.g., 60 seconds). Typing indicator entries MUST have a shorter TTL (e.g., 8 seconds). Expired entries SHALL be automatically evicted by NATS.

#### Scenario: Presence entry expires without heartbeat
- **WHEN** a client disconnects or stops sending heartbeats and the NATS KV TTL expires
- **THEN** the presence entry SHALL be automatically evicted, and the server SHALL emit a `PRESENCE_UPDATE` WebSocket event indicating the user is `offline`

#### Scenario: NATS KV handles TTL eviction
- **WHEN** a NATS KV entry's TTL expires
- **THEN** NATS SHALL automatically remove the entry without any server-side polling or cleanup process

---

### Requirement: Ephemeral Storage Only

Presence and typing data MUST NOT be persisted to the database (SQLite or PostgreSQL). All presence state SHALL exist only in NATS KV in-memory storage. A server restart SHALL result in all presence states resetting to offline, which is the correct behavior since no clients will have active heartbeats.

#### Scenario: Server restart clears all presence
- **WHEN** the server process restarts
- **THEN** all presence states SHALL be cleared (since NATS KV is ephemeral) and all users SHALL appear offline until they re-establish heartbeats

#### Scenario: No presence data in database
- **WHEN** the database is inspected at any time
- **THEN** there SHALL be no tables, rows, or columns storing presence or typing indicator data

---

### Requirement: Rate Limiting on Presence and Typing Updates

The server MUST enforce rate limits on presence and typing update endpoints to prevent abuse. The `PUT /api/v1/users/@me/presence` endpoint SHALL be rate-limited per user. The `POST /api/v1/channels/:id/typing` endpoint SHALL be rate-limited per user per channel. Excess requests MUST be rejected with HTTP 429.

#### Scenario: Presence update rate limit exceeded
- **WHEN** a client sends `PUT /api/v1/users/@me/presence` more frequently than the allowed rate
- **THEN** the server SHALL reject excess requests with HTTP 429 Too Many Requests

#### Scenario: Typing indicator rate limit exceeded
- **WHEN** a client sends `POST /api/v1/channels/:id/typing` more frequently than the allowed rate for that channel
- **THEN** the server SHALL reject excess requests with HTTP 429 Too Many Requests

#### Scenario: Rate limits are per-user scoped
- **WHEN** user A is rate-limited on the typing endpoint
- **THEN** user B SHALL NOT be affected and SHALL be able to send typing indicators normally

---

### Requirement: PRESENCE_UPDATE and TYPING_START WebSocket Events

The server SHALL emit `PRESENCE_UPDATE` WebSocket events when a user's presence state changes or when their presence entry expires. The server SHALL emit `TYPING_START` WebSocket events when a user begins typing in a channel. Both events MUST be delivered only to users who have visibility of the relevant user or channel.

#### Scenario: PRESENCE_UPDATE event delivered on state change
- **WHEN** a user's presence state changes from `online` to `idle`
- **THEN** the server SHALL emit a `PRESENCE_UPDATE` WebSocket event containing the user ID and new status to all users who share at least one server with that user

#### Scenario: PRESENCE_UPDATE event delivered on disconnect
- **WHEN** a user's presence TTL expires (indicating disconnect)
- **THEN** the server SHALL emit a `PRESENCE_UPDATE` WebSocket event with status `offline` to all relevant users

#### Scenario: TYPING_START event scoped to channel members
- **WHEN** a user triggers a typing indicator in a channel
- **THEN** the server SHALL emit a `TYPING_START` WebSocket event containing the user ID and channel ID only to other members of that channel

#### Scenario: Invisible user does not leak presence
- **WHEN** an invisible user's presence is queried or a `PRESENCE_UPDATE` would be emitted
- **THEN** the event SHALL indicate status `offline` and MUST NOT reveal the user's actual `invisible` state
