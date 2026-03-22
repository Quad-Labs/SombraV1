## ADDED Requirements

### Requirement: StorageProvider Interface
The blob storage subsystem SHALL define a `StorageProvider` Go interface that abstracts all blob read/write operations. The system MUST provide two implementations: a filesystem-backed `FileSystemStorage` for Tier 1 deployments and an S3-compatible `S3Storage` for Tier 2+ deployments. All consumers of blob storage MUST depend only on the `StorageProvider` interface, never on a concrete implementation.

#### Scenario: StorageProvider interface is implementation-agnostic
- **WHEN** a handler calls `StorageProvider.Put(ctx, key, reader)` to store a blob
- **THEN** the blob is persisted to the active backend (filesystem or S3) without the handler knowing which implementation is in use

#### Scenario: Correct implementation is selected by configuration
- **WHEN** the server starts with `STORAGE_TYPE=s3` and valid S3 credentials in environment variables
- **THEN** the `S3Storage` implementation is instantiated and injected as the `StorageProvider`

#### Scenario: Default implementation is filesystem
- **WHEN** the server starts with no `STORAGE_TYPE` environment variable set
- **THEN** the `FileSystemStorage` implementation is used as the default `StorageProvider`

### Requirement: Filesystem Storage for Tier 1
The `FileSystemStorage` implementation SHALL store all blobs as files within the `sombra-data/` directory relative to the server's working directory. The directory structure MUST be created automatically on first use. Files SHALL be organized by a deterministic path derived from the blob key to avoid excessive files in a single directory.

#### Scenario: sombra-data directory is created on first upload
- **WHEN** the first blob is uploaded and the `sombra-data/` directory does not exist
- **THEN** the `FileSystemStorage` implementation creates the `sombra-data/` directory and any required subdirectories before writing the blob

#### Scenario: Blobs are retrievable after storage
- **WHEN** a blob is stored via `FileSystemStorage` with a given key
- **THEN** a subsequent `Get(ctx, key)` call returns an `io.ReadCloser` streaming the same bytes that were originally stored

### Requirement: S3-Compatible Storage for Tier 2+
The `S3Storage` implementation SHALL connect to any S3-compatible object storage endpoint (AWS S3, MinIO, Backblaze B2, Cloudflare R2). The endpoint URL, region, bucket name, access key, and secret key MUST be configurable via environment variables (`S3_ENDPOINT`, `S3_REGION`, `S3_BUCKET`, `S3_ACCESS_KEY`, `S3_SECRET_KEY`).

#### Scenario: S3 storage connects to a configured endpoint
- **WHEN** the server starts with `STORAGE_TYPE=s3`, `S3_ENDPOINT=https://s3.example.com`, and valid credentials
- **THEN** the `S3Storage` implementation establishes a connection to the specified endpoint and uses the configured bucket for all blob operations

#### Scenario: S3 storage works with MinIO
- **WHEN** the `S3_ENDPOINT` is set to a MinIO instance URL with path-style addressing
- **THEN** the `S3Storage` implementation successfully stores and retrieves blobs using the MinIO-compatible API

### Requirement: Streaming Chunk Encryption
Large files MUST be encrypted and uploaded in streaming 5MB chunks. The system SHALL NOT load an entire file into memory at any point during encryption or upload. The encryption pipeline MUST accept an `io.Reader` as input and produce an `io.Reader` as output, processing data in fixed-size chunks as it flows through.

#### Scenario: Large file encryption stays within bounded memory
- **WHEN** a 500MB file is uploaded for encrypted storage
- **THEN** the server encrypts and stores the file using at most two 5MB buffers in memory at any time (one for reading, one for writing), never loading the full 500MB into RAM

#### Scenario: Each chunk is independently encrypted
- **WHEN** a file is encrypted in streaming mode
- **THEN** each 5MB chunk is encrypted independently with its own nonce/IV derived from the chunk index, allowing random-access decryption of individual chunks without decrypting the entire file

#### Scenario: Final chunk handles remainder
- **WHEN** a file's size is not a multiple of 5MB
- **THEN** the final chunk contains the remaining bytes (less than 5MB), is encrypted with the correct chunk index nonce, and is stored alongside the full-size chunks

### Requirement: EncryptStream Interface Semantics
The encryption interface MUST use `EncryptStream(reader io.Reader) io.Reader` semantics. The system SHALL NOT provide or use an `Encrypt([]byte) []byte` function for blob encryption. All blob encryption and decryption MUST operate on streams, not on fully materialized byte slices.

#### Scenario: EncryptStream returns a streaming reader
- **WHEN** a caller invokes `EncryptStream(plaintextReader)` with an `io.Reader` producing plaintext bytes
- **THEN** the returned `io.Reader` produces encrypted ciphertext bytes on demand as the caller reads from it, without buffering the entire plaintext or ciphertext in memory

#### Scenario: DecryptStream returns a streaming reader
- **WHEN** a caller invokes `DecryptStream(ciphertextReader)` with an `io.Reader` producing encrypted bytes
- **THEN** the returned `io.Reader` produces decrypted plaintext bytes on demand as the caller reads from it

#### Scenario: Byte-slice encryption is not available
- **WHEN** a developer attempts to encrypt a blob using a synchronous `Encrypt([]byte) []byte` signature
- **THEN** no such function exists in the blob storage package; the only available encryption interface is the streaming `EncryptStream`/`DecryptStream` pair

### Requirement: Encrypted and Public Buckets
The `StorageProvider` MUST support two distinct storage buckets: an encrypted bucket for E2EE channel attachments and a public bucket for federated forum media. The encrypted bucket SHALL store opaque ciphertext blobs. The public bucket SHALL serve media with publicly accessible URLs suitable for inclusion in ActivityPub objects.

#### Scenario: E2EE channel attachment is stored in encrypted bucket
- **WHEN** a user uploads a file attachment in an E2EE channel
- **THEN** the file is encrypted via `EncryptStream` and stored in the encrypted bucket, and the resulting blob key references the encrypted bucket

#### Scenario: Forum media is stored in public bucket
- **WHEN** a user uploads an image in a federated forum channel
- **THEN** the image is stored unencrypted in the public bucket and the resulting URL is publicly accessible without authentication

#### Scenario: Public bucket URLs are usable in ActivityPub
- **WHEN** a forum post with an image is federated via ActivityPub
- **THEN** the image URL in the ActivityPub object points to the public bucket and remote instances can fetch the image without Sombra authentication

### Requirement: Blob Upload and Download Endpoints
The server SHALL expose `POST /api/v1/blob/upload` for uploading blobs and `GET /api/v1/blob/:id` for downloading blobs. The upload endpoint MUST accept multipart/form-data with the file payload and return a blob ID. The download endpoint MUST stream the blob bytes to the client.

#### Scenario: Successful blob upload
- **WHEN** an authenticated client sends a `POST /api/v1/blob/upload` request with a multipart/form-data body containing a file
- **THEN** the server stores the blob via the `StorageProvider`, persists metadata in the relational database, and responds with a 201 Created status and a JSON body containing the blob ID

#### Scenario: Successful blob download
- **WHEN** an authenticated client sends a `GET /api/v1/blob/:id` request with a valid blob ID
- **THEN** the server retrieves the blob from the `StorageProvider` and streams the bytes to the client with the appropriate `Content-Type` header

#### Scenario: Download of nonexistent blob returns 404
- **WHEN** a client sends a `GET /api/v1/blob/:id` request with an ID that does not exist
- **THEN** the server responds with a 404 Not Found status

#### Scenario: Upload requires authentication
- **WHEN** an unauthenticated client sends a `POST /api/v1/blob/upload` request
- **THEN** the server responds with a 401 Unauthorized status

### Requirement: Attachment Metadata and Key Separation
Attachment metadata (file name, size, MIME type, blob key, uploader, channel association) SHALL be stored in the relational database. The encrypted blob bytes SHALL be stored in the blob storage backend. The decryption key for encrypted attachments MUST be included in the MLS message ciphertext, not stored on the server. The server SHALL NOT have access to decryption keys for E2EE attachments at any point.

#### Scenario: Metadata is stored in relational database
- **WHEN** a blob is uploaded as a channel attachment
- **THEN** the relational database contains a record with the file name, size, MIME type, blob storage key, uploader user ID, and associated channel ID

#### Scenario: Decryption key is in MLS ciphertext only
- **WHEN** a user sends a message with an encrypted attachment in an E2EE channel
- **THEN** the decryption key for the attachment blob is embedded in the MLS message ciphertext, and the server's attachment metadata record contains no decryption key or any material from which the key could be derived

#### Scenario: Server cannot decrypt E2EE attachments
- **WHEN** the server processes or stores an encrypted attachment blob
- **THEN** the server handles only opaque ciphertext bytes and metadata; at no point does the server possess the symmetric key needed to decrypt the attachment content
