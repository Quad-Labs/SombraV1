# Capability: crypto-core

Rust crypto core using OpenMLS — MLS key management, SFrame encode/decode, file encryption, KeyPackage generation, local encrypted vault (SQLite/sled on Tauri, IndexedDB on WASM), Web Worker execution for WASM target.

## ADDED Requirements

### Requirement: OpenMLS Wrapper for MLS Key Management
The crypto core SHALL provide an OpenMLS-based wrapper that handles MLS group creation, joining via Welcome messages, Commit/Proposal processing, application message encryption and decryption, and epoch key derivation. All MLS operations MUST conform to RFC 9420. The wrapper MUST expose a stable Rust API consumed by the TypeScript SDK via FFI (Tauri) or WASM bindings (web).

#### Scenario: Create a new MLS group
- **WHEN** the SDK requests creation of a new MLS group for a channel
- **THEN** the crypto core creates the group using OpenMLS, generates the initial epoch secret, and returns the serialized group state and initial Commit

#### Scenario: Process a Welcome message
- **WHEN** the crypto core receives a Welcome message for a group the user has been invited to
- **THEN** it processes the Welcome via OpenMLS, establishes the group state on this device, and makes the group available for subsequent encrypt/decrypt operations

#### Scenario: Encrypt an application message
- **WHEN** the SDK passes plaintext to the crypto core for a specific MLS group
- **THEN** the crypto core encrypts the plaintext as an MLS application message and returns the ciphertext blob

#### Scenario: Decrypt an application message
- **WHEN** the SDK passes a ciphertext blob to the crypto core for a specific MLS group
- **THEN** the crypto core decrypts the message via OpenMLS and returns the plaintext, or returns an error if decryption fails (e.g., wrong epoch, corrupted data)

---

### Requirement: SFrame Encode/Decode for Voice and Video
The crypto core SHALL implement SFrame (RFC 9605) encoding and decoding for voice and video frame-level encryption. SFrame symmetric keys MUST be derived from MLS epoch secrets via `MLS_Exporter`. The implementation MUST support epoch tagging so that receivers can identify which key to use for decryption, and MUST retain the previous epoch's key for a 10-second grace period during key rotation.

#### Scenario: Encode a media frame
- **WHEN** the SDK passes a raw voice or video frame to the crypto core with the current MLS group context
- **THEN** the crypto core encrypts the frame using the SFrame key derived from the current MLS epoch and returns the encrypted frame with the epoch tag

#### Scenario: Decode a media frame
- **WHEN** the SDK passes an encrypted SFrame frame to the crypto core
- **THEN** the crypto core reads the epoch tag, selects the correct key, decrypts the frame, and returns the raw media data

#### Scenario: Grace period for old epoch key
- **WHEN** an MLS epoch rotation occurs and a receiver receives a frame encrypted with the previous epoch's key within 10 seconds of the rotation
- **THEN** the crypto core successfully decrypts the frame using the retained old epoch key

#### Scenario: Old epoch key expired
- **WHEN** a frame encrypted with a previous epoch's key arrives more than 10 seconds after the epoch rotation
- **THEN** the crypto core rejects the frame with a key-expired error

---

### Requirement: File Encryption with Streaming Support
The crypto core SHALL support file encryption and decryption using streaming (chunked) processing. Large files MUST NOT be loaded entirely into memory. The encryption MUST use keys derived from the MLS group context. The streaming implementation MUST support 5MB chunk sizes to match the blob-storage upload chunk size.

#### Scenario: Encrypt a large file in streaming chunks
- **WHEN** the SDK requests encryption of a file larger than 5MB
- **THEN** the crypto core reads and encrypts the file in 5MB streaming chunks without loading the entire file into memory

#### Scenario: Decrypt a large file in streaming chunks
- **WHEN** the SDK requests decryption of an encrypted file
- **THEN** the crypto core reads and decrypts the file in streaming chunks, producing the original plaintext output progressively

#### Scenario: Encryption keys derived from MLS group
- **WHEN** a file is encrypted for a specific channel
- **THEN** the encryption key is derived from the channel's MLS group context, ensuring only group members can decrypt

---

### Requirement: KeyPackage Batch Generation
The crypto core SHALL generate MLS KeyPackages in batches of 100. KeyPackages MUST be pre-generated and uploaded to the server so that other users can add this device to MLS groups. The crypto core MUST track the number of remaining unused KeyPackages and trigger a new batch generation when the count falls below a threshold.

#### Scenario: Initial KeyPackage batch on device setup
- **WHEN** a user sets up a new device or recovers an account
- **THEN** the crypto core generates a batch of 100 MLS KeyPackages and the SDK uploads them to the server

#### Scenario: KeyPackage replenishment
- **WHEN** the number of unused KeyPackages on the server falls below a configured threshold (e.g., 20)
- **THEN** the crypto core generates a new batch of 100 KeyPackages for upload

#### Scenario: Each KeyPackage is single-use
- **WHEN** a KeyPackage is consumed by another user to add this device to an MLS group
- **THEN** that KeyPackage MUST NOT be reused for any other group join operation

---

### Requirement: StorageProvider Trait with Dual Backends
The crypto core SHALL define a `StorageProvider` trait for persisting cryptographic state (MLS group state, keys, KeyPackages). The trait MUST have two implementations: SQLite/sled for Tauri native targets (desktop and mobile) and IndexedDB for WASM web targets. The storage MUST encrypt all persisted data at rest.

#### Scenario: Native target uses SQLite/sled storage
- **WHEN** the crypto core is compiled as a native library for Tauri (desktop or mobile)
- **THEN** it uses the SQLite or sled `StorageProvider` implementation to persist cryptographic state to the local filesystem

#### Scenario: WASM target uses IndexedDB storage
- **WHEN** the crypto core is compiled to WASM for the web browser
- **THEN** it uses the IndexedDB `StorageProvider` implementation to persist cryptographic state in the browser's IndexedDB

#### Scenario: Stored data is encrypted at rest
- **WHEN** cryptographic state is written to the storage provider
- **THEN** the data is encrypted before being persisted, and decrypted when read back, using a device-local encryption key

---

### Requirement: Dual Compilation Targets
The crypto core MUST compile to two targets: a native Rust library (`.so`/`.dylib`/`.dll`) for Tauri desktop and mobile platforms, and a WASM module (`.wasm`) for web browsers. The same Rust source code SHALL produce both targets, with platform-specific behavior isolated behind conditional compilation (`#[cfg]`) and the `StorageProvider` trait.

#### Scenario: Native compilation for Tauri
- **WHEN** a developer runs the Tauri build pipeline targeting a desktop or mobile platform
- **THEN** the crypto core compiles to a native library that Tauri loads via FFI with full native performance

#### Scenario: WASM compilation for web
- **WHEN** a developer runs `wasm-pack build` targeting the web
- **THEN** the crypto core compiles to a `.wasm` module with JavaScript bindings that can be loaded in a browser

#### Scenario: Same source, both targets
- **WHEN** both native and WASM builds are compared
- **THEN** they are produced from the same Rust source crate with platform differences handled by `#[cfg(target_arch = "wasm32")]` guards and the `StorageProvider` trait

---

### Requirement: WASM Execution in Web Worker
The WASM crypto core MUST run inside a Web Worker when executing in a web browser. The crypto core SHALL NOT execute on the main UI thread. Communication between the SDK (main thread) and the crypto core (Web Worker) MUST use the `postMessage` API with structured clone transfer for binary data.

#### Scenario: Crypto operations do not block UI
- **WHEN** the SDK requests an MLS encryption or decryption operation in the web browser
- **THEN** the operation executes in a Web Worker and the main thread remains responsive for UI interactions

#### Scenario: postMessage communication
- **WHEN** the SDK sends a crypto operation request to the Web Worker
- **THEN** the request is serialized via postMessage, the Web Worker executes the operation using the WASM module, and the result is returned via postMessage

#### Scenario: Binary data transferred efficiently
- **WHEN** large binary data (e.g., encrypted file chunks) is passed between the main thread and the Web Worker
- **THEN** the data is transferred using the Transferable interface (ArrayBuffer transfer) to avoid copying

---

### Requirement: WASM RNG Bound to window.crypto
The WASM build MUST explicitly bind its random number generator to `window.crypto.getRandomValues` via the `getrandom` crate's `js` feature flag. The crypto core MUST NOT use any other source of randomness when running as WASM in a browser.

#### Scenario: WASM uses browser CSPRNG
- **WHEN** the WASM crypto core needs random bytes for key generation, nonces, or any cryptographic operation
- **THEN** it obtains them from `window.crypto.getRandomValues` via the getrandom js feature binding

#### Scenario: getrandom js feature is enabled in Cargo.toml
- **WHEN** the crypto core's `Cargo.toml` is inspected
- **THEN** it includes `getrandom = { version = "...", features = ["js"] }` as a dependency for the WASM target

---

### Requirement: Path-Passing for Large File Operations
On Tauri native targets, the crypto core SHALL accept file paths directly from the UI layer for large file encryption and upload operations. The Rust code MUST read the file from the filesystem, encrypt it in streaming chunks, and upload it to the server directly. The UI layer SHALL receive only progress events — it MUST NOT read file contents into JavaScript memory.

#### Scenario: SolidJS passes file path to Rust
- **WHEN** a user selects a large file for upload in the SolidJS UI on a Tauri target
- **THEN** the UI passes the file path (not file contents) to the Rust crypto core via Tauri invoke

#### Scenario: Rust reads, encrypts, and uploads directly
- **WHEN** the Rust crypto core receives a file path for encryption and upload
- **THEN** it reads the file from disk, encrypts it in streaming 5MB chunks, and uploads each chunk to the server without passing file data through the TypeScript layer

#### Scenario: UI receives progress events only
- **WHEN** the Rust crypto core is processing a large file
- **THEN** it emits progress events (bytes processed, total size, percentage) to the UI via Tauri event system, and the UI displays a progress indicator without handling file data

---

### Requirement: TypeScript Layer Never Touches Raw Keys
Raw cryptographic keys — including MLS epoch secrets, SFrame keys, identity private keys, and file encryption keys — MUST NOT be exposed to the TypeScript layer. All key material SHALL remain within the Rust crypto core's memory space. The SDK MUST only hold opaque references (e.g., group IDs, session handles) that the crypto core uses to look up the actual keys internally.

#### Scenario: SDK holds opaque group reference
- **WHEN** the SDK needs to encrypt a message for a channel
- **THEN** it passes the channel/group ID to the crypto core, which internally resolves the ID to the MLS group state and performs encryption without returning the key

#### Scenario: No key material in JavaScript heap
- **WHEN** the crypto core returns results to the SDK (e.g., ciphertext, decrypted plaintext)
- **THEN** the returned data does not include any raw key material — only the operation's output

#### Scenario: Web Worker isolation enforces key containment
- **WHEN** the crypto core runs in a Web Worker on the web target
- **THEN** key material exists only in the Web Worker's memory space and is never transferred to the main thread via postMessage

---

### Requirement: Cargo.toml getrandom js Feature for WASM
The crypto core's `Cargo.toml` MUST include `getrandom` with the `js` feature enabled for the WASM target. This dependency MUST be declared at the crate level (not just transitively) to ensure the CSPRNG is properly bound to `window.crypto.getRandomValues` when compiled to WASM.

#### Scenario: Cargo.toml contains correct getrandom dependency
- **WHEN** the `crypto/Cargo.toml` file is inspected
- **THEN** it contains a dependency entry equivalent to `getrandom = { version = "...", features = ["js"] }` either as a direct dependency or under `[target.'cfg(target_arch = "wasm32")'.dependencies]`

#### Scenario: WASM build succeeds with CSPRNG
- **WHEN** the crypto core is compiled to WASM and the resulting module generates random numbers
- **THEN** the random number generation succeeds by delegating to `window.crypto.getRandomValues` rather than panicking with a "getrandom: unsupported target" error
