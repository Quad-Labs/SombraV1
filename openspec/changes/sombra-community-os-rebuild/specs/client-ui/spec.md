# Capability: client-ui

SolidJS + Panda CSS glassmorphism UI — responsive desktop/tablet/mobile layouts, dark theme, reactive message rendering, Playwright + Midscene E2E testing.

## ADDED Requirements

### Requirement: SolidJS Framework with Fine-Grained Reactivity
The client UI SHALL be built using SolidJS as the sole UI framework. The implementation MUST leverage SolidJS's fine-grained reactivity system (Signals, Effects, Memos) for all state-driven rendering. The UI MUST NOT use a Virtual DOM diffing approach — DOM updates SHALL be surgically applied at the individual node level in response to signal changes.

#### Scenario: Signal change updates only affected DOM node
- **WHEN** a SolidJS Signal value changes (e.g., a user's online status)
- **THEN** only the specific DOM element bound to that Signal is updated — no parent components re-render and no Virtual DOM diff is computed

#### Scenario: No Virtual DOM library present
- **WHEN** the client's dependency tree is inspected
- **THEN** there are no Virtual DOM libraries (React, Preact, or similar) — only SolidJS and its ecosystem packages

---

### Requirement: Panda CSS with Glassmorphism Design Tokens
The client UI SHALL use Panda CSS as its zero-runtime CSS-in-JS styling solution. A glassmorphism design token system MUST be defined covering backdrop-blur values, transparency levels, border radii, glass surface colors, shadow depths, and accent colors. All UI components MUST derive their visual styles from these design tokens.

#### Scenario: Design tokens define glass surface
- **WHEN** a UI panel or card component is rendered
- **THEN** its visual style is derived from Panda CSS glassmorphism design tokens including `backdrop-filter: blur(...)`, semi-transparent background color, and subtle border

#### Scenario: Zero-runtime CSS generation
- **WHEN** the client is built for production
- **THEN** Panda CSS generates static CSS at build time with no runtime JavaScript overhead for style computation

#### Scenario: Token change propagates globally
- **WHEN** a glassmorphism design token value is updated in the Panda CSS configuration (e.g., blur radius)
- **THEN** all components referencing that token reflect the change without per-component style edits

---

### Requirement: Dark Glassmorphism Theme by Default
The client UI SHALL render with a dark glassmorphism theme as the default and initial theme. The dark theme MUST use dark semi-transparent surfaces, light text, and frosted-glass effects. No light theme is required at launch, but the design token architecture MUST support adding one in the future without structural changes.

#### Scenario: First launch renders dark theme
- **WHEN** a user opens the Sombra client for the first time without any theme preference set
- **THEN** the UI renders with the dark glassmorphism theme: dark translucent panels, light foreground text, and backdrop-blur glass effects

#### Scenario: Theme tokens are extensible
- **WHEN** a developer inspects the Panda CSS theme configuration
- **THEN** the token structure supports defining additional theme variants (e.g., a future light theme) by overriding token values without changing component code

---

### Requirement: Responsive Layouts for Desktop, Tablet, and Mobile
The client UI SHALL provide three responsive layout breakpoints: desktop (three-column: server list + channel list + message area), tablet (two-column: collapsible sidebar + message area), and mobile (single-column: stack navigation with slide transitions). Layout transitions MUST be handled by CSS media queries and the SolidJS component tree — not by loading separate component bundles.

#### Scenario: Desktop layout
- **WHEN** the viewport width is at or above the desktop breakpoint (e.g., 1024px and above)
- **THEN** the UI renders a three-column layout with the server list, channel list, and message area all visible simultaneously

#### Scenario: Tablet layout
- **WHEN** the viewport width is between the tablet and desktop breakpoints (e.g., 768px to 1023px)
- **THEN** the UI renders a two-column layout with a collapsible sidebar and the message area as the primary view

#### Scenario: Mobile layout
- **WHEN** the viewport width is below the tablet breakpoint (e.g., below 768px)
- **THEN** the UI renders a single-column stack navigation where only one view (server list, channel list, or message area) is visible at a time, with slide transitions between views

#### Scenario: Same component tree across breakpoints
- **WHEN** the browser window is resized from desktop to mobile width
- **THEN** the layout adapts using CSS media queries and conditional rendering within the same SolidJS component tree — no separate component bundles are loaded

---

### Requirement: Reactive Message Rendering
The UI MUST render incoming messages by inserting a single DOM node per message arrival. The message list MUST NOT re-render its entire component tree when a new message arrives. New messages SHALL be appended to the DOM directly via SolidJS's `<For>` or `<Index>` reactive list primitives, achieving O(1) DOM operations per message.

#### Scenario: New message appends one DOM node
- **WHEN** a `MESSAGE_CREATE` event arrives via the SDK's state store
- **THEN** the UI inserts exactly one new DOM node at the correct position in the message list without re-rendering existing message nodes

#### Scenario: Scroll position maintained on new message
- **WHEN** the user is scrolled up reading message history and a new message arrives
- **THEN** the new message is appended at the bottom of the list without changing the user's scroll position, and a "new messages" indicator appears

#### Scenario: Message update modifies one DOM node
- **WHEN** a `MESSAGE_UPDATE` event arrives for an existing message
- **THEN** only the DOM node for that specific message is updated — no other message nodes are touched

---

### Requirement: UI Layer Isolation from Internals
The client UI layer MUST only interact with `@sombra/sdk` stores and methods for all server communication, state management, and data access. The UI layer MUST NOT import from the `crypto/` directory, call `fetch()` or `XMLHttpRequest` directly, open WebSocket connections directly, or read/write to the local encrypted vault. All such operations MUST go through the SDK.

#### Scenario: UI imports only from @sombra/sdk
- **WHEN** the import statements in `client/src/` component files are inspected
- **THEN** all server communication and state access imports come from `@sombra/sdk` — there are no direct imports from `crypto/`, `node:http`, or browser fetch/WebSocket APIs

#### Scenario: No direct fetch calls in UI code
- **WHEN** the client UI source code is searched for `fetch(`, `XMLHttpRequest`, or `new WebSocket(`
- **THEN** zero matches are found in the UI layer — all network operations are delegated to the SDK

#### Scenario: No direct vault access
- **WHEN** the UI layer code is searched for IndexedDB, SQLite, or local vault access patterns
- **THEN** zero matches are found — the UI accesses cached/stored data exclusively through SDK-provided stores and methods

---

### Requirement: Playwright and Midscene E2E Testing
The client UI SHALL use Playwright with Midscene as the primary E2E testing framework and the primary confidence layer for the project. E2E tests MUST cover critical user flows including registration, login, server creation, channel navigation, message sending/receiving, and real-time updates. Tests MUST be executable in CI and produce visual reports.

#### Scenario: E2E test covers message send and receive
- **WHEN** the Playwright test suite runs the message flow test
- **THEN** the test registers a user, creates a server and channel, sends a message, and verifies the message appears in the channel message list

#### Scenario: Midscene provides visual verification
- **WHEN** a Playwright test includes a Midscene visual checkpoint
- **THEN** Midscene captures the current UI state and verifies visual elements (layout, text content, component visibility) against expected conditions

#### Scenario: E2E tests run in CI
- **WHEN** a pull request is submitted to the repository
- **THEN** the Playwright E2E test suite executes in the CI pipeline and the PR cannot merge if any E2E test fails

---

### Requirement: Mock Data Factory
The client SHALL include a mock data factory at `client/src/test/mock-factory.ts` that generates deterministic test data for users, servers, channels, messages, and other domain objects. The factory MUST produce valid data shapes matching the SDK's type definitions and MUST support overriding individual fields for test-specific scenarios.

#### Scenario: Generate a mock user
- **WHEN** a test calls the mock factory to create a user (e.g., `mockFactory.user()`)
- **THEN** the factory returns a fully populated user object with deterministic default values matching the SDK's User type

#### Scenario: Override specific fields
- **WHEN** a test calls the mock factory with field overrides (e.g., `mockFactory.user({ username: "testbot" })`)
- **THEN** the factory returns a user object with the overridden username and deterministic defaults for all other fields

#### Scenario: Deterministic output for reproducible tests
- **WHEN** the mock factory is called with the same parameters across multiple test runs
- **THEN** it produces identical output, ensuring test assertions are reproducible

---

### Requirement: Web Client Without Tauri
The SolidJS web client MUST function as a standalone web application without the Tauri shell. When running in a standard web browser (no Tauri runtime detected), the client SHALL use the WASM-compiled crypto core in a Web Worker for all cryptographic operations and IndexedDB for local storage. All core features — registration, login, messaging, E2EE — MUST work in browser-only mode.

#### Scenario: Browser-only mode detection
- **WHEN** the client starts in a standard web browser without Tauri
- **THEN** the SDK detects the absence of the Tauri runtime and initializes the WASM crypto core in a Web Worker instead of native FFI

#### Scenario: Full functionality without Tauri
- **WHEN** a user accesses Sombra via a web browser
- **THEN** they can register, log in, join servers, send and receive E2EE messages, and participate in channels with full functionality

#### Scenario: IndexedDB used for local storage
- **WHEN** the client is running in browser-only mode
- **THEN** the message cache, outbox queue, and crypto vault all use IndexedDB for persistence
