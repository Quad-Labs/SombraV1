---
name: sombra-feature
description: Build a Sombra feature using frontend-first TDD workflow (UI → SDK → Server). Enforces strict layer boundaries, [SOMBRA:DEV] logging, and mock data factory usage.
---

# Sombra Feature Development

Build features using the Sombra frontend-first TDD methodology. This is a RIGID skill — follow each step exactly.

## Prerequisites

- Read the relevant OpenSpec capability spec(s) before starting
- Identify which API endpoints, WebSocket events, and SDK methods are needed

## Workflow

### Step 1: UI First

1. Build the SolidJS component in `client/src/`
2. Use Panda CSS glassmorphism design tokens for styling
3. Component MUST only import from `@sombra/sdk` — never from `crypto/`, never call `fetch()` directly
4. Write a Playwright E2E test in `client/e2e/` — it MUST fail (RED) because no backend exists yet
5. Use `client/src/test/mock-factory.ts` for all test data — no inline mocks

### Step 2: SDK + Mocks

1. Implement the SDK method in `sdk/src/` with hardcoded mock responses
2. Wire the SolidJS component to the SDK method
3. Run the Playwright test — it MUST pass against mocks (GREEN)
4. Developer can now interact with the feature visually (mocked data)

### Step 3: Server Implementation

1. **Define the Go interface first** in the relevant `interfaces.go` file — present for approval
2. Write `go test` for the API route (RED)
3. Implement the Go handler in `handlers.go`, service in `service.go`, repository in `repository.go`
4. Add `[SOMBRA:DEV]` logging at every system boundary:
   - Every REST request/response with data shapes and timing
   - Every NATS publish/subscribe
   - Every WebSocket event fan-out
   - Mark all dev logs: `// DEV-LOG: remove before release`
5. `go test` passes (GREEN)

### Step 4: Integration

1. Wire SDK to real server (remove mocks)
2. Run Playwright E2E test against real backend — MUST pass (GREEN)
3. Console output should trace data flow: UI → SDK → Server → NATS → WebSocket → UI

### Step 5: Refactor & Simplify

1. Clean up code while keeping all tests green
2. Run `/simplify` to check for reuse opportunities, code quality, and efficiency
3. Dev logs remain (removed at release)

## Guardrails

- UI layer MUST NOT import from `crypto/` or call `fetch()` directly
- SDK is the ONLY bridge between UI and everything else
- Go handlers call services, services call repositories — no skipping
- All WebRTC state transitions use FSM (no ad-hoc if/else)
- All tests use mock-factory.ts — no inline mock data
- `[SOMBRA:DEV]` logging at every system boundary

## Available Tools

- **`/simplify`** — Run after completing the feature to review quality
- **security-reviewer agent** — Dispatched automatically for auth/crypto/permissions changes
- **code-reviewer agent** — Validates architecture compliance
- **context7 MCP** — Look up docs for OpenMLS, NATS, chi, SolidJS, Tauri, Panda CSS
- **playwright MCP** — Run and debug E2E tests
