## Why

Sombra is a privacy-first Community Operating System — the hub for everything a community needs: real-time communication, streaming, shared services, and identity — all E2EE by default and self-hostable with "Minecraft server" ease. The current codebase inherits Revolt's architecture: 19 Rust crates, 37+ git submodules, 8 programming languages, 15 Docker containers for local dev, 30-60 second incremental rebuilds, and AGPL-3.0 licensing. This is unsustainable for a 1-3 person team building an ambitious, differentiated product. A ground-up rebuild is needed to achieve the performance, multi-platform, and developer velocity goals while gaining licensing freedom under Apache-2.0.

## What Changes

- **BREAKING**: Replace entire Revolt/Stoat-derived codebase with clean-slate implementation — zero code carryover
- **BREAKING**: Backend rewritten from 19 Rust crates / 8 separate servers to a single Go binary with modular internal subsystems
- **BREAKING**: Database switches from MongoDB (required) to SQLite (embedded, Tier 1) or PostgreSQL (cloud, Tier 2+)
- **BREAKING**: Desktop app switches from Electron (~300MB RAM) to Tauri v2 (~30MB RAM)
- **BREAKING**: Mobile apps switch from 2 native apps (Kotlin + Swift, ~70K LOC) to Tauri v2 (shared codebase with web)
- Add MLS (RFC 9420) E2EE for all text messages and files — server routes ciphertext it cannot decrypt
- Add SFrame (RFC 9605) E2EE for all voice/video streams
- Add WebRTC voice/video with P2P, mesh fallback, and mediasoup SFU for groups
- Add Go Live streaming with VP9 SVC and WHIP ingest for OBS/external tools
- Add ActivityPub federation for forum channels only (Lemmy-compatible)
- Add embedded OIDC provider for "Log in with Sombra" flows
- Add community app orchestration (Docker/K8s) with internal reverse proxy
- Add bot API as first-class citizens using the same REST + WebSocket API
- Add 4-tier hosting model: Minecraft Mode (zero-config binary) → Docker Pro → Sombra Network (managed) → Own Cloud
- Reduce language count from 8 to 3 (Go, TypeScript, Rust for crypto)
- Frontend stays SolidJS but gains Panda CSS glassmorphism theme and Tauri v2 shell for 6-platform delivery
- License changes from AGPL-3.0 (Revolt copyright) to Apache-2.0 (Sombra copyright)

## Capabilities

### New Capabilities

- `server-core`: Single Go binary server skeleton — chi router, middleware chain (Recoverer → RequestID → DevLogger → RateLimiter → Authenticator → RBAC), embedded NATS JetStream event bus, environment-variable configuration
- `authentication`: User registration, login, sessions, multi-device management, account recovery via BIP-39 mnemonic, future TOTP/WebAuthn support
- `rbac-permissions`: Role-based access control with server-level roles and channel-level permission overrides, evaluated on every API call and WebSocket fan-out
- `websocket-gateway`: Persistent WebSocket connections for real-time event delivery — HELLO/IDENTIFY/READY handshake, heartbeat, sequence-based resume (60s buffer), full event envelope protocol
- `messaging`: Text channels, messages (ULID-ordered, ciphertext blobs), threads, reactions, message editing/deletion, pins, cursor-based pagination
- `mls-e2ee`: MLS (RFC 9420) Delivery Service — KeyPackage store, Commit/Proposal/Welcome routing, epoch monotonicity validation, desync recovery, multi-device leaf nodes, device revocation, channel history modes (full/from-join/ephemeral/trusted-server)
- `sframe-media-encryption`: SFrame (RFC 9605) frame-level encryption for voice/video — symmetric keys derived from MLS group secrets via MLS_Exporter, epoch tagging for key rotation, 10-second grace period for old keys
- `presence-system`: Online/idle/DND/invisible status and typing indicators via NATS KV with TTLs — fully ephemeral, never hits disk
- `voice-calling`: WebRTC voice/video — P2P for 1:1, mesh fallback for 2-4 participants, mediasoup SFU for 5+, TURN relay integration, ICE restart handling, strict call state FSM
- `live-streaming`: Go Live screen/window/camera capture, VP9 SVC adaptive quality, WHIP endpoint for OBS/external ingest, system audio capture via Tauri/cpal, multi-source streaming
- `activitypub-federation`: Forum-only ActivityPub federation — WebFinger discovery, HTTP Signatures, Lemmy-compatible JSON-LD, per-forum toggle, transactional outbox delivery, content sanitization
- `blob-storage`: StorageProvider interface — filesystem (Tier 1) or S3-compatible (Tier 2+), streaming 5MB chunk encryption, public bucket for federated media
- `oidc-provider`: Embedded OAuth2/OIDC provider (ory/fosite) — "Log in with Sombra" for community apps, bots, and third-party integrations with user identity + roles as OIDC claims
- `community-app-orchestrator`: ServiceManager interface with Docker SDK and K8s client implementations — app lifecycle management, internal reverse proxy routing `/app/{name}/*`, OIDC auth at proxy layer, SSRF prevention via network isolation
- `client-sdk`: @sombra/sdk standalone TypeScript package — REST client, WebSocket manager, state store, message cache, MLS session management, offline outbox queue, crypto bridge to Rust core
- `crypto-core`: Rust crypto core using OpenMLS — MLS key management, SFrame encode/decode, file encryption, KeyPackage generation, local encrypted vault (SQLite/sled on Tauri, IndexedDB on WASM), Web Worker execution for WASM target
- `client-ui`: SolidJS + Panda CSS glassmorphism UI — responsive desktop/tablet/mobile layouts, dark theme, reactive message rendering, Playwright + Midscene E2E testing
- `tauri-shell`: Tauri v2 application shell — Windows, macOS, Linux, iOS, Android, Steam Deck (Flatpak), native push notifications, system tray, auto-update, deep links, audio device management, screen capture, system audio loopback
- `hosting-tiers`: 4-tier hosting model — Tier 1 (SQLite, zero-config `./sombra`), Tier 2 (Docker + PostgreSQL), Tier 3 (Sombra Network managed), Tier 4 (own cloud Terraform/Pulumi), `sombra migrate` dual-pipeline tool, `sombra --tunnel` Cloudflare Quick Tunnel
- `push-notifications`: APNs (iOS) and FCM (Android) push dispatcher — generic payloads only ("New message in [Channel ID]"), triggered when MLS routes to offline user
- `bot-api`: Bots as first-class API citizens — same REST + WebSocket API, API tokens with scoped permissions, E2EE bots (full MLS members) and plaintext bots (stateless), SDKs for TypeScript/Rust/Python
- `data-model`: Repository pattern with Go interfaces — SQLite implementation (WAL mode, single writer) for Tier 1, PostgreSQL implementation (connection pooling, partitioning-ready) for Tier 2+, embedded SQL migrations, ULID primary keys

### Modified Capabilities

_(None — this is a ground-up rebuild with no existing specs to modify.)_

## Impact

- **Codebase**: Entirely new monorepo replacing all Revolt/Stoat-derived code. Structure: `server/` (Go), `crypto/` (Rust), `client/` (SolidJS + Tauri), `sdk/` (TypeScript), `bot-sdk/` (TS/Rust/Python), `deploy/` (Docker/K8s/cloud), `apps/` (community app manifests)
- **Languages**: Reduced from 8 to 3 (Go, TypeScript, Rust) — dramatically reduces agent context overhead
- **Build system**: Turborepo orchestrates across Go, Rust, and TypeScript. No git submodules.
- **Database**: MongoDB dropped entirely. SQLite (embedded) and PostgreSQL (cloud) via repository pattern
- **Dependencies**: NATS JetStream (embedded or external), mediasoup (external Node.js process for SFU), OpenMLS (Rust), ory/fosite (Go OIDC)
- **APIs**: All-new REST API under `/api/v1/` and WebSocket gateway at `/api/v1/gateway` — not backward compatible with Revolt API
- **Clients**: All existing native apps (Kotlin/Swift) and Electron desktop app are replaced
- **License**: AGPL-3.0 → Apache-2.0 with explicit patent grant
- **Deployment**: From 10-15 Docker containers to single binary (Tier 1) or minimal Docker Compose (Tier 2+)
- **Timeline**: ~44-62 weeks across 6 phases, each independently shippable
