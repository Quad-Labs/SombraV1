## ADDED Requirements

### Requirement: P2P WebRTC for 1:1 Calls with TURN Fallback
The system SHALL attempt a direct peer-to-peer WebRTC connection for all 1:1 voice and video calls. If ICE candidate gathering fails to establish a direct connection within the ICE timeout, the system MUST fall back to routing media through a TURN relay server.

#### Scenario: Direct P2P connection succeeds
- **WHEN** two participants initiate a 1:1 call and both peers have compatible network conditions (no symmetric NAT, firewall allows UDP)
- **THEN** the system establishes a direct peer-to-peer WebRTC connection without routing media through a TURN relay

#### Scenario: P2P fails and TURN relay is used
- **WHEN** two participants initiate a 1:1 call and ICE candidate gathering fails to establish a direct connection
- **THEN** the system falls back to routing media through a TURN relay server and the call connects successfully via the relay

#### Scenario: No TURN relay available and P2P fails
- **WHEN** two participants initiate a 1:1 call, ICE candidate gathering fails, and no TURN relay server is configured or reachable
- **THEN** the call transitions to the Disconnecting state and both participants are notified that the call could not be established

### Requirement: Small Group Calls with SFU Preference and Mesh Fallback
For calls with 3-4 participants, the system SHALL use a mediasoup SFU if one is available. If no SFU is available, the system MUST fall back to full-mesh WebRTC where each participant maintains a direct peer connection to every other participant.

#### Scenario: 3-4 participants with SFU available
- **WHEN** a call has 3 or 4 participants and a mediasoup SFU worker is available
- **THEN** the system routes all media streams through the mediasoup SFU rather than establishing mesh connections

#### Scenario: 3-4 participants with no SFU available
- **WHEN** a call has 3 or 4 participants and no mediasoup SFU worker is available
- **THEN** the system establishes a full-mesh WebRTC topology where each participant maintains a direct peer connection to every other participant

#### Scenario: Third participant joins a P2P call with SFU available
- **WHEN** a 1:1 P2P call exists and a third participant joins while a mediasoup SFU worker is available
- **THEN** the system transitions the call from P2P topology to SFU topology, migrating all participants' media streams through the SFU

#### Scenario: Third participant joins a P2P call without SFU
- **WHEN** a 1:1 P2P call exists and a third participant joins while no mediasoup SFU worker is available
- **THEN** the system transitions the call from P2P topology to full-mesh topology, establishing direct peer connections between all three participants

### Requirement: Large Group Calls Require mediasoup SFU
For calls with 5 or more participants, the system MUST use a mediasoup SFU. The system SHALL NOT allow a 5th participant to join a call if no SFU is available.

#### Scenario: 5th participant joins with SFU available
- **WHEN** a call has 4 participants and a 5th participant attempts to join while a mediasoup SFU worker is available
- **THEN** the 5th participant is admitted to the call and all media is routed through the mediasoup SFU

#### Scenario: 5th participant rejected without SFU
- **WHEN** a call has 4 participants operating in mesh topology and a 5th participant attempts to join while no mediasoup SFU worker is available
- **THEN** the system rejects the 5th participant's join request with an error indicating that the SFU is required for calls with 5 or more participants

#### Scenario: Large group call operates exclusively through SFU
- **WHEN** a call has 5 or more participants
- **THEN** all media streams are routed exclusively through the mediasoup SFU and no direct peer-to-peer connections are established between participants

### Requirement: Call State Finite State Machine
Every call MUST follow a strict finite state machine with the states: Idle, Connecting, ICENegotiating, Connected, ICERestarting, Disconnecting. The only valid state transitions SHALL be: Idle to Connecting, Connecting to ICENegotiating, ICENegotiating to Connected, Connected to ICERestarting, ICERestarting to Connected, Connected to Disconnecting, Disconnecting to Idle. Any attempt to perform an invalid state transition MUST be rejected.

#### Scenario: Normal call lifecycle
- **WHEN** a user initiates a call from the Idle state
- **THEN** the call transitions through Idle, Connecting, ICENegotiating, Connected, and upon hangup through Disconnecting back to Idle, following each transition in strict order

#### Scenario: ICE restart cycle returns to Connected
- **WHEN** a call is in the Connected state and an ICE restart is triggered
- **THEN** the call transitions to ICERestarting and upon successful renegotiation transitions back to Connected

#### Scenario: Invalid state transition is rejected
- **WHEN** the system attempts to transition a call from Idle directly to Connected (skipping Connecting and ICENegotiating)
- **THEN** the transition is rejected and the call remains in the Idle state

#### Scenario: ICENegotiating failure transitions to Disconnecting
- **WHEN** a call is in the ICENegotiating state and ICE negotiation fails after exhausting all candidates and TURN fallback
- **THEN** the call transitions to Disconnecting and then to Idle

### Requirement: ICE Restart Handling for Mobile Roaming
The system SHALL detect network interface changes (Wi-Fi to cellular or cellular to Wi-Fi) and automatically trigger an ICE restart to re-establish the media connection without dropping the call. The ICE restart MUST be transparent to the user.

#### Scenario: Wi-Fi to cellular handoff
- **WHEN** a participant is on a Connected call via Wi-Fi and the device switches to a cellular network
- **THEN** the system detects the network change, transitions the call to ICERestarting, performs ICE candidate re-gathering on the new network interface, and transitions back to Connected without requiring the user to rejoin the call

#### Scenario: Cellular to Wi-Fi handoff
- **WHEN** a participant is on a Connected call via cellular and the device connects to a Wi-Fi network
- **THEN** the system detects the network change, triggers an ICE restart, and re-establishes the media path over Wi-Fi without dropping the call

#### Scenario: ICE restart fails after roaming
- **WHEN** a participant's network changes and the ICE restart fails to establish a new connection
- **THEN** the system transitions the call to Disconnecting and notifies the participant that the connection was lost

### Requirement: mediasoup Workers as External Processes
mediasoup workers SHALL run as external Node.js processes, separate from the Go server binary. Communication between the Go server and mediasoup workers MUST use JSON-RPC over stdin/stdout for bare metal deployments or HTTP/gRPC for Docker and cloud deployments.

#### Scenario: Bare metal mediasoup communication via stdin/stdout
- **WHEN** the Go server is running in Tier 1 bare metal mode and a mediasoup worker is available
- **THEN** the Go server communicates with the mediasoup worker process via JSON-RPC messages over stdin/stdout pipes

#### Scenario: Docker/cloud mediasoup communication via HTTP or gRPC
- **WHEN** the Go server is running in Tier 2+ Docker or cloud mode and a mediasoup worker is available
- **THEN** the Go server communicates with the mediasoup worker via HTTP or gRPC endpoints

#### Scenario: mediasoup worker crash does not crash the Go server
- **WHEN** a mediasoup worker process crashes or becomes unresponsive
- **THEN** the Go server continues operating, affected calls are transitioned to Disconnecting, and the server attempts to spawn a replacement worker

### Requirement: Go Server Manages All Room State
The Go server SHALL be the single source of truth for all call and room state, including participant lists, topology decisions, and call FSM state. mediasoup workers MUST be stateless and derive their behavior entirely from instructions received from the Go server.

#### Scenario: Room state persists through worker restart
- **WHEN** a mediasoup worker crashes and a replacement worker is spawned
- **THEN** the Go server reconstructs the media routing on the new worker using its authoritative room state without requiring participants to rejoin

#### Scenario: Worker has no local state
- **WHEN** the Go server sends a createTransport command to a mediasoup worker
- **THEN** the worker creates the transport based solely on the parameters in the command and does not maintain any room-level or participant-level state beyond what the Go server explicitly instructs

### Requirement: TURN Relay Provider Options
The system SHALL support three TURN relay options: self-hosted TURN servers configured via environment variables, Sombra-operated relay servers (paid service), and Cloudflare TURN as a fallback. The ICE server list MUST be ordered so that self-hosted is preferred first, then Sombra relay, then Cloudflare TURN.

#### Scenario: Self-hosted TURN is used when configured
- **WHEN** the server administrator has configured self-hosted TURN server credentials via environment variables
- **THEN** the ICE server list provided to clients includes the self-hosted TURN server as the first relay candidate

#### Scenario: Sombra relay used when no self-hosted TURN
- **WHEN** no self-hosted TURN server is configured and a Sombra relay subscription is active
- **THEN** the ICE server list provided to clients includes the Sombra relay server as the primary relay candidate

#### Scenario: Cloudflare TURN used as final fallback
- **WHEN** neither a self-hosted TURN server nor a Sombra relay subscription is available
- **THEN** the ICE server list provided to clients includes Cloudflare TURN as the fallback relay candidate

### Requirement: SFU Transition Threshold Communicated via CALL_CREATE
The SFU transition threshold (the participant count at which the call transitions from mesh to SFU topology) SHALL be communicated to all participants via the CALL_CREATE WebSocket event. The default threshold MUST be 4 participants.

#### Scenario: CALL_CREATE includes SFU threshold
- **WHEN** a new call is created and participants receive the CALL_CREATE WebSocket event
- **THEN** the event payload includes an `sfu_threshold` field indicating the participant count at which the call will transition to SFU topology

#### Scenario: Default SFU threshold is 4
- **WHEN** the server has no custom SFU threshold configuration
- **THEN** the `sfu_threshold` value in the CALL_CREATE event is 4

#### Scenario: Clients use threshold to anticipate topology change
- **WHEN** a client receives a CALL_CREATE event with `sfu_threshold` set to 4 and a 4th participant joins
- **THEN** the client prepares for SFU topology by expecting media to be rerouted through the SFU

### Requirement: WebSocket Events for Call Lifecycle
The server MUST emit the following WebSocket events for call lifecycle management: VOICE_STATE_UPDATE (when a participant's voice state changes, including join, leave, mute, deafen, or stream state), CALL_CREATE (when a new call is initiated, including topology and SFU threshold), and CALL_END (when a call terminates). These events MUST be delivered to all participants in the call and to members of the associated channel.

#### Scenario: VOICE_STATE_UPDATE on participant join
- **WHEN** a participant joins an active call
- **THEN** the server emits a VOICE_STATE_UPDATE event to all channel members containing the participant's user ID, channel ID, and voice state (connected, muted, deafened)

#### Scenario: VOICE_STATE_UPDATE on mute toggle
- **WHEN** a participant toggles their microphone mute
- **THEN** the server emits a VOICE_STATE_UPDATE event to all channel members reflecting the updated mute state

#### Scenario: CALL_CREATE on call initiation
- **WHEN** a user initiates a new call in a channel
- **THEN** the server emits a CALL_CREATE event to all channel members containing the call ID, channel ID, initiator user ID, call topology, and SFU threshold

#### Scenario: CALL_END on call termination
- **WHEN** the last participant leaves a call or the call is explicitly ended
- **THEN** the server emits a CALL_END event to all channel members containing the call ID, channel ID, and call duration
