# Capability: mls-e2ee

MLS (RFC 9420) Delivery Service -- KeyPackage store, Commit/Proposal/Welcome routing, epoch monotonicity validation, desync recovery, multi-device leaf nodes, device revocation, channel history modes.

## ADDED Requirements

### Requirement: KeyPackage Upload

Clients SHALL upload batches of KeyPackages via `POST /api/v1/users/@me/keys/packages`. Each batch SHOULD contain approximately 100 KeyPackages. The server MUST store uploaded KeyPackages and associate them with the uploading user and device. KeyPackages are one-time use and MUST be consumed upon fetch.

#### Scenario: Upload a batch of KeyPackages
- **WHEN** a client sends `POST /api/v1/users/@me/keys/packages` with a batch of 100 KeyPackages
- **THEN** the server SHALL store all KeyPackages and associate them with the authenticated user's device

#### Scenario: Reject invalid KeyPackage batch
- **WHEN** a client uploads KeyPackages with malformed or invalid MLS encoding
- **THEN** the server SHALL reject the request with HTTP 400

---

### Requirement: KeyPackage Fetch and Consumption

The server SHALL expose `GET /api/v1/users/:id/keys/packages` to fetch one KeyPackage for a target user. The returned KeyPackage MUST be consumed (deleted from the store) atomically upon fetch so that it cannot be reused. If the target user has no remaining KeyPackages, the server SHALL return an appropriate error.

#### Scenario: Fetch and consume a KeyPackage
- **WHEN** a client sends `GET /api/v1/users/:id/keys/packages`
- **THEN** the server SHALL return exactly one KeyPackage and atomically delete it from the store so it cannot be fetched again

#### Scenario: No KeyPackages remaining
- **WHEN** a client requests a KeyPackage for a user who has zero remaining KeyPackages
- **THEN** the server SHALL return an error indicating no KeyPackages are available

---

### Requirement: KeyPackage Low Warning

The server SHALL emit a `KEY_PACKAGES_LOW` WebSocket event to a device when that device has fewer than 10 KeyPackages remaining in the store. This event MUST prompt the client to upload a new batch.

#### Scenario: KeyPackages drop below threshold
- **WHEN** a device's KeyPackage count drops below 10 after a consumption event
- **THEN** the server SHALL send a `KEY_PACKAGES_LOW` WebSocket event to that device, including the remaining count

#### Scenario: Client replenishes KeyPackages after warning
- **WHEN** a client receives a `KEY_PACKAGES_LOW` event and uploads a new batch
- **THEN** the server SHALL store the new KeyPackages and the device's count SHALL reflect the replenished total

---

### Requirement: MLS Commit Submission with Epoch Monotonicity

Clients SHALL submit MLS Commits via `POST /api/v1/channels/:id/mls/commit`. The server MUST validate epoch monotonicity: the Commit's epoch MUST equal the channel's current epoch plus one (`new_epoch = current_epoch + 1`). If validation passes, the server SHALL update the stored epoch and distribute the Commit to all group members. If validation fails, the server MUST reject the Commit.

#### Scenario: Valid Commit with correct epoch
- **WHEN** a client sends `POST /api/v1/channels/:id/mls/commit` with `epoch = current_epoch + 1`
- **THEN** the server SHALL accept the Commit, update the channel's epoch to the new value, and distribute the Commit to all group members via WebSocket

#### Scenario: Commit with stale epoch
- **WHEN** a client sends a Commit with an epoch that does not equal `current_epoch + 1`
- **THEN** the server SHALL reject the Commit with an error indicating epoch mismatch

#### Scenario: Concurrent Commit conflict
- **WHEN** two clients simultaneously submit Commits for the same epoch
- **THEN** the server SHALL accept only the first Commit that arrives and reject the second with an epoch mismatch error

---

### Requirement: MLS Proposal Submission

Clients SHALL submit MLS Proposals (Add, Remove, Update) via `POST /api/v1/channels/:id/mls/proposal`. The server SHALL store the Proposal and distribute it to group members. Proposals MUST be applied only when included in a subsequent Commit.

#### Scenario: Submit an Add Proposal
- **WHEN** a client sends `POST /api/v1/channels/:id/mls/proposal` with an Add Proposal containing a target user's KeyPackage
- **THEN** the server SHALL store the Proposal and distribute it to current group members via WebSocket

#### Scenario: Submit a Remove Proposal
- **WHEN** a client sends `POST /api/v1/channels/:id/mls/proposal` with a Remove Proposal for a specific leaf node
- **THEN** the server SHALL store the Proposal and distribute it to current group members via WebSocket

#### Scenario: Submit an Update Proposal
- **WHEN** a client sends `POST /api/v1/channels/:id/mls/proposal` with an Update Proposal for the sender's own leaf node
- **THEN** the server SHALL store the Proposal and distribute it to current group members via WebSocket

---

### Requirement: MLS Welcome Message Routing

Clients SHALL submit MLS Welcome messages for newly added members via `POST /api/v1/channels/:id/mls/welcome`. The server MUST store Welcome messages for up to 7 days. The server SHALL deliver stored Welcome messages to the target user when they connect or on demand.

#### Scenario: Submit a Welcome for a new member
- **WHEN** a client sends `POST /api/v1/channels/:id/mls/welcome` after a Commit that adds a new member
- **THEN** the server SHALL store the Welcome and deliver it to the new member via WebSocket or on their next connection

#### Scenario: Welcome message expires after 7 days
- **WHEN** a Welcome message has been stored for more than 7 days without being retrieved
- **THEN** the server SHALL delete the expired Welcome message

---

### Requirement: Desync Recovery

When a client detects an epoch mismatch (its local epoch does not match the server's current epoch), the server MUST support recovery. The server SHALL buffer the last 100 Commits per channel. If the client's epoch gap is within the buffer, it SHALL request and replay the missing Commits. If the gap exceeds the buffer, the client MUST perform a full rejoin.

#### Scenario: Recover via buffered Commits
- **WHEN** a client detects its epoch is behind and the missing Commits are within the server's 100-Commit buffer
- **THEN** the client SHALL request the missing Commits from the server and replay them sequentially to catch up to the current epoch

#### Scenario: Full rejoin required
- **WHEN** a client detects its epoch is behind and the gap exceeds the server's 100-Commit buffer
- **THEN** the client SHALL initiate a full rejoin via `POST /api/v1/channels/:id/mls/rejoin-request`

---

### Requirement: Rejoin Request

The server SHALL expose `POST /api/v1/channels/:id/mls/rejoin-request` for clients that need to rejoin an MLS group after unrecoverable desync. The server SHALL notify a random online group member to generate a new Welcome message for the requesting client.

#### Scenario: Client requests rejoin
- **WHEN** a desynced client sends `POST /api/v1/channels/:id/mls/rejoin-request`
- **THEN** the server SHALL select a random online group member and send them a `MLS_REJOIN_NEEDED` WebSocket event instructing them to generate a Welcome for the requesting client

#### Scenario: No online members available for rejoin
- **WHEN** a client sends a rejoin request but no other group members are online
- **THEN** the server SHALL queue the rejoin request and process it when a group member comes online

---

### Requirement: One MLSGroup per Channel

Each channel SHALL map to exactly one MLS group. The MLS group MUST be created when the channel is created (or when the first member joins an E2EE channel). All message encryption within a channel MUST use that channel's MLS group context.

#### Scenario: Channel creation establishes MLS group
- **WHEN** a new E2EE channel is created
- **THEN** the server SHALL initialize an MLS group for that channel and the creator SHALL be the first member

#### Scenario: All channel messages use the channel's MLS group
- **WHEN** a client sends a message in a channel
- **THEN** the message MUST be encrypted using the MLS group associated with that specific channel

---

### Requirement: Multi-Device Support

Each user device SHALL be represented as a separate MLS LeafNode within the group's ratchet tree. A single user with multiple devices MUST have one LeafNode per device. KeyPackages SHALL be device-specific. All devices of a group member SHALL receive all MLS messages for that group.

#### Scenario: User joins group from two devices
- **WHEN** a user with two active devices joins an MLS group
- **THEN** both devices SHALL be added as separate LeafNodes in the ratchet tree, each with their own KeyPackage

#### Scenario: Message delivered to all user devices
- **WHEN** an MLS application message is sent to the group
- **THEN** all LeafNodes (devices) of every group member SHALL be able to decrypt the message independently

---

### Requirement: Device Revocation

When a device is revoked, the server MUST invalidate the device's authentication tokens, delete all of that device's remaining KeyPackages, and issue MLS Remove Proposals for the device's LeafNode in every group the device belongs to. The Remove Proposals MUST be followed by Commits to advance the epoch, ensuring the revoked device loses access to future messages.

#### Scenario: User revokes a device
- **WHEN** a user revokes one of their devices
- **THEN** the server SHALL invalidate the device's tokens, delete its KeyPackages, and issue MLS Remove Proposals for that device's LeafNode in all groups it belongs to

#### Scenario: Epoch advances after device removal
- **WHEN** MLS Remove Proposals for a revoked device are committed
- **THEN** the group epoch SHALL advance and the revoked device SHALL be unable to decrypt any messages sent after the epoch advance

---

### Requirement: Channel History Modes

Each channel SHALL support one of four history modes that control how new members access past messages. The history mode MUST be set at channel creation and MAY be changed by a channel administrator.

The four modes are:

1. **Full history** -- existing members SHALL share epoch keys with new members so they can decrypt the full message history.
2. **From join** -- new members SHALL only have access to messages from the current epoch onward; prior epoch keys MUST NOT be shared.
3. **Ephemeral** -- clients SHALL enforce a client-side TTL on decrypted messages; the server stores ciphertext normally but clients discard plaintext after the TTL.
4. **Trusted server** -- the channel MAY opt in to plaintext storage on the server, enabling server-side search and other features.

#### Scenario: New member joins a full-history channel
- **WHEN** a new member joins a channel with full-history mode
- **THEN** existing members SHALL share past epoch keys with the new member so they can decrypt the full message archive

#### Scenario: New member joins a from-join channel
- **WHEN** a new member joins a channel with from-join mode
- **THEN** the new member SHALL only be able to decrypt messages from the current epoch onward; no prior epoch keys SHALL be shared

#### Scenario: Ephemeral channel message expiry
- **WHEN** a client decrypts a message in an ephemeral channel
- **THEN** the client SHALL delete the decrypted plaintext after the configured TTL expires

#### Scenario: Trusted server plaintext storage
- **WHEN** a channel is configured with trusted-server mode
- **THEN** the server MAY store messages in plaintext and provide server-side features such as search and indexing

---

### Requirement: KeyPackage and MLS State Retention

KeyPackages SHALL be stored until consumed. MLS Commits SHALL be buffered for 24 hours for desync recovery. Welcome messages SHALL be stored for 7 days. Epoch state (group state snapshots) SHALL be stored indefinitely for the lifetime of the channel.

#### Scenario: KeyPackage persists until consumed
- **WHEN** a KeyPackage is uploaded
- **THEN** it SHALL remain in the store until it is fetched and consumed, with no time-based expiry

#### Scenario: Commit buffer expires after 24 hours
- **WHEN** an MLS Commit has been buffered for more than 24 hours
- **THEN** the server SHALL remove it from the buffer

#### Scenario: Welcome expires after 7 days
- **WHEN** a Welcome message has been stored for more than 7 days
- **THEN** the server SHALL delete the expired Welcome

#### Scenario: Epoch state persists indefinitely
- **WHEN** a channel's MLS group advances through multiple epochs
- **THEN** the server SHALL retain epoch state for all epochs for the lifetime of the channel
