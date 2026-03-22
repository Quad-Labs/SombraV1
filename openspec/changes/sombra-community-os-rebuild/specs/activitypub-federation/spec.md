## ADDED Requirements

### Requirement: Federation Scoped Exclusively to Forum Channels
ActivityPub federation SHALL apply exclusively to Forum channels. Text channels, voice channels, and direct messages MUST NOT produce or accept any ActivityPub activities. Any attempt to enable federation on a non-Forum channel type MUST be rejected.

#### Scenario: Forum channel federates
- **WHEN** a Forum channel has federation enabled and a new thread is posted
- **THEN** the server creates an ActivityPub Create activity with a Page object and delivers it to all subscribed remote instances

#### Scenario: Text channel does not federate
- **WHEN** a message is sent in a standard text channel
- **THEN** no ActivityPub activity is created or delivered, regardless of any server-level federation settings

#### Scenario: Voice channel does not federate
- **WHEN** a voice channel event occurs (call start, participant join)
- **THEN** no ActivityPub activity is created or delivered

#### Scenario: DM does not federate
- **WHEN** a direct message is sent between users
- **THEN** no ActivityPub activity is created or delivered

### Requirement: Per-Forum Federation Toggle
Each Forum channel MUST have a `federation_enabled` boolean field that controls whether that forum participates in ActivityPub federation. This toggle SHALL default to `false`. When set to `true`, the forum produces outbound activities and accepts inbound activities. When set to `false`, the forum MUST NOT produce or accept any ActivityPub activities.

#### Scenario: Federation disabled by default
- **WHEN** a new Forum channel is created
- **THEN** the `federation_enabled` field is set to `false` and the forum does not produce or accept ActivityPub activities

#### Scenario: Federation enabled on existing forum
- **WHEN** a server administrator sets `federation_enabled` to `true` on a Forum channel
- **THEN** new threads and replies in that forum are delivered to subscribed remote instances and the forum begins accepting inbound Follow requests

#### Scenario: Federation disabled on active forum
- **WHEN** a server administrator sets `federation_enabled` to `false` on a Forum channel that was previously federated
- **THEN** the forum immediately stops producing outbound ActivityPub activities, stops accepting inbound activities, and existing remote followers receive no further updates

### Requirement: Federated Content Is Plaintext
All content federated via ActivityPub MUST be transmitted as plaintext. Federated forum posts SHALL NOT be encrypted with MLS or any other E2EE mechanism. The system MUST clearly distinguish federated forums from E2EE channels in the UI.

#### Scenario: Federated post is plaintext in transit
- **WHEN** a thread is posted in a federated Forum channel
- **THEN** the ActivityPub Create activity contains the post body as plaintext (rendered Markdown or sanitized HTML) and no MLS ciphertext

#### Scenario: E2EE channels remain encrypted
- **WHEN** a message is sent in a non-federated text channel with MLS E2EE enabled
- **THEN** the message is encrypted via MLS and is never exposed as plaintext to any federation process

### Requirement: WebFinger Discovery for Forum Actors
Each federated Forum channel SHALL be discoverable via WebFinger (RFC 7033). The WebFinger response MUST resolve the forum's actor URI using the `acct:` URI scheme and return a JRD document containing a link to the forum's ActivityPub actor endpoint with `rel="self"` and `type="application/activity+json"`.

#### Scenario: WebFinger resolves forum actor
- **WHEN** a remote instance queries `/.well-known/webfinger?resource=acct:forum-name@sombra.example.com`
- **THEN** the server responds with a JRD document containing a link with `rel="self"`, `type="application/activity+json"`, and the `href` pointing to the forum's ActivityPub actor endpoint

#### Scenario: WebFinger returns 404 for non-federated forum
- **WHEN** a remote instance queries WebFinger for a Forum channel that has `federation_enabled` set to `false`
- **THEN** the server responds with a 404 Not Found status

#### Scenario: WebFinger returns 404 for non-forum resources
- **WHEN** a remote instance queries WebFinger for a text channel, voice channel, or user account
- **THEN** the server responds with a 404 Not Found status because only Forum channels are federated actors

### Requirement: HTTP Signatures on All Outbound Deliveries
All outbound ActivityPub HTTP deliveries MUST be signed using HTTP Signatures (draft-cavage-http-signatures). Each federated Forum channel SHALL have its own RSA keypair. The public key MUST be published in the forum's ActivityPub actor document. The server MUST NOT deliver any unsigned ActivityPub requests.

#### Scenario: Outbound delivery is signed
- **WHEN** the server delivers a Create activity to a remote instance's inbox
- **THEN** the HTTP request includes a `Signature` header computed using the forum's RSA private key, covering at minimum the `(request-target)`, `host`, `date`, and `digest` headers

#### Scenario: Forum actor publishes public key
- **WHEN** a remote instance fetches a federated forum's actor document
- **THEN** the actor document includes a `publicKey` object containing the forum's RSA public key in PEM format, the key ID, and the actor's owner URI

#### Scenario: RSA keypair generated on federation enable
- **WHEN** `federation_enabled` is set to `true` on a Forum channel that has no existing keypair
- **THEN** the server generates a new RSA keypair for the forum and stores it in the database

### Requirement: Lemmy-Compatible JSON-LD Context and Extensions
All outbound ActivityPub documents MUST use a JSON-LD context that includes the standard ActivityStreams context and Lemmy-compatible extensions. The server MUST include the `audience`, `stickied`, and `commentsEnabled` properties on applicable objects to ensure interoperability with Lemmy instances.

#### Scenario: JSON-LD context includes Lemmy extensions
- **WHEN** the server serializes an ActivityPub object for outbound delivery
- **THEN** the `@context` array includes `https://www.w3.org/ns/activitystreams` and the Lemmy extension context that defines `stickied`, `commentsEnabled`, and other Lemmy-specific properties

#### Scenario: Thread includes Lemmy-compatible properties
- **WHEN** a thread is posted in a federated Forum channel
- **THEN** the resulting Page object includes `audience` (pointing to the forum Group actor), `commentsEnabled` (boolean), and `stickied` (boolean) properties

#### Scenario: Inbound Lemmy Page is accepted
- **WHEN** a remote Lemmy instance sends a Create activity with a Page object containing Lemmy-specific extensions
- **THEN** the server accepts the activity and correctly maps `stickied`, `commentsEnabled`, and `audience` to local thread properties

### Requirement: ActivityPub Type Mapping
The system SHALL map Sombra domain objects to ActivityPub types as follows: Forum to Group, Thread to Page, Reply to Note, Upvote to Like, Downvote to Dislike, Follow to Follow (with Accept response), and Lock to Lock. All inbound activities using these types MUST be correctly mapped back to local domain objects.

#### Scenario: Forum actor has Group type
- **WHEN** a remote instance fetches a federated forum's actor document
- **THEN** the actor document has `"type": "Group"`

#### Scenario: Thread is serialized as Page
- **WHEN** a new thread is created in a federated forum
- **THEN** the outbound Create activity wraps an object with `"type": "Page"` containing the thread title, body, and attribution

#### Scenario: Reply is serialized as Note
- **WHEN** a user replies to a thread in a federated forum
- **THEN** the outbound Create activity wraps an object with `"type": "Note"` with `inReplyTo` pointing to the parent Page or Note

#### Scenario: Upvote is serialized as Like
- **WHEN** a user upvotes a thread or reply in a federated forum
- **THEN** the outbound activity has `"type": "Like"` with the `object` pointing to the upvoted Page or Note

#### Scenario: Downvote is serialized as Dislike
- **WHEN** a user downvotes a thread or reply in a federated forum
- **THEN** the outbound activity has `"type": "Dislike"` with the `object` pointing to the downvoted Page or Note

#### Scenario: Follow request triggers Accept
- **WHEN** a remote actor sends a Follow activity targeting a federated forum
- **THEN** the server responds with an Accept activity and adds the remote actor to the forum's followers collection

#### Scenario: Lock activity locks a thread
- **WHEN** a moderator locks a thread in a federated forum
- **THEN** the server sends a Lock activity referencing the Page object, and remote instances receiving it disable commenting on that thread

### Requirement: Transactional Outbox with Background Delivery
When a post or reply is created in a federated forum, the system MUST write both the local database record and the corresponding ActivityPub outbox event within the same SQL transaction. A background goroutine SHALL process outbox events and deliver them to remote inboxes. Failed deliveries MUST be retried with exponential backoff, with a maximum retry window of 3 days.

#### Scenario: Post and outbox event written atomically
- **WHEN** a user creates a thread in a federated forum
- **THEN** the thread row and the ActivityPub outbox event row are inserted in the same SQL transaction, ensuring both succeed or both fail

#### Scenario: Background goroutine delivers outbox events
- **WHEN** an ActivityPub outbox event is written to the database
- **THEN** a background goroutine picks up the event and delivers it to each remote follower's inbox via HTTP POST

#### Scenario: Failed delivery retried with exponential backoff
- **WHEN** a delivery to a remote inbox fails (network error or 5xx response)
- **THEN** the system retries the delivery with exponential backoff (e.g., 1 minute, 5 minutes, 30 minutes, 2 hours, 12 hours)

#### Scenario: Delivery abandoned after 3 days
- **WHEN** a delivery to a remote inbox has failed continuously for 3 days
- **THEN** the system marks the delivery as permanently failed and stops retrying

#### Scenario: Transaction rollback prevents orphaned outbox event
- **WHEN** a thread creation fails (e.g., database constraint violation)
- **THEN** both the thread row and the outbox event row are rolled back and no federation delivery is attempted

### Requirement: Inbound Content Sanitization
All inbound ActivityPub HTML content MUST be sanitized through a strict allowlist before storage or rendering. The allowed HTML elements SHALL be limited to: `p`, `a`, `strong`, `em`, `code`, `pre`, `blockquote`, `ul`, `ol`, `li`, `img`. All other HTML elements and attributes not on the allowlist MUST be stripped. The SolidJS client MUST render sanitized content through a Markdown AST parser and SHALL NOT use `innerHTML` or `dangerouslySetInnerHTML`.

#### Scenario: Allowed HTML elements are preserved
- **WHEN** an inbound ActivityPub Note contains HTML with `<p>`, `<strong>`, `<a href="...">`, and `<code>` elements
- **THEN** the sanitizer preserves those elements and their permitted attributes (e.g., `href` on `<a>`, `src` on `<img>`) and stores the sanitized HTML

#### Scenario: Disallowed HTML elements are stripped
- **WHEN** an inbound ActivityPub Note contains `<script>`, `<iframe>`, `<style>`, `<form>`, or `<div>` elements
- **THEN** the sanitizer removes those elements entirely and stores only the text content or allowed child elements

#### Scenario: Disallowed attributes are stripped
- **WHEN** an inbound ActivityPub element contains `onclick`, `onerror`, `style`, or other non-allowlisted attributes
- **THEN** the sanitizer removes those attributes while preserving the element if it is on the allowlist

#### Scenario: Client renders via Markdown AST, never innerHTML
- **WHEN** the SolidJS client renders a federated forum post
- **THEN** the content is parsed through a Markdown AST parser and rendered using SolidJS components, with no use of `innerHTML` or `dangerouslySetInnerHTML` at any point in the rendering pipeline

### Requirement: Public Blob Bucket for Federated Media
Federated forum attachments (images, files) MUST be stored in a public blob bucket that is accessible without authentication. These federated media files SHALL NOT be encrypted, as they must be retrievable by remote ActivityPub instances. The system MUST use the public bucket exclusively for federated forum media and SHALL NOT store non-federated content in the public bucket.

#### Scenario: Federated image stored in public bucket
- **WHEN** a user attaches an image to a thread in a federated forum
- **THEN** the image is stored in the public blob bucket without encryption and the ActivityPub object references the publicly accessible URL

#### Scenario: Remote instance retrieves federated media
- **WHEN** a remote ActivityPub instance fetches a media URL from a federated forum post
- **THEN** the media is served from the public blob bucket without requiring authentication or decryption

#### Scenario: Non-federated media not stored in public bucket
- **WHEN** a user attaches a file in a non-federated text channel
- **THEN** the file is encrypted and stored in the private blob storage, not the public bucket

#### Scenario: Federation disabled moves no existing media
- **WHEN** `federation_enabled` is set to `false` on a Forum channel that previously had federated media
- **THEN** existing media in the public bucket remains accessible (URLs already shared with remote instances) but new attachments in that forum are stored in the private encrypted bucket
