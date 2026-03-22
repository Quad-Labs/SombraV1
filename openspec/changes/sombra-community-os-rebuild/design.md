## Context

Sombra is being rebuilt from scratch to replace a Revolt/Stoat-derived codebase that has grown unsustainable: 19 Rust crates, 37+ git submodules, 8 programming languages, 15 Docker containers for local development, and 30-60 second incremental rebuild times under AGPL-3.0 licensing. The new architecture targets a 1-3 person team using agentic AI-assisted development (Claude Code) as a primary workflow, optimizing for fast iteration, minimal configuration, and cross-platform delivery from a single codebase.

The system is a monorepo with three languages: Go (server), TypeScript (frontend/SDK), and Rust (crypto core). It follows a frontend-first development methodology where features are built UI → SDK → Server, with Playwright E2E tests as the primary confidence layer.

## Goals / Non-Goals

**Goals:**

- Single Go binary that starts in <2 seconds with zero configuration (Tier 1 "Minecraft Mode")
- MLS (RFC 9420) E2EE for all text/file communication — server routes ciphertext it cannot decrypt
- SFrame (RFC 9605) E2EE for all voice/video streams
- 6-platform delivery (Windows, macOS, Linux, iOS, Android, Steam Deck) from one SolidJS + Tauri v2 codebase
- ActivityPub federation for forum channels with Lemmy ecosystem interop
- Community app orchestration (Plex, Nextcloud, etc.) via Docker/K8s with OIDC SSO
- Apache-2.0 licensing with revenue from services (hosting, SFU relay, TURN fleet), not source restrictions
- Agentic development optimized: 3 languages, no submodules, heavy structured logging, fast builds

**Non-Goals:**

- Full federation for real-time chat (E2EE and cross-instance chat are architecturally incompatible)
- Server-side message search (messages are opaque ciphertext; search is client-side)
- Backward compatibility with Revolt/Stoat API (clean break, new API surface)
- Custom SFU implementation (mediasoup is proven; Pion/Go-native SFU is a Phase 3 evaluation only)
- Enterprise edition with gated features (same binary everywhere; revenue is services)
- Zero-knowledge metadata (server must know membership and routing; "we can't read your conversations" is the precise, honest claim)

## Decisions

### D1: Go for the backend, Rust only for client crypto

**Choice:** Single Go binary server with Rust crypto core compiled to native (Tauri) and WASM (web).

**Alternatives considered:**
- Full Rust backend (current Stoat): 30-60s incremental rebuilds, complex async/await + tokio concurrency, steep learning curve. The server routes E2EE ciphertext — it doesn't do heavy computation where Rust's zero-cost abstractions matter.
- Node.js/Deno: GC pauses acceptable but no single-binary deployment, larger memory footprint, weaker concurrency model for WebSocket management at scale.

**Rationale:** Go's goroutines are ideal for I/O-bound WebSocket management. <2s builds enable rapid agentic iteration. Cross-compilation is trivial (`GOARCH=arm64 go build`). CPU-intensive crypto stays in Rust where it matters — in the client.

### D2: SQLite (Tier 1) + PostgreSQL (Tier 2+) via Repository Pattern

**Choice:** Go interfaces for all data access. SQLite implementation for zero-config local hosting, PostgreSQL for cloud deployments. Switched by `DB_TYPE` environment variable.

**Alternatives considered:**
- MongoDB (current): Required external service even for local dev. No embedded option. AGPL licensing concerns.
- SQLite only: Cannot scale horizontally for Tier 3 managed hosting.
- PostgreSQL only: Requires running a separate database even for "Minecraft Mode" — violates zero-config goal.

**Rationale:** Repository pattern with dual backends gives zero-config simplicity for self-hosters while supporting cloud scale. SQLite WAL mode with single-writer connection handles concurrent goroutine access without "database is locked" errors.

### D3: NATS JetStream as the event bus

**Choice:** Embedded NATS for Tier 1 (in-process, no separate service), external NATS for Tier 2+ (horizontal scaling). NATS KV for all ephemeral state (presence, typing, rate limits).

**Alternatives considered:**
- Redis: Requires separate service even for Tier 1. NATS embeds cleanly.
- In-memory channels only: No persistence, no replay on reconnect, no scaling path.

**Rationale:** NATS embeds directly into the Go binary for single-process deployment, then scales to external cluster for multi-node. KV store replaces Redis for ephemeral state. JetStream provides event replay for WebSocket reconnection buffering.

### D4: SolidJS + Panda CSS for the frontend

**Choice:** Keep SolidJS (from existing codebase), add Panda CSS for zero-runtime CSS-in-JS with glassmorphism design tokens.

**Alternatives considered:**
- React: 4x slower partial updates, 6x larger bundle. Struggles with high-frequency chat message rendering.
- Svelte: Comparable performance but team already knows SolidJS. Zero ramp-up.

**Rationale:** Fine-grained reactivity means inserting one DOM node per message instead of re-rendering component trees. 7KB gzipped. Team familiarity eliminates migration cost.

### D5: Tauri v2 for desktop and mobile

**Choice:** Tauri v2 for all native platforms (Windows, macOS, Linux, iOS, Android, Steam Deck via Flatpak).

**Alternatives considered:**
- Electron (current): ~300MB RAM, ~150MB bundle, no mobile, bundles Chromium.
- React Native: Different framework, different paradigm. No desktop. Can't share code with web.
- Capacitor/Ionic: WebView-based but limited native API access compared to Tauri's Rust plugins.

**Rationale:** ~30MB RAM, ~5-15MB bundle, 6 platforms from 1 codebase, native Rust access for crypto core (no WASM overhead on desktop/mobile).

### D6: MLS for E2EE, SFrame for media

**Choice:** MLS (RFC 9420) via OpenMLS for text/file encryption. SFrame (RFC 9605) for voice/video frame-level encryption. MLS is single source of truth for all keys.

**Alternatives considered:**
- Signal Protocol (Double Ratchet): Designed for 1:1 conversations. Group chat requires fan-out O(N) messages. MLS provides O(log N) via ratchet trees.
- Custom protocol: Unauditable. RFC standards provide security guarantees.

**Rationale:** MLS handles group key agreement efficiently. SFrame keys are deterministically derived from MLS epoch secrets via `MLS_Exporter` — no manual key distribution needed. Forward secrecy on member join/leave is automatic.

### D7: mediasoup as external SFU process

**Choice:** mediasoup (Node.js + C++ workers) as an external process, communicating with Go server via JSON-RPC over stdin/stdout (bare metal) or HTTP/gRPC (Docker/cloud).

**Alternatives considered:**
- Pion (Go-native SFU): Eliminates Node.js dependency but less battle-tested for SVC/simulcast. Evaluate in Phase 3.
- LiveKit: Full platform, not embeddable. Overkill for Sombra's architecture.
- Janus: C-based, harder to deploy and maintain.

**Rationale:** mediasoup is proven for VP9 SVC, Insertable Streams, and large-scale routing. Node.js process is isolated from Go core — a streaming spike cannot crash text. The Go server holds all room state; mediasoup workers are stateless.

### D8: Frontend-first TDD development methodology

**Choice:** Every feature built UI → SDK → Server. Playwright E2E tests written first (RED), then SDK mocks make them pass (GREEN), then Go backend replaces mocks, then integration verification.

**Rationale:** Visual feedback before any backend exists. Playwright/Midscene give Claude Code browser-based test results. API contracts are defined from the consumer's perspective. The developer sees and interacts with features immediately.

### D9: Console-driven observability with [SOMBRA:DEV] logging

**Choice:** Heavy structured dev logging at every system boundary with the `[SOMBRA:DEV]` prefix. Log data shapes (not content), timing, and decision points. Controlled by `SOMBRA_DEV_LOG=true`. All dev log blocks marked `// DEV-LOG: remove before release`.

**Rationale:** When a test fails, the agent reads console output and sees exactly where in the data flow things went wrong. The agent never has to guess what the server did — it's all in the logs.

### D10: Monorepo with Turborepo, no submodules

**Choice:** Single repository with `server/`, `crypto/`, `client/`, `sdk/`, `bot-sdk/`, `deploy/`, `apps/` directories. Turborepo orchestrates builds across Go, Rust, and TypeScript.

**Alternatives considered:**
- Polyrepo: Coordination overhead between repos. Agent can't update API + SDK + UI in one PR.
- Git submodules (current Revolt): 37+ submodules are the root cause of current complexity.

**Rationale:** One PR can touch API handler, SDK types, and UI component simultaneously. No submodule synchronization. Agent context stays focused on one repo.

## Risks / Trade-offs

- **[Risk] WebKitGTK WebRTC support varies on Steam Deck** → Mitigation: Phase 4 includes WebKitGTK capability detection with graceful fallback (VP8 instead of VP9, audio-only if SFrame unsupported)

- **[Risk] COOP/COEP headers required for multi-threaded WASM restrict cross-origin resources** → Mitigation: Start with single-threaded WASM; only add SharedArrayBuffer if crypto performance demands it. If needed, federated media embeds and CDN assets must include CORP headers.

- **[Risk] mediasoup adds a 4th runtime (Node.js) for SFU** → Mitigation: SFU is optional (P2P-only on Tier 1), isolated to a separate process/container, never touches core Go binary. Evaluate Pion (Go-native) in Phase 3 if Node.js proves problematic.

- **[Risk] SQLite single-writer limits write throughput for large communities on Tier 1** → Mitigation: Tier 1 targets small communities (friend groups, small guilds). `sombra migrate` provides a checkpoint-based upgrade path to PostgreSQL when scaling is needed.

- **[Risk] MLS ratchet tree updates for large groups (1000+ members) may cause latency** → Mitigation: O(log N) cost is bounded. WASM crypto runs in Web Worker to prevent UI thread blocking. KeyPackage batching (100 at a time) amortizes upload cost.

- **[Risk] Full rebuild timeline (~44-62 weeks) for 1-3 person team** → Mitigation: Each phase is independently shippable. Launch publicly after Phase 2 (text chat + E2EE) and iterate in the open. Agentic development accelerates implementation.

- **[Trade-off] Server CAN see metadata (who, when, where)** → Accepted: Server must know membership and routing to function. "We cannot read your conversations" is the precise, verifiable, honest claim. Zero-knowledge metadata would require onion routing, which is a fundamentally different architecture.

- **[Trade-off] Federated forum content is plaintext** → Accepted: E2EE and federation are mutually exclusive by design. Forum posts that federate to Lemmy must be readable by remote instances. Per-forum toggle lets server owners choose.

- **[Trade-off] Client-side search only for E2EE channels** → Accepted: Server cannot index ciphertext. Local encrypted vault maintains a client-side search index. "Trusted server" mode exists as opt-in escape hatch for self-hosted instances that want server-side search.
