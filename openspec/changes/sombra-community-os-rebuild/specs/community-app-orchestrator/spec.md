## ADDED Requirements

### Requirement: ServiceManager Interface with Tiered Implementations
The community app orchestrator SHALL define a `ServiceManager` Go interface that abstracts all container/service lifecycle operations. The system MUST provide three implementations: `NoopManager` for Tier 1 (returns "not supported" for all operations), `DockerManager` for Tier 2 and Tier 4 (uses the Docker SDK), and `KubernetesManager` for Tier 3 (uses the Kubernetes client-go library). The active implementation MUST be selected based on the detected or configured hosting tier.

#### Scenario: Tier 1 uses NoopManager
- **WHEN** the server is running in Tier 1 mode (zero-config binary, no Docker socket available)
- **THEN** the `NoopManager` is used, and all app install/remove/lifecycle requests return a clear error indicating community apps are not supported in Tier 1

#### Scenario: Tier 2 uses DockerManager
- **WHEN** the server is running in Tier 2 mode with a Docker socket available
- **THEN** the `DockerManager` is instantiated and used for all container lifecycle operations via the Docker SDK

#### Scenario: Tier 3 uses KubernetesManager
- **WHEN** the server is running in Tier 3 mode with Kubernetes credentials configured
- **THEN** the `KubernetesManager` is instantiated and manages app workloads as Kubernetes Deployments and Services

#### Scenario: Tier 4 uses DockerManager
- **WHEN** the server is running in Tier 4 mode (own cloud with Docker)
- **THEN** the `DockerManager` is used for container lifecycle operations, same as Tier 2

### Requirement: App Lifecycle Management
The `ServiceManager` interface SHALL support the following lifecycle operations: install, configure, remove, and health check. Installing an app MUST pull the container image, create the container with the specified configuration, and start it. Removing an app MUST stop the container, remove it, and clean up associated resources. Health checks MUST report the current running state of the app.

#### Scenario: App installation pulls and starts container
- **WHEN** a server owner installs a community app
- **THEN** the `ServiceManager` pulls the specified container image, creates the container with ports, volumes, and environment variables from the app manifest, starts the container, and reports success

#### Scenario: App removal stops and cleans up container
- **WHEN** a server owner removes an installed community app
- **THEN** the `ServiceManager` stops the running container, removes the container, and cleans up any associated Docker network connections

#### Scenario: App health check reports running state
- **WHEN** the system performs a health check on an installed app
- **THEN** the `ServiceManager` returns the container's current state (running, stopped, restarting, or not found)

#### Scenario: App configuration update
- **WHEN** a server owner updates the configuration of an installed app (e.g., changes an environment variable)
- **THEN** the `ServiceManager` recreates the container with the updated configuration and starts it

### Requirement: Declarative App Catalog with Manifests
Each community app MUST be defined by a `manifest.json` file that declares the app's container image, exposed ports, required volumes, environment variable schema, OIDC client configuration, resource limits, and network whitelist rules. The server SHALL load app manifests from the `apps/` directory in the monorepo and from any user-provided manifest URLs.

#### Scenario: Curated app has a valid manifest
- **WHEN** the server reads the `apps/jellyfin/manifest.json` file
- **THEN** the manifest contains at minimum: `name`, `image`, `ports`, `volumes`, `oidc` (client config), and `resources` (CPU/memory limits) fields

#### Scenario: Manifest defines OIDC integration
- **WHEN** an app manifest includes an `oidc` section with `redirect_uri` and `scopes`
- **THEN** the orchestrator automatically registers the app as an OIDC client with the embedded OIDC provider during installation

#### Scenario: Invalid manifest is rejected
- **WHEN** a server owner attempts to install an app with a manifest missing required fields
- **THEN** the installation fails with a descriptive error indicating which required fields are missing

### Requirement: Internal Reverse Proxy
The server SHALL include an internal reverse proxy that routes all requests matching `/app/{name}/*` to the corresponding community app's container. The proxy MUST resolve the target container's IP address and port dynamically from the `ServiceManager`. Only port 8080 on the Sombra server needs to be exposed; all community apps SHALL be accessible exclusively through this reverse proxy.

#### Scenario: Request is proxied to correct container
- **WHEN** a client sends a request to `/app/jellyfin/web/index.html`
- **THEN** the reverse proxy strips the `/app/jellyfin` prefix and forwards the request to the Jellyfin container's internal IP and configured port, returning the container's response to the client

#### Scenario: Request to unknown app returns 404
- **WHEN** a client sends a request to `/app/nonexistent/path`
- **THEN** the reverse proxy responds with a 404 Not Found status

#### Scenario: Only port 8080 is externally exposed
- **WHEN** community apps are installed and running
- **THEN** the apps are accessible only through the Sombra server's port 8080 via the `/app/{name}/*` routes; no additional ports need to be opened on the host firewall

#### Scenario: WebSocket proxying is supported
- **WHEN** a community app requires WebSocket connections (e.g., Gitea live updates)
- **THEN** the reverse proxy upgrades the connection and proxies WebSocket frames bidirectionally between the client and the app container

### Requirement: OIDC Authentication at Proxy Layer
The reverse proxy SHALL enforce OIDC authentication for all requests to community apps that declare OIDC integration in their manifest. Authentication MUST be checked at the proxy layer before the request reaches the community app container. If the user is not authenticated, the proxy SHALL redirect them to the "Log in with Sombra" flow.

#### Scenario: Unauthenticated user is redirected to login
- **WHEN** an unauthenticated user accesses `/app/nextcloud/`
- **THEN** the reverse proxy redirects the user to the Sombra OIDC authorization endpoint with the Nextcloud app's configured client ID and redirect URI

#### Scenario: Authenticated user's request is proxied with identity headers
- **WHEN** an authenticated user accesses `/app/nextcloud/` with a valid session
- **THEN** the reverse proxy injects identity headers (e.g., `X-Sombra-User-ID`, `X-Sombra-Username`, `X-Sombra-Roles`) into the proxied request and forwards it to the Nextcloud container

#### Scenario: App without OIDC config is publicly accessible
- **WHEN** an app manifest does not include an `oidc` section
- **THEN** the reverse proxy forwards requests to the app container without requiring authentication

### Requirement: SSRF Prevention via Network Isolation
Community app containers MUST be placed on an isolated Docker network (`app_net`) that prevents them from reaching the Go server's internal APIs, the database, NATS, or the wider internet unless explicitly whitelisted in the app manifest. The `app_net` network SHALL have no default route to the host network or other internal services.

#### Scenario: App container cannot reach Sombra internal API
- **WHEN** a community app container attempts to make an HTTP request to the Sombra server's internal API (e.g., `http://host.docker.internal:8080/api/v1/`)
- **THEN** the request is blocked by network isolation and the connection fails

#### Scenario: App container cannot reach the database
- **WHEN** a community app container attempts to connect to the PostgreSQL or SQLite database
- **THEN** the connection is blocked by network isolation

#### Scenario: Whitelisted external access is permitted
- **WHEN** an app manifest includes a `network_whitelist` entry for `api.themoviedb.org:443` and the Plex container makes an HTTPS request to that host
- **THEN** the request is allowed through the network isolation rules

#### Scenario: Non-whitelisted internet access is blocked
- **WHEN** a community app container without internet whitelist entries attempts to make an outbound HTTP request to an arbitrary external host
- **THEN** the connection is blocked by network isolation

### Requirement: Resource Isolation via Container Limits
Each community app container MUST have CPU and memory limits applied as specified in the app manifest. The `ServiceManager` SHALL enforce these limits using Docker container resource constraints (or Kubernetes resource requests/limits). If no limits are specified in the manifest, the system MUST apply sensible defaults to prevent a single app from consuming all host resources.

#### Scenario: Container respects declared memory limit
- **WHEN** a community app with a manifest declaring `"memory": "512m"` is installed
- **THEN** the Docker container is created with a 512MB memory limit and the container is OOM-killed if it exceeds this limit

#### Scenario: Container respects declared CPU limit
- **WHEN** a community app with a manifest declaring `"cpus": "1.0"` is installed
- **THEN** the Docker container is created with a CPU limit of 1.0 cores

#### Scenario: Default limits are applied when manifest omits them
- **WHEN** a community app manifest does not include a `resources` section
- **THEN** the `ServiceManager` applies default resource limits (e.g., 512MB memory, 1.0 CPU) to the container

### Requirement: Curated App Templates
The system SHALL provide curated app manifest templates for popular self-hosted applications including but not limited to: Plex, Jellyfin, Nextcloud, Gitea, and Minecraft. Each curated template MUST include a tested manifest with appropriate image, ports, volumes, OIDC configuration, resource limits, and network whitelist entries.

#### Scenario: Curated app installs without custom configuration
- **WHEN** a server owner selects "Jellyfin" from the curated app catalog
- **THEN** the system uses the curated `jellyfin/manifest.json` template to install Jellyfin with sensible default configuration, and the app is accessible at `/app/jellyfin/`

#### Scenario: Curated app templates are overridable
- **WHEN** a server owner installs a curated app and provides custom environment variable overrides
- **THEN** the custom values are merged with the template defaults, with custom values taking precedence

### Requirement: Custom Docker Image Installation
Server owners SHALL be able to install any Docker image by providing a container image URL and a custom manifest. The system MUST NOT restrict installations to only curated apps. The server owner MUST acknowledge that custom images are not vetted when installing non-curated apps.

#### Scenario: Server owner installs custom Docker image
- **WHEN** a server owner provides a Docker image URL (e.g., `ghcr.io/my-org/my-app:latest`) and a valid manifest
- **THEN** the `ServiceManager` pulls the image, creates the container per the manifest, and makes the app accessible at `/app/{name}/`

#### Scenario: Custom image without manifest uses defaults
- **WHEN** a server owner provides only a Docker image URL without a full manifest
- **THEN** the system creates a minimal manifest with the image URL, default resource limits, default network isolation (no internet access), and no OIDC integration

### Requirement: App Management API Endpoints
The server SHALL expose the following REST API endpoints for community app management: `GET /api/v1/servers/:id/apps` to list installed apps, `POST /api/v1/servers/:id/apps` to install a new app, and `DELETE /api/v1/servers/:id/apps/:appId` to remove an installed app. All endpoints MUST require authentication and appropriate server-level permissions (server owner or admin role).

#### Scenario: List installed apps
- **WHEN** an authenticated server admin sends a `GET /api/v1/servers/:id/apps` request
- **THEN** the server responds with a 200 OK status and a JSON array of installed apps, each including the app name, status, image, and accessible URL path

#### Scenario: Install a new app
- **WHEN** an authenticated server owner sends a `POST /api/v1/servers/:id/apps` request with a valid app manifest or curated app name in the request body
- **THEN** the server installs the app via the `ServiceManager`, registers the OIDC client (if applicable), configures the reverse proxy route, and responds with a 201 Created status

#### Scenario: Remove an installed app
- **WHEN** an authenticated server owner sends a `DELETE /api/v1/servers/:id/apps/:appId` request
- **THEN** the server removes the app via the `ServiceManager`, deregisters the OIDC client, removes the reverse proxy route, and responds with a 200 OK status

#### Scenario: Non-admin cannot manage apps
- **WHEN** an authenticated user without server owner or admin role sends a `POST /api/v1/servers/:id/apps` request
- **THEN** the server responds with a 403 Forbidden status

#### Scenario: App management unavailable on Tier 1
- **WHEN** an authenticated server owner sends a `POST /api/v1/servers/:id/apps` request on a Tier 1 server
- **THEN** the server responds with a 501 Not Implemented status indicating community apps require Tier 2 or higher
