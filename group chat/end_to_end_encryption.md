# End To End Encryption
## Problem Statement
We want to ensure that:
* Messages sent between users cannot be read by anyone except the intended recipient(s), including WhatsApp itself.
* We support features like group messaging, media sharing, and message delivery acknowledgments.
* The system remains scalable for billions of users.

## High-Level Requirements
### Functional:
* One-to-one chat encryption.
* Group chat encryption.
* Media and file sharing.
* Message delivery acknowledgment.
### Non-functional:
* Low latency (near real-time messaging).
* Scalability (millions of concurrent users).
* Forward secrecy (compromise of one key should not affect past messages).
* Device flexibility (multi-device support).

## Architecture
[Whatsapp End To End Encryption](../assets/whatsapp_end_to_end.png)
## Key Concepts / Protocols
WhatsApp uses the Signal Protocol, which combines:
* Double Ratchet Algorithm – ensures forward secrecy and post-compromise security.
* X3DH (Extended Triple Diffie-Hellman) – for initial key agreement.
* Prekeys – to allow asynchronous messaging.

## User Identity and Key Setup
Each user device has:
* Identity Key Pair (IK): long-term key.
* Signed Prekeys (SPK): medium-term key.
* One-time Prekeys (OPK): ephemeral keys to bootstrap new sessions.

### When User A wants to send a message to User B:
* User B uploads SPK + OPK to the WhatsApp server.
* User A fetches B’s prekeys from the server.
* User A uses X3DH to compute a shared secret without B being online.
> Design Insight: The server never sees private keys, only facilitates key exchange.

## Message Encryption Flow (One-to-One)
Session Setup:
```
Shared Secret = X3DH(IK_A, SPK_B, OPK_B)
```
**Message Encryption:**
* Each message uses a session key derived from the shared secret.
* Uses AES-256 in CBC or GCM mode for encryption.
* Each message gets a unique ratchet key, which advances with every message (Double Ratchet).
**Message Delivery:**
* Encrypted message + MAC is sent to WhatsApp servers.
* Server forwards it to the recipient.
* Only recipient can decrypt using their private keys.

## Double Ratchet Algorithm
The ratchet ensures:
* **Forward secrecy**: If a session key is compromised, previous messages are safe.
* **Post-compromise security**: If a current key is compromised, future keys are secure.
* **Works by combining**:
    * Diffie-Hellman ratchet – changes the shared secret each time a DH exchange occurs.
    * Symmetric ratchet – derives message keys from the shared secret.

## Group Messaging
* Each group has a group session key.
* The sender encrypts the group key separately for each participant using their public key.
* Messages are encrypted with the group session key.

> Design Note: WhatsApp uses a Sender Key system for efficiency — each sender encrypts the key once per user.

## Media Encryption
* Media (images, videos) is encrypted separately using a random AES key.
* The AES key is encrypted with the session key and sent along with the media.
* This ensures large files don’t require ratcheting the main session.

## Server Role
* Stores messages temporarily until delivered.
* Never has access to plaintext messages.
* Handles:
    * Offline message delivery.
    * Key distribution.
    * Message ordering and retries.

## Key Security Considerations
* Trust on first use (TOFU): The first time you contact someone, you trust their public key. Later changes trigger a warning.
* Device binding: Keys are per device; adding new devices requires secure key sync.
* Replay protection: Each message includes unique identifiers.
* Integrity: Messages include MACs to detect tampering.
