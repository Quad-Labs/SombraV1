## ADDED Requirements

### Requirement: APNs and FCM Push Dispatch
The server SHALL support sending push notifications via Apple Push Notification service (APNs) for iOS devices and Firebase Cloud Messaging (FCM) for Android devices. The push dispatcher MUST select the correct transport (APNs or FCM) based on the device platform recorded in the device's push token registration. APNs credentials and FCM credentials MUST be configurable via environment variables.

#### Scenario: Push notification is delivered via APNs for iOS device
- **WHEN** the server needs to send a push notification to an iOS device
- **THEN** the push dispatcher sends the notification via APNs using the configured APNs credentials and the device's APNs push token

#### Scenario: Push notification is delivered via FCM for Android device
- **WHEN** the server needs to send a push notification to an Android device
- **THEN** the push dispatcher sends the notification via FCM using the configured FCM credentials and the device's FCM registration token

#### Scenario: Push is skipped when credentials are not configured
- **WHEN** the server has no APNs or FCM credentials configured and a push notification is triggered
- **THEN** the push dispatcher logs a warning and skips the notification without crashing or blocking message delivery

### Requirement: Generic Payloads Only
All push notification payloads MUST contain only generic, content-free information. The payload SHALL include at most the channel ID where a new message was received (e.g., "New message in [Channel ID]") and the server identity. The payload MUST NOT include the message body, sender name, attachment previews, or any data derived from the MLS ciphertext. The server cannot read message content and SHALL NOT attempt to include it.

#### Scenario: Push payload contains only channel ID
- **WHEN** a push notification is dispatched for a new message in a channel
- **THEN** the notification payload contains only a generic indicator such as `"New message in [Channel ID]"` and the server identifier, with no message content, sender name, or preview text

#### Scenario: Push payload does not contain message body
- **WHEN** the MLS Delivery Service routes a message to an offline user and triggers a push notification
- **THEN** the push payload contains zero bytes of the MLS ciphertext, zero bytes of any decrypted content, and no metadata about the message sender

#### Scenario: Push payload does not contain attachment information
- **WHEN** a message with an encrypted file attachment triggers a push notification for an offline user
- **THEN** the push payload does not include the attachment file name, file size, MIME type, or any thumbnail data

### Requirement: Triggered by Offline MLS Routing
Push notifications SHALL be triggered when the MLS Delivery Service attempts to route a message to a user who has no active WebSocket connections. The push dispatcher MUST be invoked only after the Delivery Service determines the recipient is offline. A user SHALL be considered offline when none of their registered devices have an active WebSocket connection to the server.

#### Scenario: Offline user receives push notification
- **WHEN** the MLS Delivery Service routes a message to a user with no active WebSocket connections on any device
- **THEN** the push dispatcher sends a push notification to all of the user's registered push-enabled devices

#### Scenario: Online user does not receive push notification
- **WHEN** the MLS Delivery Service routes a message to a user who has at least one active WebSocket connection
- **THEN** no push notification is dispatched, as the message is delivered via the WebSocket connection in real time

#### Scenario: Multi-device user with mixed online/offline
- **WHEN** a user has three devices, one connected via WebSocket and two offline
- **THEN** no push notification is sent because the user has at least one active connection and is not considered offline

### Requirement: Device Push Token Storage
Device push tokens (APNs tokens for iOS, FCM registration tokens for Android) SHALL be stored in the relational database alongside the device's MLS-related information (KeyPackages, device ID). Each device record MUST include the push token, push platform (APNs or FCM), and a timestamp of when the token was last updated. Users MUST be able to register and update push tokens via a dedicated API endpoint.

#### Scenario: Device registers a push token
- **WHEN** a mobile client sends a `POST /api/v1/devices/:deviceId/push-token` request with a valid push token and platform identifier
- **THEN** the server stores the push token and platform in the device record alongside the device's MLS KeyPackage information

#### Scenario: Push token is updated on app restart
- **WHEN** a mobile client obtains a new push token from APNs or FCM (e.g., after an app reinstall) and sends an update request
- **THEN** the server replaces the old push token with the new one and updates the last-updated timestamp

#### Scenario: Push token is removed on device logout
- **WHEN** a user logs out of a device or deregisters a device
- **THEN** the push token is removed from the device record and no further push notifications are sent to that device

#### Scenario: Invalid push token is handled gracefully
- **WHEN** the push dispatcher attempts to send a notification to a device with an expired or invalid push token and the APNs/FCM service returns an error
- **THEN** the server marks the push token as invalid and does not retry delivery to that token until it is updated by the client

### Requirement: No Sensitive Content in Push Payloads
Push notification payloads MUST comply with E2EE guarantees by containing zero sensitive information. The server SHALL NOT include any of the following in push payloads: message ciphertext, decrypted message plaintext, sender user ID, sender display name, channel name, attachment metadata, reaction data, or thread context. This requirement SHALL be enforced regardless of the notification trigger.

#### Scenario: Push payload passes E2EE compliance audit
- **WHEN** all fields of a push notification payload are inspected
- **THEN** the payload contains only: a notification type identifier, the channel ID (opaque ULID), the server ID, and a generic title string; no other data is present

#### Scenario: Channel name is not included
- **WHEN** a push notification is generated for a message in a channel named "secret-planning"
- **THEN** the push payload contains only the channel's opaque ULID, not the human-readable channel name

#### Scenario: Sender identity is not included
- **WHEN** a push notification is generated for a message sent by user "alice"
- **THEN** the push payload does not contain the sender's user ID, display name, avatar URL, or any other identifying information about the sender
