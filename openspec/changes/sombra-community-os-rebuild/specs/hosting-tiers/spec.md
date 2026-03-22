# Capability: hosting-tiers

4-tier hosting model -- Tier 1 (SQLite, zero-config ./sombra), Tier 2 (Docker + PostgreSQL), Tier 3 (Sombra Network managed), Tier 4 (own cloud Terraform/Pulumi), sombra migrate dual-pipeline tool, sombra --tunnel Cloudflare Quick Tunnel.

## ADDED Requirements

### Requirement: Tier 1 Minecraft Mode
Tier 1 SHALL provide a zero-configuration, free hosting mode using SQLite as the embedded database, P2P-only media (no SFU), and filesystem blob storage. Community apps MUST NOT be available in Tier 1. The server MUST start with `./sombra` and no additional setup.

#### Scenario: Zero-config startup on Tier 1
- **WHEN** a user runs `./sombra` with no environment variables or arguments on a clean machine
- **THEN** the server SHALL start with SQLite (WAL mode), filesystem blob storage, embedded NATS, and P2P-only media, binding to `0.0.0.0:8080`

#### Scenario: No community apps on Tier 1
- **WHEN** a user attempts to install or enable a community app on a Tier 1 instance
- **THEN** the server SHALL reject the request with an error indicating community apps require Tier 2 or above

#### Scenario: P2P-only media on Tier 1
- **WHEN** a voice or video call is initiated on a Tier 1 instance
- **THEN** the system SHALL use only direct peer-to-peer WebRTC connections and MUST NOT attempt to route media through an SFU

---

### Requirement: Tier 2 Docker Pro
Tier 2 SHALL provide a free self-hosted mode using Docker Compose with PostgreSQL as the database. An optional local mediasoup SFU MAY be included. Community apps SHALL be Docker-managed containers orchestrated by the Sombra server.

#### Scenario: Docker Compose starts Tier 2 services
- **WHEN** a user runs `docker compose up` using a Tier 2 template
- **THEN** the Sombra server, PostgreSQL database, and optionally a mediasoup SFU worker SHALL start as Docker containers

#### Scenario: Community apps run as Docker containers on Tier 2
- **WHEN** a server admin installs a community app on a Tier 2 instance
- **THEN** the app SHALL be launched as a Docker container managed by the Sombra server's app orchestrator

#### Scenario: Optional SFU on Tier 2
- **WHEN** a Tier 2 deployment includes a mediasoup SFU worker container
- **THEN** group calls with 3 or more participants SHALL route media through the SFU instead of using mesh topology

---

### Requirement: Tier 3 Sombra Network
Tier 3 SHALL be a subscription-based managed hosting tier operated by the Sombra project. It MUST use PostgreSQL, provide a full SFU and TURN relay fleet, and run community apps as Kubernetes-managed auto-scaled workloads.

#### Scenario: Sombra Network provides managed PostgreSQL
- **WHEN** a user creates a community on the Sombra Network
- **THEN** the system SHALL provision a managed PostgreSQL database for that community without requiring the user to configure any database settings

#### Scenario: Full SFU and TURN fleet on Tier 3
- **WHEN** a voice or video call is initiated on a Tier 3 instance
- **THEN** the system SHALL route media through the managed SFU fleet and provide TURN relay servers for participants behind restrictive NATs

#### Scenario: Auto-scaled community apps on Tier 3
- **WHEN** a community app experiences increased load on a Tier 3 instance
- **THEN** the Kubernetes orchestrator SHALL auto-scale the app's pods based on resource utilization

---

### Requirement: Tier 4 Own Cloud
Tier 4 SHALL support deploying Sombra to a user's own cloud infrastructure using Terraform or Pulumi modules. It MUST use PostgreSQL, self-provisioned media infrastructure (SFU, TURN), and Docker or Kubernetes for community apps. Cloud billing SHALL be the user's responsibility.

#### Scenario: Terraform deployment to own cloud
- **WHEN** a user runs `terraform apply` with the Sombra Terraform module targeting their AWS/GCP/Azure account
- **THEN** the infrastructure SHALL be provisioned including the Sombra server, PostgreSQL database, and media infrastructure on the user's cloud account

#### Scenario: Pulumi deployment to own cloud
- **WHEN** a user runs `pulumi up` with the Sombra Pulumi module targeting their cloud provider
- **THEN** the infrastructure SHALL be provisioned with the same components as the Terraform deployment

#### Scenario: Self-provisioned media on Tier 4
- **WHEN** a Tier 4 deployment includes SFU and TURN server configurations
- **THEN** the Sombra server SHALL use the user-provisioned media infrastructure for voice and video calls

---

### Requirement: Single Binary Across All Tiers
The same Go binary MUST run across all four tiers. The difference between tiers SHALL be configuration (environment variables) only, not code. There MUST NOT be separate builds, feature flags, or conditional compilation for different tiers.

#### Scenario: Same binary on Tier 1 and Tier 2
- **WHEN** the same compiled `sombra` binary is executed with `DB_TYPE=sqlite` and then with `DB_TYPE=postgres`
- **THEN** the binary SHALL operate correctly in both configurations without requiring recompilation

#### Scenario: No tier-specific builds
- **WHEN** a developer inspects the build system
- **THEN** there SHALL be exactly one build target producing one binary, with no build tags, feature flags, or conditional compilation directives that differentiate between tiers

#### Scenario: Binary on Tier 3 managed infrastructure
- **WHEN** the Sombra Network deploys the server binary on managed Kubernetes
- **THEN** the binary running in Tier 3 SHALL be byte-identical to the binary a user downloads for Tier 1

---

### Requirement: Dual-Pipeline Migration Tool
The `sombra migrate` command SHALL provide a dual-pipeline migration tool that migrates both SQLite data to PostgreSQL AND filesystem blobs to S3-compatible storage. Migrations MUST be checkpoint-based and resumable on crash. The tool MUST preserve old data until the user explicitly deletes it.

#### Scenario: SQLite to PostgreSQL data migration
- **WHEN** a user runs `sombra migrate --target-db postgres://host:5432/sombra`
- **THEN** the tool SHALL copy all data from the local SQLite database to the target PostgreSQL database, preserving all records and relationships

#### Scenario: Filesystem blobs to S3 migration
- **WHEN** a user runs `sombra migrate --target-blobs s3://bucket-name`
- **THEN** the tool SHALL upload all local filesystem blobs to the target S3-compatible storage, preserving blob metadata and references

#### Scenario: Checkpoint-based resumable migration
- **WHEN** a migration is interrupted by a crash or power failure at 60% completion
- **THEN** running `sombra migrate` again with the same arguments SHALL resume from the last checkpoint and complete the remaining 40% without re-processing already-migrated data

#### Scenario: Old data preserved until explicit deletion
- **WHEN** a migration completes successfully
- **THEN** the original SQLite database and filesystem blobs SHALL remain intact, and the tool SHALL NOT delete them unless the user explicitly runs `sombra migrate --cleanup`

#### Scenario: Dual pipeline runs concurrently
- **WHEN** a user runs `sombra migrate` with both `--target-db` and `--target-blobs` specified
- **THEN** the SQLite-to-PostgreSQL and filesystem-to-S3 pipelines SHALL run concurrently, each with independent checkpoints

---

### Requirement: Cloudflare Quick Tunnel via --tunnel Flag
The `sombra --tunnel` flag SHALL start a Cloudflare Quick Tunnel that provides instant HTTPS access to the local Sombra instance. This MUST solve the Secure Context requirement for WebRTC and Web Crypto APIs in browsers.

#### Scenario: Start server with tunnel
- **WHEN** a user runs `./sombra --tunnel`
- **THEN** the server SHALL start normally and additionally establish a Cloudflare Quick Tunnel, printing the assigned `https://*.trycloudflare.com` URL to stdout

#### Scenario: HTTPS tunnel enables WebRTC in browsers
- **WHEN** a client connects to the Sombra instance via the Cloudflare Quick Tunnel HTTPS URL
- **THEN** the browser's Secure Context requirement SHALL be satisfied, enabling WebRTC and Web Crypto API usage without certificate warnings

#### Scenario: Tunnel operates without Cloudflare account
- **WHEN** a user runs `./sombra --tunnel` without any Cloudflare account credentials configured
- **THEN** the Quick Tunnel SHALL be established using Cloudflare's free unauthenticated tunnel service

---

### Requirement: Pre-Built Docker Compose Templates
The project SHALL provide pre-built Docker Compose templates for common deployment setups. Templates MUST cover at minimum: Tier 2 basic (Sombra + PostgreSQL), Tier 2 with SFU (Sombra + PostgreSQL + mediasoup), and Tier 2 full (Sombra + PostgreSQL + mediasoup + TURN).

#### Scenario: Basic Tier 2 template
- **WHEN** a user copies the basic Docker Compose template and runs `docker compose up`
- **THEN** the Sombra server and PostgreSQL database SHALL start and be operational with no additional configuration

#### Scenario: Full Tier 2 template with media infrastructure
- **WHEN** a user copies the full Docker Compose template and runs `docker compose up`
- **THEN** the Sombra server, PostgreSQL, mediasoup SFU, and TURN relay SHALL all start and be interconnected

#### Scenario: Templates require no manual editing for basic use
- **WHEN** a user uses any pre-built Docker Compose template without modification
- **THEN** the services SHALL start successfully with sensible defaults, including auto-generated secrets for inter-service communication

---

### Requirement: Cloud Deployment Wizard Modules
The project SHALL provide Terraform and Pulumi modules for deploying Sombra to major cloud providers. Modules MUST support at minimum AWS, GCP, and Azure. Each module SHALL provision the Sombra server, PostgreSQL, media infrastructure, and networking.

#### Scenario: Terraform module deploys to AWS
- **WHEN** a user configures the Sombra Terraform module with AWS provider credentials and runs `terraform apply`
- **THEN** the module SHALL provision an ECS/EC2 instance running Sombra, an RDS PostgreSQL database, and associated networking and security groups

#### Scenario: Pulumi module deploys to GCP
- **WHEN** a user configures the Sombra Pulumi module with GCP provider credentials and runs `pulumi up`
- **THEN** the module SHALL provision a Cloud Run or GCE instance running Sombra, a Cloud SQL PostgreSQL database, and associated networking

#### Scenario: Modules include media infrastructure
- **WHEN** a user deploys using any cloud module with media infrastructure enabled
- **THEN** the module SHALL provision mediasoup SFU instances and TURN relay servers alongside the core Sombra server

---

### Requirement: Bots Work on All Tiers Including Tier 1
Bots MUST be functional on all tiers, including Tier 1 localhost deployments. Bot API endpoints and WebSocket event delivery SHALL operate identically regardless of the hosting tier.

#### Scenario: Bot connects to Tier 1 localhost
- **WHEN** a bot uses an API token to connect to a Tier 1 Sombra instance running on `http://localhost:8080`
- **THEN** the bot SHALL authenticate successfully and receive WebSocket events for its subscribed channels

#### Scenario: Same bot code works across tiers
- **WHEN** a bot written against the Sombra Bot SDK is pointed at a Tier 1 instance and then reconfigured to point at a Tier 3 instance
- **THEN** the bot SHALL function identically on both tiers without code changes

---

### Requirement: No Enterprise Edition
There SHALL NOT be an enterprise edition, premium build, or feature-gated binary. The same single binary MUST contain all features for all tiers. There SHALL be no commercial license that unlocks additional server-side capabilities.

#### Scenario: All features present in single binary
- **WHEN** a user downloads the Sombra binary
- **THEN** the binary SHALL contain every feature available across all tiers, with no features locked behind a license key or payment

#### Scenario: No feature flags for paid capabilities
- **WHEN** a developer inspects the server source code
- **THEN** there SHALL be no conditional logic that enables or disables features based on a license, subscription status, or edition flag
