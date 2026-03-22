## ADDED Requirements

### Requirement: Single Binary Deployment
The server SHALL be compiled via `go build` into a single executable binary that contains all subsystems (HTTP server, WebSocket gateway, embedded NATS, static web UI assets). No external processes or sidecar binaries SHALL be required for Tier 1 operation.

#### Scenario: Build produces one executable
- **WHEN** a developer runs `go build` targeting the server package
- **THEN** exactly one executable binary is produced that can start the entire Sombra server

#### Scenario: Binary runs without external dependencies on Tier 1
- **WHEN** the binary is executed on a clean machine with no pre-installed services
- **THEN** the server starts successfully using embedded SQLite, embedded NATS, and filesystem blob storage without requiring any external processes

### Requirement: Chi Router with Versioned API
The server SHALL use the chi router to expose all REST API endpoints under the `/api/v1/` prefix. All API routes MUST be grouped under this versioned namespace.

#### Scenario: API routes are accessible under versioned prefix
- **WHEN** a client sends a request to `/api/v1/auth/login`
- **THEN** the request is routed to the login handler via the chi router

#### Scenario: Requests outside versioned prefix do not reach API handlers
- **WHEN** a client sends a request to `/auth/login` (without the `/api/v1/` prefix)
- **THEN** the request does not reach the authentication API handler and the server responds with a 404 status or serves the web UI fallback

### Requirement: Middleware Chain Order
The chi middleware chain MUST be applied in the following strict order for every `/api/v1/` request: Recoverer, RequestID, DevLogger, RateLimiter, Authenticator, RBAC. The order SHALL NOT be rearranged, as each middleware depends on context set by its predecessors.

#### Scenario: Panic recovery wraps all subsequent middleware
- **WHEN** a panic occurs in any handler or downstream middleware
- **THEN** the Recoverer middleware catches the panic, logs the stack trace, and returns a 500 Internal Server Error response without crashing the process

#### Scenario: RequestID is available to all downstream middleware
- **WHEN** a request passes through the RequestID middleware
- **THEN** a unique request ID is injected into the request context and is available to DevLogger, RateLimiter, Authenticator, RBAC, and the final handler

#### Scenario: DevLogger runs before authentication
- **WHEN** a request arrives and `SOMBRA_DEV_LOG` is enabled
- **THEN** the DevLogger middleware logs the request method, path, and request ID before the Authenticator middleware executes, ensuring unauthenticated requests are also logged

#### Scenario: RateLimiter runs before authentication
- **WHEN** a client exceeds the rate limit
- **THEN** the RateLimiter middleware rejects the request with a 429 Too Many Requests response before any authentication or authorization logic executes

#### Scenario: Authenticator runs before RBAC
- **WHEN** a request requires both authentication and authorization
- **THEN** the Authenticator middleware validates the session token and injects the user identity into the context before the RBAC middleware evaluates permissions

#### Scenario: RBAC is the last middleware before the handler
- **WHEN** a request reaches the RBAC middleware
- **THEN** the RBAC middleware has access to the authenticated user identity (from Authenticator) and evaluates role-based permissions before passing control to the route handler

### Requirement: Embedded NATS JetStream for Tier 1
The server SHALL embed a NATS server with JetStream enabled when running in Tier 1 mode. For Tier 2 and above, the server MUST connect to an external NATS cluster specified by the `NATS_URL` environment variable.

#### Scenario: Tier 1 starts embedded NATS
- **WHEN** the server starts without a `NATS_URL` environment variable (or with `NATS_URL` unset)
- **THEN** the server starts an in-process embedded NATS server with JetStream enabled and connects to it internally

#### Scenario: Tier 2+ connects to external NATS
- **WHEN** the server starts with `NATS_URL` set to an external NATS address (e.g., `nats://nats.example.com:4222`)
- **THEN** the server connects to the external NATS cluster and does not start an embedded NATS instance

#### Scenario: JetStream streams are available for event replay
- **WHEN** the embedded or external NATS connection is established
- **THEN** JetStream streams are created for event buffering, enabling WebSocket reconnection replay

### Requirement: Environment Variable Configuration
All server configuration MUST be supplied exclusively through environment variables. The server SHALL NOT read configuration from files, command-line flags, or interactive prompts. Required variables include but are not limited to: `DB_TYPE`, `DB_URL`, `NATS_URL`, `SOMBRA_DEV_LOG`.

#### Scenario: Configuration is read from environment
- **WHEN** the server starts with `DB_TYPE=postgres` and `DB_URL=postgres://localhost:5432/sombra` set as environment variables
- **THEN** the server connects to the specified PostgreSQL database using those values

#### Scenario: Unrecognized configuration sources are ignored
- **WHEN** a config file named `sombra.yaml` exists in the working directory
- **THEN** the server ignores it and reads configuration only from environment variables

### Requirement: Zero-Config Startup
The server SHALL start successfully with no arguments and no environment variables set. In this default state, it MUST create a `sombra.db` SQLite database in the working directory, create a `sombra-data/` directory for blob storage, start the embedded NATS server, bind to `0.0.0.0:8080`, and serve the web UI.

#### Scenario: Default startup with no arguments
- **WHEN** a user executes `./sombra` with no arguments and no environment variables
- **THEN** the server creates `sombra.db` in the current working directory, creates a `sombra-data/` directory, starts embedded NATS with JetStream, binds the HTTP server to `0.0.0.0:8080`, and serves the web UI at the root path

#### Scenario: Server is accessible after zero-config start
- **WHEN** the server has started with default configuration
- **THEN** a client can connect to `http://localhost:8080` and receive the web UI, and API requests to `http://localhost:8080/api/v1/` are routed correctly

### Requirement: Dev Logging with SOMBRA_DEV_LOG
The DevLogger middleware SHALL emit structured log lines prefixed with `[SOMBRA:DEV]` when the `SOMBRA_DEV_LOG` environment variable is set to `true`. When `SOMBRA_DEV_LOG` is unset or set to any other value, dev logging MUST be completely disabled.

#### Scenario: Dev logging enabled
- **WHEN** the server starts with `SOMBRA_DEV_LOG=true`
- **THEN** every API request produces structured log output prefixed with `[SOMBRA:DEV]` containing the request method, path, request ID, response status, and duration

#### Scenario: Dev logging disabled by default
- **WHEN** the server starts without the `SOMBRA_DEV_LOG` environment variable set
- **THEN** no `[SOMBRA:DEV]` prefixed log lines are emitted during request processing

#### Scenario: Dev logging disabled with explicit false
- **WHEN** the server starts with `SOMBRA_DEV_LOG=false`
- **THEN** no `[SOMBRA:DEV]` prefixed log lines are emitted during request processing

### Requirement: Health Check Endpoint
The server SHALL expose a health check endpoint that returns the server's operational status. This endpoint MUST be accessible without authentication.

#### Scenario: Health check returns OK
- **WHEN** a client sends a GET request to the health check endpoint
- **THEN** the server responds with a 200 OK status and a response body indicating the server is healthy

#### Scenario: Health check is unauthenticated
- **WHEN** a client sends a GET request to the health check endpoint without an authorization header
- **THEN** the server responds with a 200 OK status without requiring a session token

### Requirement: Graceful Shutdown
The server MUST handle OS termination signals (SIGINT, SIGTERM) by initiating a graceful shutdown sequence. The sequence SHALL stop accepting new connections, wait for in-flight requests to complete (up to a configurable timeout), drain the NATS connection, close the database connection, and then exit with status code 0.

#### Scenario: SIGTERM triggers graceful shutdown
- **WHEN** the server process receives a SIGTERM signal
- **THEN** the server stops accepting new HTTP connections, waits for in-flight requests to complete, drains the NATS connection, closes the database, and exits with code 0

#### Scenario: SIGINT triggers graceful shutdown
- **WHEN** the server process receives a SIGINT signal (e.g., Ctrl+C)
- **THEN** the server performs the same graceful shutdown sequence as for SIGTERM

#### Scenario: In-flight requests complete before shutdown
- **WHEN** the server receives SIGTERM while HTTP requests are being processed
- **THEN** those in-flight requests are allowed to complete before the server closes the HTTP listener
