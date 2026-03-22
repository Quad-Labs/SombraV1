# Sombra — Claude Code Configuration

## Project Overview

Sombra is a privacy-first Community Operating System. Single Go binary backend, SolidJS + Tauri v2 client, Rust crypto core (OpenMLS/SFrame). E2EE by default, self-hostable, Apache-2.0.

## Monorepo Structure

```
server/          # Go backend (single binary, chi router)
crypto/          # Rust crypto core (OpenMLS, SFrame, WASM)
client/          # SolidJS + Panda CSS + Tauri v2
sdk/             # @sombra/sdk (TypeScript)
bot-sdk/         # Bot SDKs (TS/Rust/Python)
deploy/          # Docker/K8s/Terraform configs
apps/            # Community app manifests
openspec/        # Specifications (source of truth)
```

## Architecture Guardrails

### Layer Boundaries (STRICT)

- **SolidJS UI** MUST only interact with `@sombra/sdk` stores and methods
- **UI MUST NOT** import from `crypto/`, call `fetch()` directly, or manipulate the local encrypted vault
- **SDK** is the only bridge between UI and everything else
- **SDK** calls the Rust crypto core — never touches raw keys directly

### Go Server Layering (STRICT)

- **Handlers** → **Services** → **Repositories** — no skipping layers
- No handler should directly query the database
- Each Go file does one thing: `handlers.go` (HTTP), `service.go` (business logic), `repository.go` (data access)
- Interface-first: define Go interfaces in `interfaces.go` before writing implementations
- Use `/sombra-interface` skill when creating new subsystems

### Middleware Chain (Fixed Order)

Recoverer → RequestID → DevLogger → RateLimiter → Authenticator → RBAC

### WebRTC State Machine

All call state transitions (Go + TypeScript) MUST use explicit FSM (enum states + transition table). No ad-hoc if/else chains.

## Development Methodology

### Frontend-First TDD

Every feature follows: **UI → SDK → Server** (not backend-first).

1. Build SolidJS component
2. Write Playwright E2E test (RED)
3. Build SDK method with mocks (GREEN)
4. Build Go backend (make test pass)
5. Wire SDK to real API (remove mocks)
6. Verify full-stack E2E

Use `/sombra-feature` skill to enforce this workflow.

### Dev Logging

- All dev logs use `[SOMBRA:DEV]` prefix — greppable, removable
- Log at system boundaries: REST request/response, WebSocket events, SDK calls, crypto ops, NATS pub/sub
- Log data shapes, NOT data content (`ciphertext: [1024 bytes]`, not actual ciphertext)
- Log timing on every operation
- Controlled by `SOMBRA_DEV_LOG=true`
- Every dev log block: `// DEV-LOG: remove before release`

### Mock Data Factory

All E2E and SDK tests use `client/src/test/mock-factory.ts` for deterministic mock data. No inline mock data in test files.

## Available Tools & Skills

### Skills (invoke with /skill-name)

| Skill | When to Use |
|-------|-------------|
| `/sombra-feature` | Building any new feature (enforces frontend-first TDD) |
| `/sombra-interface` | Creating new Go subsystem interfaces |
| `/simplify` | After completing a feature — reviews for reuse, quality, efficiency |
| `/opsx:propose` | Starting a new OpenSpec change |
| `/opsx:apply` | Implementing tasks from an OpenSpec change |
| `/opsx:archive` | Finalizing a completed change |

### Hooks (automatic)

- **go fmt**: Auto-formats Go files after every edit
- **Sensitive file protection**: Blocks edits to .env, credentials, secrets, generated WASM
- **Simplify reminder**: After completing task groups, reminds to run `/simplify`

### Subagents (dispatched automatically when relevant)

- **security-reviewer**: Reviews auth, crypto, permissions, federation, SSRF changes
- **code-reviewer**: Validates architecture compliance and layer boundaries

### MCP Servers

- **context7**: Live documentation for OpenMLS, NATS, chi, SolidJS, Tauri, Panda CSS, mediasoup, fosite
- **playwright**: Browser automation for E2E testing

## Commands

```bash
# Start server (Tier 1 zero-config)
./sombra

# Run Go tests
go test ./server/internal/...

# Run Rust crypto tests
cargo test --manifest-path crypto/Cargo.toml

# Run E2E tests
npx playwright test

# Validate OpenSpec
openspec validate
```

## Key Decisions Reference

- Messages are opaque ciphertext — server cannot read content
- ULIDs for message IDs (time-ordered, globally unique)
- NATS KV for all ephemeral state (presence, typing, rate limits)
- One MLSGroup per channel
- SFrame keys derived from MLS via MLS_Exporter (never manually distributed)
- Federated forum content is plaintext (E2EE and federation are mutually exclusive per channel)
- SQLite (Tier 1) / PostgreSQL (Tier 2+) via repository pattern
