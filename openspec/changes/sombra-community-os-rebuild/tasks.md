## 1. Project Scaffolding

- [ ] 1.1 Initialize monorepo structure: `server/`, `crypto/`, `client/`, `sdk/`, `bot-sdk/`, `deploy/`, `apps/`
- [ ] 1.2 Set up `server/go.mod` with Go module and initial dependencies (chi, nats.go, sqlc/sqlx, ulid)
- [ ] 1.3 Set up `crypto/Cargo.toml` with OpenMLS, SFrame, getrandom (js feature for WASM target)
- [ ] 1.4 Set up `client/` with SolidJS + Vite + Panda CSS + Tauri v2 scaffolding
- [ ] 1.5 Set up `sdk/` with TypeScript package scaffolding (`@sombra/sdk`)
- [ ] 1.6 Configure Turborepo (`turbo.json`) for cross-language build orchestration
- [ ] 1.7 Create `deploy/docker/Dockerfile` for Go server and `docker-compose.yml` templates
- [ ] 1.8 Add Apache-2.0 LICENSE file
- [ ] 1.9 Create `client/src/test/mock-factory.ts` with deterministic mock data generators (users, servers, channels, messages, ULIDs, KeyPackages)

## 2. Phase 1A: Server Core & Data Layer

- [ ] 2.1 Implement `server/cmd/sombra/main.go` entry point with zero-config startup (create sombra.db, sombra-data/, bind 0.0.0.0:8080)
- [ ] 2.2 Set up chi router with `/api/v1/` prefix and middleware chain (Recoverer → RequestID → DevLogger → RateLimiter → Authenticator → RBAC)
- [ ] 2.3 Implement `[SOMBRA:DEV]` structured logging middleware controlled by `SOMBRA_DEV_LOG` env var
- [ ] 2.4 Implement embedded NATS JetStream initialization (in-process for Tier 1, external via `NATS_URL` for Tier 2+)
- [ ] 2.5 Implement NATS KV store for ephemeral state (presence, typing, rate limits)
- [ ] 2.6 Implement health check endpoint (`GET /api/v1/health`)
- [ ] 2.7 Implement graceful shutdown with signal handling
- [ ] 2.8 Define all repository interfaces in `server/internal/db/interfaces.go` (UserRepo, ServerRepo, ChannelRepo, MessageRepo, MemberRepo, RoleRepo, InviteRepo, BotRepo, KeyPackageRepo, MLSGroupRepo, BlobMetaRepo, AppServiceRepo)
- [ ] 2.9 Create SQL migration files for core entities (users, devices, sessions, servers, channels, messages, members, roles, invites, mls_groups, key_packages)
- [ ] 2.10 Implement SQLite repository (WAL mode, single writer + reader pool) with auto-migration on startup
- [ ] 2.11 Implement PostgreSQL repository with connection pooling
- [ ] 2.12 Add `DB_TYPE` env var switching between SQLite and PostgreSQL implementations
- [ ] 2.13 Write `go test` unit tests for both repository implementations against embedded SQLite

## 3. Phase 1B: Authentication & Permissions

- [ ] 3.1 Implement user registration endpoint (`POST /api/v1/auth/register`) with username, email, password validation
- [ ] 3.2 Implement login endpoint (`POST /api/v1/auth/login`) returning session token
- [ ] 3.3 Implement logout endpoint (`POST /api/v1/auth/logout`)
- [ ] 3.4 Implement session token validation middleware (Authenticator in middleware chain)
- [ ] 3.5 Implement multi-device session management (list devices, revoke individual session, revoke all sessions)
- [ ] 3.6 Implement current user profile endpoints (`GET|PATCH /api/v1/users/@me`, `GET /api/v1/users/:id`)
- [ ] 3.7 Implement RBAC engine: roles as permission sets, channel-level overrides with deny-takes-precedence
- [ ] 3.8 Implement RBAC middleware (permission evaluation on every API call)
- [ ] 3.9 Implement default roles: @everyone (base) and server owner (full permissions)
- [ ] 3.10 Implement role CRUD endpoints (`POST|GET|DELETE /api/v1/servers/:id/roles`)
- [ ] 3.11 Write `go test` unit tests for auth and RBAC

## 4. Phase 1C: Servers, Channels, Messages (REST)

- [ ] 4.1 Implement server CRUD endpoints (`POST /api/v1/servers`, `GET|PATCH|DELETE /api/v1/servers/:id`)
- [ ] 4.2 Implement channel CRUD endpoints (`POST /api/v1/servers/:id/channels`, `GET|PATCH|DELETE /api/v1/channels/:id`)
- [ ] 4.3 Implement member management (`POST /api/v1/servers/:id/members`, `GET|DELETE /api/v1/servers/:id/members/:id`)
- [ ] 4.4 Implement invite system (`POST /api/v1/servers/:id/invites`, `POST /api/v1/invites/:code/accept`)
- [ ] 4.5 Implement message creation (`POST /api/v1/channels/:id/messages`) storing opaque ciphertext blobs with ULID IDs
- [ ] 4.6 Implement message listing with cursor-based pagination (`GET /api/v1/channels/:id/messages?before=&after=&limit=`)
- [ ] 4.7 Implement message editing and deletion (`PATCH|DELETE /api/v1/channels/:id/messages/:id`)
- [ ] 4.8 Implement rate limiting via NATS KV (per-IP, per-user sliding window, 429 with Retry-After)
- [ ] 4.9 Write `go test` integration tests for full API flows

## 5. Phase 1D: WebSocket Gateway

- [ ] 5.1 Implement WebSocket endpoint at `wss://host/api/v1/gateway`
- [ ] 5.2 Implement HELLO → IDENTIFY → READY handshake with initial state payload
- [ ] 5.3 Implement RESUME with session ID + sequence number (60-second event buffer)
- [ ] 5.4 Implement heartbeat mechanism with configurable interval and timeout detection
- [ ] 5.5 Implement event envelope format (`{ "op", "t", "s", "d" }`)
- [ ] 5.6 Implement server-to-client event fan-out via NATS subscriptions (MESSAGE_CREATE, MESSAGE_UPDATE, MESSAGE_DELETE, CHANNEL_*, MEMBER_*)
- [ ] 5.7 Implement RBAC-filtered event fan-out (only send events to users with channel access)
- [ ] 5.8 Write integration tests for WebSocket lifecycle and event delivery

## 6. Phase 1E: MLS E2EE Delivery Service

- [ ] 6.1 Implement KeyPackage upload endpoint (`POST /api/v1/users/@me/keys/packages`) for batch upload (100 at a time)
- [ ] 6.2 Implement KeyPackage fetch endpoint (`GET /api/v1/users/:id/keys/packages`) with one-time consumption
- [ ] 6.3 Implement KEY_PACKAGES_LOW WebSocket event when device has <10 remaining
- [ ] 6.4 Implement MLS Commit endpoint (`POST /api/v1/channels/:id/mls/commit`) with epoch monotonicity validation
- [ ] 6.5 Implement MLS Proposal endpoint (`POST /api/v1/channels/:id/mls/proposal`)
- [ ] 6.6 Implement MLS Welcome endpoint (`POST /api/v1/channels/:id/mls/welcome`) with 7-day storage
- [ ] 6.7 Implement desync recovery: Commit buffer (last 100 per group, 24-hour retention) and rejoin-request endpoint
- [ ] 6.8 Implement one MLSGroup per channel mapping with epoch state tracking
- [ ] 6.9 Implement MLS WebSocket events (MLS_COMMIT, MLS_WELCOME, MLS_PROPOSAL) fan-out
- [ ] 6.10 Write `go test` tests for MLS delivery service

## 7. Phase 1F: Rust Crypto Core

- [ ] 7.1 Implement OpenMLS wrapper in `crypto/src/mls.rs` (group create, add member, remove member, encrypt, decrypt)
- [ ] 7.2 Implement SFrame encode/decode in `crypto/src/sframe.rs`
- [ ] 7.3 Implement StorageProvider trait in `crypto/src/storage.rs`
- [ ] 7.4 Implement native StorageProvider (SQLite/sled) in `crypto/src/storage_native.rs`
- [ ] 7.5 Implement WASM StorageProvider (IndexedDB) in `crypto/src/storage_web.rs`
- [ ] 7.6 Implement streaming file encryption (5MB chunk processing)
- [ ] 7.7 Implement KeyPackage batch generation (100 at a time)
- [ ] 7.8 Configure dual build targets: native lib (Tauri) and WASM (web browser)
- [ ] 7.9 Implement Web Worker glue in `crypto/wasm/` for WASM execution
- [ ] 7.10 Verify `getrandom = { features = ["js"] }` for WASM RNG
- [ ] 7.11 Write `cargo test` unit tests for MLS wrapper, SFrame, and file encryption

## 8. Phase 1G: Client SDK & Basic UI

- [ ] 8.1 Implement SDK REST client for auth endpoints (register, login, logout)
- [ ] 8.2 Implement SDK REST client for server/channel/message CRUD
- [ ] 8.3 Implement SDK WebSocket manager with HELLO/IDENTIFY/READY/RESUME lifecycle
- [ ] 8.4 Implement SDK reactive state store (users, servers, channels, messages, presence)
- [ ] 8.5 Implement SDK crypto bridge — async message passing to Rust core (Tauri IPC or Web Worker)
- [ ] 8.6 Implement SDK MLS session management (encrypt/decrypt messages via crypto bridge)
- [ ] 8.7 Implement SDK offline outbox queue (IndexedDB on web, SQLite on Tauri)
- [ ] 8.8 Build SolidJS login/register page with glassmorphism styling
- [ ] 8.9 Build SolidJS server list and channel list components
- [ ] 8.10 Build SolidJS message list with reactive rendering (one DOM node per message)
- [ ] 8.11 Build SolidJS message input with E2EE send flow (encrypt → POST → optimistic render)
- [ ] 8.12 Implement channel history modes UI (full/from-join/ephemeral/trusted-server selector)
- [ ] 8.13 Write Playwright E2E tests: registration → login → create server → send/receive encrypted message
- [ ] 8.14 Configure `./sombra` to serve the built web client directly (zero-config Tier 1)

## 9. Phase 2: Real-Time & Presence

- [ ] 9.1 Implement presence system: PUT /api/v1/users/@me/presence with NATS KV + TTL
- [ ] 9.2 Implement typing indicators: POST /api/v1/channels/:id/typing with NATS KV
- [ ] 9.3 Implement PRESENCE_UPDATE and TYPING_START WebSocket event fan-out
- [ ] 9.4 Implement rate limiting on presence and typing updates
- [ ] 9.5 Implement reactions (encrypted in MLS payloads)
- [ ] 9.6 Implement threads as child channels
- [ ] 9.7 Implement message pins per channel
- [ ] 9.8 Implement basic moderation: kick, ban, timeout
- [ ] 9.9 Implement audit log for server events
- [ ] 9.10 Implement push dispatcher: APNs + FCM with generic payloads only
- [ ] 9.11 Implement blob storage: filesystem StorageProvider for Tier 1 (sombra-data/)
- [ ] 9.12 Implement blob storage: S3-compatible StorageProvider for Tier 2+
- [ ] 9.13 Implement streaming file encryption/upload (5MB chunks, EncryptStream interface)
- [ ] 9.14 Implement file attachment endpoints (POST /api/v1/blob/upload, GET /api/v1/blob/:id)
- [ ] 9.15 Build SDK methods for presence, typing, reactions, threads, file attachments
- [ ] 9.16 Build SolidJS UI components for presence indicators, typing dots, reactions, threads, file upload
- [ ] 9.17 Write Playwright E2E tests for presence, typing, reactions, threads, and file sharing

## 10. Dirty Network Alpha

- [ ] 10.1 Deploy to VPS with real-world network conditions
- [ ] 10.2 Battle-test MLS desync recovery under packet loss and reconnection
- [ ] 10.3 Test encrypted vault corruption handling and recovery
- [ ] 10.4 Verify NATS KV TTL behavior under load
- [ ] 10.5 Test push notification reliability (APNs + FCM)
- [ ] 10.6 Fix all issues discovered during alpha testing

## 11. Phase 3A: Voice Calling

- [ ] 11.1 Implement WebRTC signaling handler in Go (SDP offer/answer, ICE candidate exchange)
- [ ] 11.2 Implement call state FSM: Idle → Connecting → ICENegotiating → Connected → ICERestarting → Connected → Disconnecting → Idle
- [ ] 11.3 Implement P2P WebRTC for 1:1 calls with TURN fallback
- [ ] 11.4 Implement SFrame integration: derive symmetric key from MLS via MLS_Exporter with label "sframe_key"
- [ ] 11.5 Implement SFrame epoch tagging and 10-second grace period for key rotation
- [ ] 11.6 Implement ICE restart handling for mobile roaming (Wi-Fi ↔ cellular)
- [ ] 11.7 Implement mediasoup SFU integration: JSON-RPC communication over stdin/stdout
- [ ] 11.8 Implement tiered media routing: P2P (1:1) → mesh (2-4) → SFU (5+)
- [ ] 11.9 Implement TURN relay configuration (self-hosted, Sombra relay, Cloudflare TURN)
- [ ] 11.10 Implement VOICE_STATE_UPDATE, CALL_CREATE, CALL_END WebSocket events
- [ ] 11.11 Implement call state FSM in TypeScript SDK (matching Go server FSM)
- [ ] 11.12 Build SolidJS voice call UI (call controls, participant list, connection indicators)
- [ ] 11.13 Write Playwright E2E tests for 1:1 voice call lifecycle

## 12. Phase 3B: Live Streaming

- [ ] 12.1 Implement Go Live signaling: screen/window/camera capture → SFU
- [ ] 12.2 Implement VP9 SVC configuration (spatial + temporal layers)
- [ ] 12.3 Implement WHIP endpoint for OBS/external ingest with stream key auth
- [ ] 12.4 Implement system audio capture via Tauri/cpal native loopback
- [ ] 12.5 Implement multi-source streaming (camera + screen as separate producers)
- [ ] 12.6 Build SolidJS viewer UI: PiP, side-by-side layouts, quality selector
- [ ] 12.7 Build SolidJS streamer UI: source selection, stream controls
- [ ] 12.8 Write Playwright E2E tests for Go Live streaming lifecycle

## 13. Phase 4: Multi-Platform & Polish

- [ ] 13.1 Implement Panda CSS glassmorphism design tokens (blur, transparency, dark theme variables)
- [ ] 13.2 Implement responsive layouts: desktop ↔ tablet ↔ mobile breakpoints
- [ ] 13.3 Configure Tauri v2 desktop builds: Windows (.msi), macOS (.dmg), Linux (.AppImage/.deb/.flatpak)
- [ ] 13.4 Configure Tauri v2 mobile builds: iOS (.ipa), Android (.apk/.aab)
- [ ] 13.5 Configure Steam Deck packaging (Flatpak on WebKitGTK)
- [ ] 13.6 Implement WebKitGTK capability detection with graceful fallback (VP8 fallback, audio-only if SFrame unsupported)
- [ ] 13.7 Implement system tray integration with unread badge
- [ ] 13.8 Implement auto-update mechanism
- [ ] 13.9 Implement deep link handling (sombra:// protocol)
- [ ] 13.10 Implement notification preferences UI
- [ ] 13.11 Build user profile and custom status UI
- [ ] 13.12 Build emoji picker and markdown rendering
- [ ] 13.13 Implement local encrypted vault client-side search index
- [ ] 13.14 Write Playwright E2E tests for responsive layouts and multi-platform features

## 14. Phase 5: Community & Federation

- [ ] 14.1 Implement forum channel type (text channel with thread-first UX)
- [ ] 14.2 Implement ActivityPub: WebFinger discovery for forum actors (Group type)
- [ ] 14.3 Implement ActivityPub: HTTP Signatures on all outbound deliveries (forum RSA keypair)
- [ ] 14.4 Implement ActivityPub: inbox processing (Follow, Create, Like, Delete, Update, Lock)
- [ ] 14.5 Implement ActivityPub: outbound activity delivery via transactional outbox with exponential backoff
- [ ] 14.6 Implement ActivityPub: Lemmy-compatible JSON-LD context and extensions
- [ ] 14.7 Implement ActivityPub: content sanitization (strict allowlist HTML sanitizer via bluemonday)
- [ ] 14.8 Implement public blob bucket for federated media
- [ ] 14.9 Implement per-forum `federation_enabled` toggle
- [ ] 14.10 Implement embedded OIDC provider using ory/fosite (discovery endpoint, token introspection, refresh)
- [ ] 14.11 Implement "Log in with Sombra" flow for community apps
- [ ] 14.12 Implement bot API: POST /api/v1/servers/:id/bots with scoped API tokens
- [ ] 14.13 Implement E2EE bot support (full MLS group membership)
- [ ] 14.14 Implement plaintext bot support (stateless API wrappers)
- [ ] 14.15 Publish @sombra/sdk and @sombra/bot-sdk packages
- [ ] 14.16 Implement "trusted server" mode (opt-in plaintext channels with UI warning)
- [ ] 14.17 Test Lemmy federation interop with live Lemmy instances
- [ ] 14.18 Write Playwright E2E tests for federation, OIDC, and bot API

## 15. Phase 6: Community OS

- [ ] 15.1 Implement DockerManager (ServiceManager interface + Docker SDK)
- [ ] 15.2 Implement internal reverse proxy routing `/app/{name}/*` to container IPs
- [ ] 15.3 Implement OIDC auth check at proxy layer
- [ ] 15.4 Implement SSRF prevention: isolated `app_net` Docker network, whitelist per app manifest
- [ ] 15.5 Implement app catalog: manifest.json parsing (image, ports, volumes, OIDC config)
- [ ] 15.6 Create curated app manifests: Plex, Jellyfin, Nextcloud, Gitea, Minecraft
- [ ] 15.7 Implement GET|POST|DELETE /api/v1/servers/:id/apps endpoints
- [ ] 15.8 Implement resource isolation (container CPU/memory limits)
- [ ] 15.9 Build SolidJS community app management UI (install, configure, remove)
- [ ] 15.10 Implement KubernetesManager (ServiceManager interface + K8s client-go)
- [ ] 15.11 Create Helm charts for Tier 3 Sombra Network deployment
- [ ] 15.12 Implement `sombra migrate` dual-pipeline tool (SQLite→Postgres + filesystem→S3, checkpoint-based)
- [ ] 15.13 Implement `sombra --tunnel` Cloudflare Quick Tunnel integration
- [ ] 15.14 Create Terraform/Pulumi modules for Tier 4 cloud deployment
- [ ] 15.15 Implement billing integration (Stripe, compute/storage metering) for Sombra Network
- [ ] 15.16 Create pre-built Docker Compose templates (chat-only, chat+voice, chat+voice+streaming, full stack)
- [ ] 15.17 Implement account recovery key generation (BIP-39 mnemonic) at signup
- [ ] 15.18 Implement device revocation flow (invalidate tokens → delete KeyPackages → MLS Remove proposals)
- [ ] 15.19 Write Playwright E2E tests for community app management and tier migration
