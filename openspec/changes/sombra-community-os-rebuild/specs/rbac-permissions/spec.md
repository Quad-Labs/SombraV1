## ADDED Requirements

### Requirement: Roles as Server-Level Permission Sets
Roles SHALL be defined as named sets of permissions assigned at the server level. Each role MUST contain a bitmask or set of discrete permissions (e.g., SEND_MESSAGES, MANAGE_CHANNELS, KICK_MEMBERS). A user's effective server-level permissions SHALL be the union of all permissions from all roles assigned to that user.

#### Scenario: User inherits permissions from assigned roles
- **WHEN** a user is assigned two roles, one granting SEND_MESSAGES and another granting MANAGE_CHANNELS
- **THEN** the user's effective server-level permissions include both SEND_MESSAGES and MANAGE_CHANNELS

#### Scenario: User with no additional roles has only @everyone permissions
- **WHEN** a user has no explicitly assigned roles beyond the default @everyone role
- **THEN** the user's effective server-level permissions are exactly those defined on the @everyone role

### Requirement: Channel-Level Permission Overrides
The server SHALL support per-role permission overrides at the channel level. Each override MUST specify allow and deny permission sets for a given role on a given channel. Deny overrides SHALL take precedence over allow overrides. Channel overrides MUST be evaluated on top of the user's server-level permissions.

#### Scenario: Channel override denies a server-level permission
- **WHEN** a user has SEND_MESSAGES granted at the server level but a channel override denies SEND_MESSAGES for one of the user's roles on a specific channel
- **THEN** the user cannot send messages in that channel

#### Scenario: Channel override allows a permission not granted at server level
- **WHEN** a user does not have MANAGE_MESSAGES at the server level but a channel override allows MANAGE_MESSAGES for one of the user's roles on a specific channel
- **THEN** the user can manage messages in that channel

#### Scenario: Deny takes precedence over allow at channel level
- **WHEN** a user has two roles on a channel, one with an allow override for SEND_MESSAGES and another with a deny override for SEND_MESSAGES
- **THEN** the deny override wins and the user cannot send messages in that channel

### Requirement: Permission Evaluation on Every API Call
The RBAC middleware MUST evaluate the authenticated user's permissions on every API call that requires authorization. The middleware SHALL compute the user's effective permissions by combining server-level role permissions with any applicable channel-level overrides, then verify the user has the required permission for the requested operation.

#### Scenario: Authorized API call succeeds
- **WHEN** an authenticated user sends a request to create a message in a channel and the user has the SEND_MESSAGES permission for that channel
- **THEN** the request proceeds to the handler and the message is created

#### Scenario: Unauthorized API call rejected
- **WHEN** an authenticated user sends a request to delete a channel and the user does not have the MANAGE_CHANNELS permission
- **THEN** the RBAC middleware responds with a 403 Forbidden status before the handler executes

### Requirement: Permission Evaluation on WebSocket Event Fan-Out
The server MUST evaluate permissions when fanning out events via WebSocket. Before delivering an event to a connected client, the server SHALL verify that the recipient has the required permission to receive that event in the relevant channel or server context.

#### Scenario: Event delivered to authorized user
- **WHEN** a MESSAGE_CREATE event occurs in a channel and a connected WebSocket client has READ_MESSAGES permission for that channel
- **THEN** the event is delivered to that client

#### Scenario: Event withheld from unauthorized user
- **WHEN** a MESSAGE_CREATE event occurs in a channel and a connected WebSocket client does not have READ_MESSAGES permission for that channel
- **THEN** the event is not delivered to that client

### Requirement: Default Roles
Every server MUST have two default roles created automatically: an @everyone role that is assigned to all members and defines the base permission set, and a server owner role that grants all permissions. The server owner role SHALL NOT be deletable or assignable to other users.

#### Scenario: @everyone role exists on server creation
- **WHEN** a new server is created
- **THEN** an @everyone role is automatically created with a base set of permissions and is assigned to all members

#### Scenario: Server owner has full permissions
- **WHEN** a user creates a new server
- **THEN** the user is assigned the server owner role which grants all permissions, and this role cannot be deleted or transferred to another user via the roles API

#### Scenario: Server owner role cannot be deleted
- **WHEN** any user (including the server owner) sends a DELETE request to remove the server owner role
- **THEN** the server responds with a 403 Forbidden or 400 Bad Request status and the role remains intact

### Requirement: Custom Role Management
The server SHALL support creating, ordering, and assigning custom roles. Custom roles MUST have a position value that determines their hierarchy — higher-positioned roles take visual and logical precedence. A user SHALL only be able to manage roles positioned below their own highest role.

#### Scenario: Create a custom role
- **WHEN** an authorized user sends a POST request to create a new role with a name and permission set
- **THEN** the server creates the role, assigns it a position in the hierarchy, and returns the new role object

#### Scenario: Role ordering determines hierarchy
- **WHEN** two custom roles exist with different position values
- **THEN** the role with the higher position value takes precedence in display ordering and permission hierarchy

#### Scenario: User cannot manage roles above their own
- **WHEN** a user with a highest role at position 5 attempts to edit or delete a role at position 10
- **THEN** the server rejects the request with a 403 Forbidden status

#### Scenario: Assign a custom role to a member
- **WHEN** an authorized user assigns a custom role to a server member
- **THEN** the member's effective permissions are recalculated to include the permissions from the newly assigned role

### Requirement: Role CRUD Endpoints
The server SHALL expose the following endpoints for role management: POST `/api/v1/servers/:id/roles` to create a role, GET `/api/v1/servers/:id/roles` to list all roles for a server, and DELETE `/api/v1/servers/:id/roles/:roleId` to delete a role.

#### Scenario: Create role via POST
- **WHEN** an authorized user sends a POST request to `/api/v1/servers/:id/roles` with a role name and permissions
- **THEN** the server creates the role and responds with a 201 Created status and the new role object

#### Scenario: List roles via GET
- **WHEN** an authenticated member sends a GET request to `/api/v1/servers/:id/roles`
- **THEN** the server responds with a 200 OK status and an array of all roles for that server, ordered by position

#### Scenario: Delete role via DELETE
- **WHEN** an authorized user sends a DELETE request to `/api/v1/servers/:id/roles/:roleId`
- **THEN** the server deletes the role, removes it from all members, recalculates affected members' permissions, and responds with a 200 OK or 204 No Content status

#### Scenario: Unauthorized role creation rejected
- **WHEN** a user without the MANAGE_ROLES permission sends a POST request to `/api/v1/servers/:id/roles`
- **THEN** the server responds with a 403 Forbidden status
