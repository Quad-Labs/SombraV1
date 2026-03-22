# Capability: data-model

Repository pattern with Go interfaces -- SQLite implementation (WAL mode, single writer) for Tier 1, PostgreSQL implementation (connection pooling, partitioning-ready) for Tier 2+, embedded SQL migrations, ULID primary keys.

## ADDED Requirements

### Requirement: Go Interfaces for All Data Access
All data access MUST be defined through Go interfaces. The system SHALL define the following repository interfaces: UserRepo, ServerRepo, ChannelRepo, MessageRepo, MemberRepo, RoleRepo, InviteRepo, BotRepo, KeyPackageRepo, MLSGroupRepo, BlobMetaRepo, and AppServiceRepo. No concrete database implementation details SHALL leak through these interfaces.

#### Scenario: UserRepo interface defines user operations
- **WHEN** a developer inspects the UserRepo interface
- **THEN** it SHALL declare methods for creating, reading, updating, and deleting users, as well as querying by ID, email, and username, all returning domain types with no database-specific types in signatures

#### Scenario: All repositories are interfaces
- **WHEN** a developer inspects the data access layer
- **THEN** each of the twelve repositories (UserRepo, ServerRepo, ChannelRepo, MessageRepo, MemberRepo, RoleRepo, InviteRepo, BotRepo, KeyPackageRepo, MLSGroupRepo, BlobMetaRepo, AppServiceRepo) SHALL be defined as a Go interface

#### Scenario: Implementations satisfy the interface contract
- **WHEN** the SQLite or PostgreSQL implementation of any repository is compiled
- **THEN** the Go compiler SHALL verify that the implementation satisfies the corresponding interface at compile time

---

### Requirement: SQLite Implementation with WAL Mode and Single Writer
The SQLite implementation SHALL use WAL (Write-Ahead Logging) mode for concurrent read access. The writer connection MUST be limited to a single open connection via `SetMaxOpenConns(1)`. A configurable reader pool SHALL be provided for concurrent read operations.

#### Scenario: WAL mode is enabled on SQLite
- **WHEN** the SQLite database is opened
- **THEN** the implementation SHALL execute `PRAGMA journal_mode=WAL` and confirm that WAL mode is active

#### Scenario: Single writer connection
- **WHEN** the SQLite writer connection pool is configured
- **THEN** `SetMaxOpenConns(1)` SHALL be called on the writer pool to ensure only one write operation executes at a time

#### Scenario: Configurable reader pool
- **WHEN** the SQLite implementation is initialized
- **THEN** a separate reader connection pool SHALL be created with a configurable number of maximum open connections (defaulting to the number of CPU cores)

#### Scenario: Concurrent reads do not block on writes
- **WHEN** a write operation is in progress on the SQLite writer connection
- **THEN** concurrent read operations on the reader pool SHALL proceed without blocking, leveraging WAL mode's concurrent read capability

---

### Requirement: PostgreSQL Implementation with Connection Pooling
The PostgreSQL implementation SHALL use connection pooling for efficient database access. The implementation MUST support full ACID transactions, JOINs, and be partitioning-ready for large-scale deployments.

#### Scenario: Connection pool is configured
- **WHEN** the PostgreSQL implementation is initialized with `DB_URL=postgres://host:5432/sombra`
- **THEN** a connection pool SHALL be established with configurable maximum open connections, maximum idle connections, and connection lifetime settings

#### Scenario: ACID transactions are supported
- **WHEN** a service performs a multi-step database operation (e.g., creating a server with default roles and channels)
- **THEN** the PostgreSQL implementation SHALL wrap the operations in a transaction that either commits all changes or rolls back entirely on failure

#### Scenario: Partitioning-ready schema
- **WHEN** the PostgreSQL schema is inspected for high-volume tables (e.g., messages)
- **THEN** the schema SHALL use ULID-based primary keys and timestamp columns suitable for range partitioning without requiring schema migration

---

### Requirement: DB_TYPE Environment Variable Switches Implementations
The `DB_TYPE` environment variable SHALL control which database implementation is used. When `DB_TYPE` is set to `sqlite` (or unset), the server MUST use the SQLite implementation. When `DB_TYPE` is set to `postgres`, the server MUST use the PostgreSQL implementation. No other values SHALL be accepted.

#### Scenario: Default to SQLite when DB_TYPE is unset
- **WHEN** the server starts with no `DB_TYPE` environment variable set
- **THEN** the server SHALL use the SQLite implementation with `sombra.db` in the working directory

#### Scenario: Switch to PostgreSQL with DB_TYPE=postgres
- **WHEN** the server starts with `DB_TYPE=postgres` and a valid `DB_URL`
- **THEN** the server SHALL use the PostgreSQL implementation and connect to the specified database

#### Scenario: Reject invalid DB_TYPE values
- **WHEN** the server starts with `DB_TYPE=mysql`
- **THEN** the server SHALL fail to start and log an error message indicating that only `sqlite` and `postgres` are supported values for `DB_TYPE`

---

### Requirement: Embedded SQL Schema Migrations
Schema migrations SHALL be embedded in the Go binary using Go's `embed` package. Migration SQL files MUST be compiled into the binary so that the server auto-migrates on startup. Users SHALL NOT be required to run separate migration commands.

#### Scenario: Auto-migrate on startup
- **WHEN** the server starts and the database schema is not up to date
- **THEN** the server SHALL automatically apply all pending migrations in order before accepting connections

#### Scenario: Migrations are embedded in the binary
- **WHEN** the server binary is inspected
- **THEN** the SQL migration files SHALL be embedded via `//go:embed` directives and require no external migration files on disk

#### Scenario: Fresh database is fully migrated
- **WHEN** the server starts against a brand-new empty database
- **THEN** all migrations SHALL be applied sequentially, creating the complete schema from scratch

#### Scenario: Already-migrated database is not re-migrated
- **WHEN** the server starts against a database that has all migrations already applied
- **THEN** the server SHALL detect that no pending migrations exist and proceed to start without modifying the schema

#### Scenario: Migration failure prevents server startup
- **WHEN** a migration fails to apply (e.g., due to a schema conflict)
- **THEN** the server SHALL log the migration error and exit with a non-zero status code without accepting connections

---

### Requirement: Core Entity Definitions
The data model SHALL define the following core entities with their relationships: User (with Devices, KeyPackages, Sessions), Server (with Channels, Roles, Members, Invites, Bots, AppServices, AuditLogEntries), Channel (type: text|voice|forum, with MLSGroup, Messages, Threads, Pins, optional ActivityPub fields), Message (ULID id, ciphertext blob, Attachments, Reactions), and MLSGroup (channel_id, epoch, tree_hash).

#### Scenario: User entity with nested associations
- **WHEN** a User record is created
- **THEN** it SHALL support associated Devices (multi-device login), KeyPackages (MLS key material), and Sessions (active authentication sessions)

#### Scenario: Server entity with community structures
- **WHEN** a Server (community) record is queried with its associations
- **THEN** it SHALL include related Channels, Roles, Members, Invites, Bots, AppServices, and AuditLogEntries

#### Scenario: Channel entity with type discriminator
- **WHEN** a Channel record is created with `type: text`, `type: voice`, or `type: forum`
- **THEN** the channel type SHALL be persisted and determine available operations (e.g., voice channels support calls, forum channels support thread-first posting)

#### Scenario: Channel entity with optional ActivityPub fields
- **WHEN** a Channel has ActivityPub federation enabled
- **THEN** the channel record SHALL include optional fields for ActivityPub actor URI, inbox URL, outbox URL, and followers collection

#### Scenario: Message entity stores opaque ciphertext
- **WHEN** a Message record is created
- **THEN** it SHALL store a ULID as its primary key, an opaque ciphertext blob, and support associated Attachments and Reactions

#### Scenario: MLSGroup entity tracks epoch state
- **WHEN** an MLSGroup record is created for a channel
- **THEN** it SHALL store the channel_id, current epoch number, and tree_hash, enabling MLS group state verification

---

### Requirement: ULID Primary Keys
All primary keys across all entities MUST be ULIDs (Universally Unique Lexicographically Sortable Identifiers). ULIDs SHALL provide time-ordered, globally unique identifiers without requiring coordination between database nodes.

#### Scenario: Primary keys are valid ULIDs
- **WHEN** any entity (User, Server, Channel, Message, etc.) is created
- **THEN** its primary key SHALL be a ULID that encodes the creation timestamp in its most significant bits

#### Scenario: Lexicographic ordering matches chronological ordering
- **WHEN** entities are sorted by their ULID primary keys in lexicographic order
- **THEN** the resulting order SHALL match the chronological order of entity creation

#### Scenario: ULIDs are globally unique without coordination
- **WHEN** two server instances running concurrently generate ULIDs at the same millisecond
- **THEN** the generated ULIDs SHALL be unique due to the random component, without requiring a centralized ID generation service

---

### Requirement: Interface-First Development
Repository interfaces MUST be defined in an `interfaces.go` file before any implementation is written. All service and handler code MUST depend on the interface types, never on concrete implementation types. This SHALL enable seamless swapping of SQLite and PostgreSQL implementations.

#### Scenario: Interfaces defined before implementations
- **WHEN** a developer adds a new repository
- **THEN** the Go interface MUST be declared in `interfaces.go` before any SQLite or PostgreSQL implementation is written

#### Scenario: Services depend on interfaces
- **WHEN** a service struct is defined
- **THEN** its fields SHALL reference repository interface types (e.g., `UserRepo`) and SHALL NOT reference concrete types (e.g., `sqliteUserRepo` or `pgUserRepo`)

#### Scenario: Dependency injection at startup
- **WHEN** the server starts and `DB_TYPE` is resolved
- **THEN** the appropriate concrete implementations (SQLite or PostgreSQL) SHALL be instantiated and injected into services via their interface types

---

### Requirement: No Direct Database Access from Handlers
HTTP handlers MUST NOT directly query the database. Handlers SHALL call service methods, and services SHALL call repository methods. This three-layer architecture (handler -> service -> repository) MUST be enforced throughout the codebase.

#### Scenario: Handler calls service, not repository
- **WHEN** an HTTP handler processes a request to create a message
- **THEN** the handler SHALL call a service method (e.g., `MessageService.Create()`), which in turn calls the repository method (e.g., `MessageRepo.Create()`), and the handler SHALL NOT import or reference any repository type

#### Scenario: Service encapsulates business logic
- **WHEN** a service method is called to create a server with default roles
- **THEN** the service SHALL coordinate calls to `ServerRepo.Create()`, `RoleRepo.Create()`, and `ChannelRepo.Create()` within a transaction, encapsulating the business logic that handlers do not need to know about

#### Scenario: Repository is the only database-touching layer
- **WHEN** a search is performed across the codebase for raw SQL queries or database driver imports
- **THEN** those references SHALL appear only within repository implementation files (e.g., `sqlite_user_repo.go`, `pg_user_repo.go`) and migration files, never in handler or service files
