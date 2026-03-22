# Capability: tauri-shell

Tauri v2 application shell — Windows, macOS, Linux, iOS, Android, Steam Deck (Flatpak), native push notifications, system tray, auto-update, deep links, audio device management, screen capture, system audio loopback.

## ADDED Requirements

### Requirement: Six-Platform Delivery
The Tauri v2 shell SHALL produce installable application packages for six platforms: Windows (`.msi`), macOS (`.dmg`), Linux (`.AppImage`, `.deb`, and `.flatpak`), iOS (`.ipa`), Android (`.apk` and `.aab`), and Steam Deck (Flatpak running on WebKitGTK). All six platform builds MUST be produced from the same Tauri project configuration and SolidJS codebase.

#### Scenario: Windows build produces .msi
- **WHEN** the CI pipeline runs the Tauri build for Windows
- **THEN** a `.msi` installer package is produced that installs the Sombra application on Windows 10 and later

#### Scenario: macOS build produces .dmg
- **WHEN** the CI pipeline runs the Tauri build for macOS
- **THEN** a `.dmg` disk image is produced that installs the Sombra application on macOS 12 (Monterey) and later

#### Scenario: Linux build produces multiple formats
- **WHEN** the CI pipeline runs the Tauri build for Linux
- **THEN** `.AppImage`, `.deb`, and `.flatpak` packages are produced for broad Linux distribution compatibility

#### Scenario: iOS build produces .ipa
- **WHEN** the CI pipeline runs the Tauri build for iOS
- **THEN** an `.ipa` archive is produced suitable for TestFlight distribution and App Store submission

#### Scenario: Android build produces .apk and .aab
- **WHEN** the CI pipeline runs the Tauri build for Android
- **THEN** both an `.apk` (sideload) and `.aab` (Google Play) are produced

#### Scenario: Steam Deck build is a Flatpak on WebKitGTK
- **WHEN** the Sombra application is packaged for Steam Deck
- **THEN** it is distributed as a Flatpak that runs on WebKitGTK, the rendering engine available on Steam Deck's SteamOS

#### Scenario: Single codebase for all platforms
- **WHEN** the platform-specific build configurations are inspected
- **THEN** they all reference the same SolidJS frontend and Rust crypto core with platform differences handled by Tauri's build system and conditional compilation

---

### Requirement: Native Push Notifications
The Tauri shell SHALL integrate with platform-native push notification APIs to deliver notifications when the application is in the background or closed. On iOS, notifications MUST use APNs. On Android, notifications MUST use FCM. On desktop platforms, notifications MUST use the OS notification system (Windows Toast, macOS Notification Center, Linux libnotify).

#### Scenario: iOS receives push via APNs
- **WHEN** the Sombra server dispatches a push notification for an offline iOS user
- **THEN** the notification is delivered via APNs and appears in the iOS notification center

#### Scenario: Android receives push via FCM
- **WHEN** the Sombra server dispatches a push notification for an offline Android user
- **THEN** the notification is delivered via FCM and appears in the Android notification shade

#### Scenario: Desktop notification displayed
- **WHEN** a real-time event triggers a notification while the desktop app is minimized or in the background
- **THEN** the Tauri shell displays a native OS notification (Toast on Windows, Notification Center on macOS, libnotify on Linux)

---

### Requirement: System Tray Integration
The Tauri shell on desktop platforms SHALL provide a system tray icon with a context menu. The tray icon MUST display unread message count (badge). The context menu SHALL include options to open the main window, set status (online, idle, DND, invisible), and quit the application. Closing the main window MUST minimize to the system tray rather than quitting the application.

#### Scenario: Tray icon shows unread badge
- **WHEN** the user has unread messages and the main window is minimized
- **THEN** the system tray icon displays a badge or overlay indicating the unread count

#### Scenario: Close minimizes to tray
- **WHEN** the user clicks the window close button on the main window
- **THEN** the application minimizes to the system tray instead of quitting, and the tray icon remains visible

#### Scenario: Tray context menu opens window
- **WHEN** the user right-clicks the system tray icon and selects "Open Sombra"
- **THEN** the main application window is restored and brought to the foreground

#### Scenario: Quit from tray
- **WHEN** the user selects "Quit" from the tray context menu
- **THEN** the application performs a graceful shutdown and exits completely

---

### Requirement: Auto-Update Mechanism
The Tauri shell SHALL include an auto-update mechanism that checks for new versions on startup and at configurable intervals. When an update is available, the application MUST notify the user and provide an option to download and install the update. The update process MUST be atomic — either the update completes fully or the previous version is preserved.

#### Scenario: Update check on startup
- **WHEN** the application starts
- **THEN** it checks the update server for a newer version and, if available, notifies the user with a non-blocking prompt

#### Scenario: User-initiated update
- **WHEN** the user accepts the update prompt
- **THEN** the application downloads the update package, verifies its integrity, applies the update, and restarts with the new version

#### Scenario: Failed update preserves current version
- **WHEN** an update download or installation fails midway
- **THEN** the current version of the application remains intact and functional, and the user is notified of the failure

---

### Requirement: Deep Link Handling
The Tauri shell SHALL register as the handler for the `sombra://` protocol on all supported platforms. Deep links MUST be parsed and routed to the appropriate UI view. The protocol MUST support at minimum: server invites (`sombra://invite/{code}`), channel navigation (`sombra://server/{id}/channel/{id}`), and user profiles (`sombra://user/{id}`).

#### Scenario: Invite deep link opens invite acceptance flow
- **WHEN** a user clicks a `sombra://invite/abc123` link while the application is running
- **THEN** the application navigates to the invite acceptance UI for invite code `abc123`

#### Scenario: Deep link launches application if closed
- **WHEN** a user clicks a `sombra://` deep link while the application is not running
- **THEN** the OS launches the Sombra application and passes the deep link URL, which the app processes after startup

#### Scenario: Channel deep link navigates to channel
- **WHEN** a user clicks `sombra://server/01HQXYZ/channel/01HQABC`
- **THEN** the application navigates to server `01HQXYZ` and opens channel `01HQABC` in the message view

---

### Requirement: Audio Device Management
The Tauri shell SHALL provide audio device enumeration, selection, and configuration for voice communication. The shell MUST expose input (microphone) and output (speaker/headphone) device lists, allow the user to select preferred devices, and detect device hot-plug events (e.g., connecting a headset). Audio device management MUST be available on all desktop and mobile platforms.

#### Scenario: Enumerate audio devices
- **WHEN** the user opens the audio settings panel
- **THEN** the application lists all available audio input devices and audio output devices with their names

#### Scenario: Select preferred audio device
- **WHEN** the user selects a specific microphone and speaker from the device list
- **THEN** the application routes all voice input through the selected microphone and all voice output through the selected speaker

#### Scenario: Hot-plug detection
- **WHEN** the user connects or disconnects an audio device (e.g., plugs in a USB headset)
- **THEN** the application detects the change, updates the device list, and optionally switches to the newly connected device based on user preference

---

### Requirement: Screen Capture for Go Live Streaming
The Tauri shell SHALL provide screen and window capture capabilities for Go Live streaming. The shell MUST enumerate capturable screens and windows, allow the user to select a capture source, and provide a video frame stream to the SFrame encryption pipeline. Capture MUST support both full-screen and individual window sources.

#### Scenario: Enumerate capturable sources
- **WHEN** the user initiates a Go Live session
- **THEN** the application presents a list of available screens (monitors) and individual application windows that can be captured

#### Scenario: Capture selected screen
- **WHEN** the user selects a screen or window for capture
- **THEN** the Tauri shell begins capturing video frames from the selected source and feeds them to the media pipeline for SFrame encryption and WebRTC transmission

#### Scenario: Capture stops on session end
- **WHEN** the user ends the Go Live session
- **THEN** screen capture stops immediately and all capture resources are released

---

### Requirement: System Audio Loopback via cpal
The Tauri shell SHALL capture system audio output (loopback) using the `cpal` Rust crate for Go Live and screen sharing scenarios. System audio MUST be mixable with microphone input. The loopback capture MUST work on Windows (WASAPI loopback), macOS (CoreAudio aggregate device), and Linux (PulseAudio/PipeWire monitor source). Mobile platforms are excluded from this requirement.

#### Scenario: Capture system audio on Windows
- **WHEN** a user enables "Share System Audio" during Go Live on Windows
- **THEN** the Tauri shell captures system audio output via WASAPI loopback using cpal and mixes it into the outgoing audio stream

#### Scenario: Capture system audio on macOS
- **WHEN** a user enables "Share System Audio" during Go Live on macOS
- **THEN** the Tauri shell captures system audio output via CoreAudio aggregate device using cpal and mixes it into the outgoing audio stream

#### Scenario: Capture system audio on Linux
- **WHEN** a user enables "Share System Audio" during Go Live on Linux
- **THEN** the Tauri shell captures system audio output via PulseAudio or PipeWire monitor source using cpal and mixes it into the outgoing audio stream

#### Scenario: Mix system audio with microphone
- **WHEN** both system audio loopback and microphone input are active during a Go Live session
- **THEN** the audio pipeline mixes both sources into a single outgoing audio stream

---

### Requirement: Native Crypto Core on Desktop and Mobile
On Tauri desktop and mobile targets, the crypto core MUST run as a native Rust library — not as WASM. The Tauri shell SHALL load the crypto core via direct FFI (Tauri invoke commands), providing native CPU performance for all MLS, SFrame, and file encryption operations. WASM compilation of the crypto core is reserved exclusively for the browser-only web client.

#### Scenario: Desktop crypto runs natively
- **WHEN** the Sombra application runs on a desktop platform (Windows, macOS, Linux)
- **THEN** the crypto core executes as a native Rust library with full CPU performance and no WASM overhead

#### Scenario: Mobile crypto runs natively
- **WHEN** the Sombra application runs on a mobile platform (iOS, Android)
- **THEN** the crypto core executes as a native Rust library linked into the Tauri mobile runtime

#### Scenario: Tauri invoke calls crypto core directly
- **WHEN** the SDK makes a crypto operation request on a Tauri target
- **THEN** the request is routed via Tauri's invoke mechanism directly to the native Rust crypto core function — no WASM instantiation or Web Worker is involved

---

### Requirement: Resource Efficiency Targets
The Tauri shell MUST target approximately 30MB of RAM usage at idle and approximately 5-15MB for the application bundle (installed size). These targets represent the design goal enabled by Tauri's architecture (no bundled Chromium, native webview). Actual measurements MAY vary by platform, but SHALL remain within the same order of magnitude as these targets.

#### Scenario: RAM usage at idle
- **WHEN** the Sombra application is running on a desktop platform with no active voice calls or streaming sessions
- **THEN** the total process memory usage SHALL be approximately 30MB, remaining well below the ~300MB typical of Electron-based applications

#### Scenario: Bundle size
- **WHEN** the production installer package is built for any desktop platform
- **THEN** the installed application size SHALL be approximately 5-15MB, well below the ~150MB typical of Electron-based applications

---

### Requirement: Steam Deck WebKitGTK Caveat — Capability Detection with Graceful Fallback
On Steam Deck (Flatpak on WebKitGTK), the Tauri shell MUST detect WebRTC and media API capabilities at runtime, as WebKitGTK's support for WebRTC, Insertable Streams, and advanced codecs varies across versions. The shell MUST gracefully degrade functionality when capabilities are absent rather than crashing or showing blank UI. Fallback behavior MUST include VP8 instead of VP9 when VP9 is unsupported, audio-only mode when video codecs are unavailable, and disabling SFrame if Insertable Streams are not available.

#### Scenario: Detect VP9 support
- **WHEN** the Sombra application starts on Steam Deck
- **THEN** it probes WebKitGTK for VP9 codec support and, if unavailable, falls back to VP8 for all video encoding and decoding

#### Scenario: Fallback to audio-only
- **WHEN** WebKitGTK on Steam Deck does not support any video codecs compatible with Sombra's media pipeline
- **THEN** the application enters audio-only mode for voice channels and displays a notice to the user explaining the limitation

#### Scenario: Disable SFrame if Insertable Streams unavailable
- **WHEN** WebKitGTK does not support Insertable Streams (required for SFrame E2EE on media)
- **THEN** the application disables SFrame encryption for voice/video on this device, clearly warns the user that voice/video is unencrypted, and continues to function for text E2EE

#### Scenario: Text messaging works regardless
- **WHEN** the Sombra application runs on Steam Deck with limited WebKitGTK capabilities
- **THEN** all text messaging, MLS E2EE, file sharing, and non-media features work fully regardless of WebRTC capability limitations
