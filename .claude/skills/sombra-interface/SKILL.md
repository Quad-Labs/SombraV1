---
name: sombra-interface
description: Define a Go subsystem interface in interfaces.go before writing implementations. Enforces interface-first development and handlers → services → repositories layering.
---

# Sombra Interface Definition

Define Go interfaces before implementations. This is a RIGID skill — present the interface for approval before writing any implementation code.

## Workflow

### Step 1: Identify the Subsystem

Read the relevant OpenSpec capability spec to understand:
- What operations this subsystem needs
- What data it consumes and produces
- What other subsystems it depends on

### Step 2: Define the Interface

Create or update the interface in the appropriate `interfaces.go` file:

```
server/internal/<subsystem>/interfaces.go
```

Rules:
- One interface per concern (e.g., `MLSGroupRepo` for MLS data access, `MLSService` for MLS business logic)
- Use Go idioms: small interfaces, accept interfaces, return structs
- Repository interfaces handle data access only — no business logic
- Service interfaces handle business logic — call repositories, never query DB directly
- Method signatures include `context.Context` as first parameter
- Return `error` as last return value

### Step 3: Present for Approval

Show the interface to the user before proceeding. Include:
- Interface name and package
- Each method with its signature
- Brief comment on what each method does
- Which OpenSpec requirement it satisfies

### Step 4: Implement

Only after approval:
1. Create `repository_sqlite.go` and/or `repository_postgres.go` for repository interfaces
2. Create `service.go` for service interfaces
3. Create `handlers.go` for HTTP handlers that use the service interface
4. Add `[SOMBRA:DEV]` logging at every boundary
5. Write `go test` for each implementation

## File Organization (STRICT)

```
server/internal/<subsystem>/
├── interfaces.go       # Go interfaces (define FIRST)
├── service.go          # Business logic (implements service interface)
├── handlers.go         # HTTP handlers (uses service interface)
├── repository_sqlite.go   # SQLite implementation
├── repository_postgres.go # PostgreSQL implementation
└── *_test.go           # Tests for all implementations
```

## Guardrails

- NEVER write implementation before the interface is defined and approved
- Handlers → Services → Repositories — no layer skipping
- No handler should directly query the database
- Each file does ONE thing
