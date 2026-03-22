# Capability: sframe-media-encryption

SFrame (RFC 9605) frame-level encryption for voice/video -- symmetric keys derived from MLS group secrets via MLS_Exporter, epoch tagging for key rotation, 10-second grace period for old keys.

## ADDED Requirements

### Requirement: SFrame Encryption of All Media Frames

Every voice and video frame SHALL be encrypted using SFrame (RFC 9605) before being passed to the WebRTC transport layer. The SFrame encryption MUST occur at the application layer, above SRTP, so that intermediaries such as SFUs cannot access plaintext media content.

#### Scenario: Voice frame encrypted before transport
- **WHEN** a client captures a voice frame for transmission
- **THEN** the client SHALL encrypt the frame using SFrame before passing it to the WebRTC transport

#### Scenario: Video frame encrypted before transport
- **WHEN** a client captures a video frame for transmission
- **THEN** the client SHALL encrypt the frame using SFrame before passing it to the WebRTC transport

#### Scenario: Receiver decrypts SFrame payload
- **WHEN** a client receives an SFrame-encrypted media frame from the WebRTC transport
- **THEN** the client SHALL decrypt the SFrame payload using the appropriate epoch key before rendering

---

### Requirement: Symmetric Key Derivation from MLS Group Secrets

The symmetric key used for SFrame encryption SHALL be derived from the MLS group secrets using the `MLS_Exporter` function defined in RFC 9420 Section 8. The exporter label MUST be `"sframe_key"` and the context MUST include the current MLS epoch. This derivation MUST be deterministic so that all group members independently derive the same key.

#### Scenario: All group members derive the same SFrame key
- **WHEN** the MLS group is at a given epoch and each member calls `MLS_Exporter` with label `"sframe_key"` and the current epoch as context
- **THEN** every group member SHALL independently derive the identical symmetric key

#### Scenario: Key derivation uses correct parameters
- **WHEN** a client derives the SFrame encryption key
- **THEN** the client SHALL use `MLS_Exporter` (RFC 9420 Section 8) with the label `"sframe_key"` and the current epoch number as the context parameter

---

### Requirement: Epoch Tagging in SFrame Header

Each SFrame-encrypted frame SHALL include the MLS epoch identifier in the SFrame header as specified in RFC 9605 Section 4.1. Receivers MUST use the epoch tag to select the correct decryption key.

#### Scenario: SFrame header contains epoch tag
- **WHEN** a client encrypts a media frame with SFrame
- **THEN** the SFrame header SHALL contain the current MLS epoch identifier per RFC 9605 Section 4.1

#### Scenario: Receiver selects key by epoch tag
- **WHEN** a receiver processes an incoming SFrame-encrypted frame
- **THEN** the receiver SHALL read the epoch tag from the SFrame header and use the corresponding key for decryption

---

### Requirement: Dual Key Retention During Rotation

During key rotation (triggered by an MLS epoch advance), receivers SHALL hold both the old epoch key and the new epoch key simultaneously. This dual retention MUST ensure that frames encrypted with either the old or new key can be decrypted during the transition period.

#### Scenario: Frames from old epoch decoded during transition
- **WHEN** the MLS epoch advances and a receiver still receives frames tagged with the previous epoch
- **THEN** the receiver SHALL successfully decrypt those frames using the retained old epoch key

#### Scenario: Frames from new epoch decoded during transition
- **WHEN** the MLS epoch advances and a receiver receives frames tagged with the new epoch
- **THEN** the receiver SHALL successfully decrypt those frames using the newly derived epoch key

---

### Requirement: 10-Second Grace Period for Old Keys

After an MLS epoch advance, receivers MUST retain the old epoch key for exactly 10 seconds. After the 10-second grace period expires, the old epoch key MUST be discarded. Frames arriving with a discarded epoch tag SHALL be dropped.

#### Scenario: Old key retained during grace period
- **WHEN** an MLS epoch advance occurs
- **THEN** receivers SHALL retain the previous epoch key for 10 seconds from the moment the new epoch key is derived

#### Scenario: Old key discarded after grace period
- **WHEN** 10 seconds have elapsed since an MLS epoch advance
- **THEN** receivers SHALL discard the old epoch key and MUST NOT use it for decryption

#### Scenario: Late frame with expired epoch dropped
- **WHEN** a frame arrives with an epoch tag whose key has already been discarded (past the 10-second grace period)
- **THEN** the receiver SHALL drop the frame and MUST NOT attempt decryption

---

### Requirement: Key Rotation Triggered by MLS Epoch Advance

SFrame key rotation SHALL be triggered exclusively by MLS epoch advances, which occur when members join or leave the group (or when an Update Commit is processed). When the epoch advances, all group members MUST derive a new SFrame key from the new epoch's group secrets and begin encrypting with the new key immediately.

#### Scenario: Member join triggers key rotation
- **WHEN** a new member joins the voice/video channel and an MLS Commit advances the epoch
- **THEN** all group members SHALL derive a new SFrame key from the new epoch and begin encrypting new frames with it

#### Scenario: Member leave triggers key rotation
- **WHEN** a member leaves the voice/video channel and an MLS Commit advances the epoch
- **THEN** all remaining group members SHALL derive a new SFrame key from the new epoch and begin encrypting new frames with it, ensuring the departed member cannot decrypt future frames

---

### Requirement: SFU Cannot Decrypt SFrame Payload

The mediasoup SFU SHALL strip the SRTP routing layer for forwarding purposes but MUST NOT be able to decrypt the SFrame payload within the RTP frame. The SFU operates on encrypted payloads and has no access to MLS group secrets or SFrame keys.

#### Scenario: SFU forwards encrypted payload without decryption
- **WHEN** a media frame routed through the mediasoup SFU arrives at a receiver
- **THEN** the SFrame-encrypted payload SHALL be intact and unmodified, proving the SFU did not decrypt or alter the content

#### Scenario: SFU has no access to SFrame keys
- **WHEN** the mediasoup SFU processes media frames
- **THEN** the SFU SHALL have no access to MLS group secrets, `MLS_Exporter` outputs, or SFrame symmetric keys

---

### Requirement: SFrame for P2P Calls

Even peer-to-peer (1:1) calls SHALL use SFrame encryption for consistency. The same SFrame encryption pipeline MUST be used regardless of whether media flows through an SFU, a mesh topology, or a direct P2P connection.

#### Scenario: P2P call uses SFrame
- **WHEN** two users establish a direct P2P voice or video call
- **THEN** all media frames SHALL be SFrame-encrypted using keys derived from the MLS group for that call

#### Scenario: Consistent encryption across topologies
- **WHEN** a call transitions from P2P to SFU-routed (e.g., a third participant joins)
- **THEN** the SFrame encryption pipeline SHALL remain the same, with only the MLS epoch advancing to account for the new member
