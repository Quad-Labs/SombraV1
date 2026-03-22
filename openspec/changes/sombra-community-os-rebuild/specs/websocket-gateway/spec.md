## ADDED Requirements

### Requirement: Gateway Connection Endpoint
Clients SHALL connect to the WebSocket gateway at `wss://host/api/v1/gateway`. The server MUST upgrade the HTTP connection to a WebSocket connection at this path. Non-WebSocket requests to this path SHALL be rejected.

#### Scenario: Successful WebSocket upgrade
- **WHEN** a client sends an HTTP upgrade request to `/api/v1/gateway` with valid WebSocket headers
- **THEN** the server upgrades the connection to a persistent WebSocket connection

#### Scenario: Non-WebSocket request rejected
- **WHEN** a client sends a regular HTTP GET request to `/api/v1/gateway` without WebSocket upgrade headers
- **THEN** the server responds with a 400 Bad Request or 426 Upgrade Required status

### Requirement: HELLO Handshake Initiation
Immediately after a WebSocket connection is established, the server SHALL send a HELLO message to the client. The HELLO message MUST include the heartbeat interval (in milliseconds) that the client is expected to use.

#### Scenario: Server sends HELLO on connect
- **WHEN** a client establishes a WebSocket connection to the gateway
- **THEN** the server sends a HELLO message as the first frame, containing the heartbeat interval value

#### Scenario: HELLO includes heartbeat interval
- **WHEN** the server sends the HELLO message
- **THEN** the message payload includes a `heartbeat_interval` field with a positive integer value in milliseconds

### Requirement: Client IDENTIFY with Auth Token
After receiving HELLO, the client SHALL send an IDENTIFY message containing a valid session token. If the token is invalid or missing, the server MUST close the connection with an appropriate close code. Alternatively, the client MAY send a RESUME message instead of IDENTIFY to reconnect to a previous session.

#### Scenario: Successful IDENTIFY
- **WHEN** a client sends an IDENTIFY message with a valid session token after receiving HELLO
- **THEN** the server authenticates the client, associates the connection with the user, and proceeds to send READY

#### Scenario: Invalid IDENTIFY token
- **WHEN** a client sends an IDENTIFY message with an invalid or expired session token
- **THEN** the server closes the WebSocket connection with a 4001 (or equivalent) close code indicating authentication failure

#### Scenario: Client sends RESUME instead of IDENTIFY
- **WHEN** a client sends a RESUME message with a valid session ID and last received sequence number after receiving HELLO
- **THEN** the server attempts to replay buffered events from the given sequence number onward

### Requirement: READY Payload with Initial State
After a successful IDENTIFY, the server SHALL send a READY message containing the initial state snapshot. The READY payload MUST include the authenticated user object, the list of servers the user belongs to, channels within those servers, and the user's current presence state.

#### Scenario: READY contains full initial state
- **WHEN** the server sends a READY message after successful IDENTIFY
- **THEN** the payload includes the user object, an array of server objects the user is a member of, an array of channel objects within those servers, and the user's presence state

#### Scenario: READY is sent only after successful authentication
- **WHEN** a client has not yet sent a valid IDENTIFY message
- **THEN** the server does not send a READY message

### Requirement: Event Envelope Format
All server-to-client event messages SHALL use a standardized envelope format containing the following fields: `op` (operation type, e.g., "EVENT"), `t` (event type name, e.g., "MESSAGE_CREATE"), `s` (monotonically increasing sequence number), and `d` (event-specific data payload).

#### Scenario: Event envelope structure
- **WHEN** the server sends a MESSAGE_CREATE event to a client
- **THEN** the message is formatted as `{ "op": "EVENT", "t": "MESSAGE_CREATE", "s": <sequence_number>, "d": { ... } }` where `s` is a monotonically increasing integer and `d` contains the message data

#### Scenario: Sequence numbers increment per connection
- **WHEN** the server sends multiple events to the same client connection
- **THEN** each successive event has a sequence number exactly one greater than the previous event's sequence number

### Requirement: Client-to-Server Operations
The server SHALL accept the following client-to-server operations: IDENTIFY (authenticate the connection), RESUME (reconnect to a previous session), HEARTBEAT (keep the connection alive), and PRESENCE_UPDATE (change the user's presence status).

#### Scenario: Client sends HEARTBEAT
- **WHEN** a client sends a HEARTBEAT operation to the server
- **THEN** the server acknowledges receipt by responding with a HEARTBEAT_ACK and resets the connection's heartbeat timeout timer

#### Scenario: Client sends PRESENCE_UPDATE
- **WHEN** a client sends a PRESENCE_UPDATE operation with a new status (e.g., "idle", "dnd", "invisible")
- **THEN** the server updates the user's presence state and broadcasts a PRESENCE_UPDATE event to all relevant connected clients

#### Scenario: Unknown client operation rejected
- **WHEN** a client sends a message with an unrecognized operation type
- **THEN** the server ignores the message or closes the connection with an appropriate error close code

### Requirement: Server-to-Client Event Types
The server SHALL support broadcasting the following event types to connected clients: MESSAGE_CREATE, MESSAGE_UPDATE, MESSAGE_DELETE, TYPING_START, PRESENCE_UPDATE, CHANNEL_CREATE, CHANNEL_UPDATE, CHANNEL_DELETE, MEMBER_JOIN, MEMBER_LEAVE, MEMBER_UPDATE, MLS_COMMIT, MLS_WELCOME, MLS_PROPOSAL, VOICE_STATE_UPDATE, CALL_CREATE, and CALL_END. Each event type MUST carry an appropriate data payload in the `d` field.

#### Scenario: MESSAGE_CREATE event delivered
- **WHEN** a new message is created in a channel
- **THEN** the server sends a MESSAGE_CREATE event to all connected clients who have permission to read that channel, with the message object in the `d` field

#### Scenario: TYPING_START event delivered
- **WHEN** a user begins typing in a channel
- **THEN** the server sends a TYPING_START event to all connected clients in that channel with the user ID and channel ID in the `d` field

#### Scenario: MLS events routed to group members
- **WHEN** an MLS_COMMIT, MLS_WELCOME, or MLS_PROPOSAL event is generated for an MLS group
- **THEN** the server routes the event only to connected clients who are members of that MLS group

#### Scenario: VOICE_STATE_UPDATE event delivered
- **WHEN** a user joins, leaves, or changes their voice state in a voice channel
- **THEN** the server sends a VOICE_STATE_UPDATE event to all connected clients in the relevant server

#### Scenario: CALL_CREATE and CALL_END events delivered
- **WHEN** a call is initiated or terminated
- **THEN** the server sends the corresponding CALL_CREATE or CALL_END event to all relevant connected clients

#### Scenario: MEMBER_JOIN and MEMBER_LEAVE events delivered
- **WHEN** a user joins or leaves a server
- **THEN** the server sends a MEMBER_JOIN or MEMBER_LEAVE event to all connected clients in that server

### Requirement: Sequence-Based Resume
The server SHALL buffer events for each authenticated connection for 60 seconds after the connection is lost. When a client sends a RESUME with a session ID and last sequence number, the server MUST replay all buffered events with sequence numbers greater than the provided value. If the buffer has expired or the session ID is invalid, the server SHALL reject the RESUME and require a fresh IDENTIFY.

#### Scenario: Successful resume within buffer window
- **WHEN** a client disconnects and reconnects within 60 seconds, sending a RESUME with the last received sequence number
- **THEN** the server replays all events buffered since that sequence number and resumes normal event delivery without requiring a new IDENTIFY

#### Scenario: Resume after buffer expiry fails
- **WHEN** a client disconnects and reconnects after more than 60 seconds, sending a RESUME
- **THEN** the server rejects the RESUME with an appropriate close code indicating the session is no longer resumable, and the client must send a fresh IDENTIFY

#### Scenario: Resume with invalid session ID fails
- **WHEN** a client sends a RESUME with a session ID that the server does not recognize
- **THEN** the server rejects the RESUME and closes the connection with an error close code

#### Scenario: No duplicate events on resume
- **WHEN** a client resumes and the server replays buffered events
- **THEN** only events with sequence numbers strictly greater than the client's last acknowledged sequence number are replayed — no duplicates are sent

### Requirement: Heartbeat Mechanism
The server SHALL require clients to send periodic HEARTBEAT messages at the interval specified in the HELLO message. If a client fails to send a HEARTBEAT within the expected interval (plus a reasonable grace period), the server MUST close the connection. The heartbeat interval SHALL be configurable by the server.

#### Scenario: Client heartbeat keeps connection alive
- **WHEN** a client sends HEARTBEAT messages at or before the interval specified in HELLO
- **THEN** the server keeps the connection open and responds with HEARTBEAT_ACK

#### Scenario: Missed heartbeat closes connection
- **WHEN** a client fails to send a HEARTBEAT within the interval plus grace period
- **THEN** the server closes the WebSocket connection with a close code indicating missed heartbeat

#### Scenario: Heartbeat interval is configurable
- **WHEN** the server is configured with a specific heartbeat interval (e.g., via environment variable or internal configuration)
- **THEN** the HELLO message sent to clients contains that configured interval value
