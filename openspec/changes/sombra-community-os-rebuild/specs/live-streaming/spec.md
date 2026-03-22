## ADDED Requirements

### Requirement: Go Live Capture Sources
The Go Live feature SHALL support three capture sources: screen capture (entire display), window capture (individual application window), and camera capture (webcam feed). Each capture source MUST produce a media stream that is sent to the mediasoup SFU for distribution to viewers.

#### Scenario: Screen capture starts streaming
- **WHEN** a user selects "Go Live" and chooses a screen as the capture source
- **THEN** the system captures the entire display output and sends the video stream to the mediasoup SFU for distribution to channel viewers

#### Scenario: Window capture starts streaming
- **WHEN** a user selects "Go Live" and chooses a specific application window as the capture source
- **THEN** the system captures only the selected window's content and sends the video stream to the mediasoup SFU

#### Scenario: Camera capture starts streaming
- **WHEN** a user selects "Go Live" and chooses a webcam as the capture source
- **THEN** the system captures the webcam feed and sends the video stream to the mediasoup SFU

### Requirement: VP9 SVC Adaptive Quality
All Go Live streams MUST be encoded using VP9 with Scalable Video Coding (SVC) combining spatial and temporal scalability layers. The encoder SHALL produce a single encode with multiple quality levels, and the SFU MUST selectively forward the appropriate layers to each viewer based on their available bandwidth.

#### Scenario: Single encode produces multiple quality levels
- **WHEN** a streamer starts a Go Live session with VP9 SVC encoding
- **THEN** the encoder produces a single bitstream containing spatial layers (e.g., 360p, 720p, 1080p) and temporal layers (e.g., 15fps, 30fps) that the SFU can independently forward

#### Scenario: Viewer on low bandwidth receives lower layers
- **WHEN** a viewer is watching a Go Live stream and their available bandwidth drops below the threshold for the highest spatial layer
- **THEN** the SFU forwards only the lower spatial and temporal layers to that viewer while other viewers with sufficient bandwidth continue receiving higher layers

#### Scenario: Viewer selects quality manually
- **WHEN** a viewer manually selects a lower quality level from the quality selector
- **THEN** the SFU forwards only the layers corresponding to the selected quality level regardless of the viewer's available bandwidth

### Requirement: WHIP Endpoint for External Ingest
The server SHALL expose a WebRTC-HTTP Ingestion Protocol (WHIP) endpoint that allows external streaming tools such as OBS Studio to send media streams into a channel. Authentication to the WHIP endpoint MUST use a stream key that the channel owner generates. The WHIP endpoint MUST accept standard WebRTC offers and respond with WebRTC answers.

#### Scenario: OBS connects via WHIP with valid stream key
- **WHEN** an external tool sends an HTTP POST to the WHIP endpoint with a valid stream key and a WebRTC SDP offer
- **THEN** the server authenticates the stream key, creates a mediasoup transport, and responds with a WebRTC SDP answer, establishing the ingest connection

#### Scenario: WHIP rejects invalid stream key
- **WHEN** an external tool sends an HTTP POST to the WHIP endpoint with an invalid or expired stream key
- **THEN** the server responds with a 401 Unauthorized status and does not create a media transport

#### Scenario: WHIP stream is distributed to channel viewers
- **WHEN** an external tool has established a WHIP ingest connection and is sending media
- **THEN** the media is routed through the mediasoup SFU and distributed to all viewers in the associated channel, identical to a native Go Live stream

#### Scenario: Stream key generation by channel owner
- **WHEN** a channel owner requests a new stream key for WHIP ingest
- **THEN** the server generates a unique, cryptographically random stream key associated with that channel and returns it to the channel owner

### Requirement: System Audio Capture via Tauri and cpal
On desktop platforms running Tauri, the system SHALL capture system audio (application audio output) using cpal native loopback audio capture. The captured system audio MUST be mixed or sent as a separate audio track alongside the screen or window capture stream.

#### Scenario: System audio captured during screen share on Tauri
- **WHEN** a user starts a Go Live screen capture on the Tauri desktop client and enables system audio
- **THEN** the system uses cpal to capture the native audio loopback and sends the system audio as an audio track alongside the screen capture video

#### Scenario: System audio not available on browser
- **WHEN** a user starts a Go Live screen capture on the web browser client
- **THEN** the system audio capture falls back to the browser's getDisplayMedia audio capabilities, which may or may not include system audio depending on browser and OS support

#### Scenario: System audio toggle during stream
- **WHEN** a streamer toggles system audio capture off during an active Go Live session
- **THEN** the system audio track is muted or removed from the stream without interrupting the video capture

### Requirement: Multi-Source Streaming
The system SHALL support simultaneous streaming from multiple capture sources. A streamer MUST be able to send both a camera feed and a screen capture as separate producers to the mediasoup SFU at the same time. Each source SHALL be an independent media producer.

#### Scenario: Camera and screen shared simultaneously
- **WHEN** a streamer enables both camera and screen capture during a Go Live session
- **THEN** the system creates two separate mediasoup producers (one for camera, one for screen) and both streams are available to viewers simultaneously

#### Scenario: Viewer sees both sources
- **WHEN** a viewer is watching a stream that has both camera and screen producers active
- **THEN** the viewer's client receives both media streams and can display them according to the selected layout

#### Scenario: One source stopped independently
- **WHEN** a streamer stops their camera capture while screen capture is still active
- **THEN** only the camera producer is closed and the screen capture continues streaming without interruption

### Requirement: Viewer UI Layouts and Quality Selector
The viewer UI MUST provide at least the following display options: Picture-in-Picture (PiP) mode where one source overlays another, side-by-side layout where multiple sources are displayed adjacent to each other, and a quality selector that allows the viewer to choose between available VP9 SVC quality levels.

#### Scenario: PiP layout with camera over screen
- **WHEN** a viewer selects the PiP layout while watching a multi-source stream
- **THEN** the screen capture is displayed as the primary view and the camera feed is displayed as a smaller overlay in a corner of the viewport

#### Scenario: Side-by-side layout
- **WHEN** a viewer selects the side-by-side layout while watching a multi-source stream
- **THEN** the screen capture and camera feed are displayed adjacent to each other, each occupying approximately half the viewport

#### Scenario: Quality selector lists available levels
- **WHEN** a viewer opens the quality selector during a Go Live stream
- **THEN** the selector displays all available quality levels derived from the VP9 SVC spatial layers (e.g., 360p, 720p, 1080p) and includes an "Auto" option for bandwidth-adaptive selection

### Requirement: Streaming Always Requires mediasoup SFU
All Go Live streaming sessions MUST route media through a mediasoup SFU. Streaming SHALL NOT operate in peer-to-peer or mesh mode under any circumstances. If no mediasoup SFU worker is available, the system MUST reject the Go Live request with an error.

#### Scenario: Go Live rejected without SFU
- **WHEN** a user attempts to start a Go Live session and no mediasoup SFU worker is available
- **THEN** the system rejects the request with an error indicating that the SFU is required for streaming and the Go Live session does not start

#### Scenario: Stream always routes through SFU
- **WHEN** a Go Live session is active with a single viewer
- **THEN** the media is routed through the mediasoup SFU even though only one viewer is present, and no peer-to-peer connection is established

#### Scenario: SFU worker loss terminates stream
- **WHEN** a Go Live session is active and the mediasoup SFU worker handling the stream crashes
- **THEN** the stream is terminated and all participants (streamer and viewers) are notified that the stream has ended due to an infrastructure failure

### Requirement: Browser Capture via getDisplayMedia and Tauri via Native Capture
On web browser clients, screen and window capture SHALL use the getDisplayMedia Web API. On Tauri desktop clients, capture SHALL use native screen capture APIs provided by the Tauri platform combined with cpal for audio. The system MUST abstract capture source selection so that the user experience is consistent across both platforms.

#### Scenario: Browser client uses getDisplayMedia
- **WHEN** a user starts a Go Live session on the web browser client
- **THEN** the system invokes the getDisplayMedia API to present the browser's native screen/window picker and captures the selected source

#### Scenario: Tauri client uses native capture
- **WHEN** a user starts a Go Live session on the Tauri desktop client
- **THEN** the system uses Tauri's native screen capture APIs to enumerate available screens and windows and captures the selected source without invoking getDisplayMedia

#### Scenario: Consistent capture picker UI
- **WHEN** a user starts a Go Live session on either browser or Tauri
- **THEN** the capture source selection UI presents screens and windows in a consistent manner, allowing the user to select a source and preview it before starting the stream
