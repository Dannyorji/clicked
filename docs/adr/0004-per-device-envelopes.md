# ADR 0004: Per-device envelopes over a shared ciphertext

- Status: accepted
- Date: 2026-09-29

## Context

A user has many devices. Each device has its own identity key. A message
the server stores as one blob encrypted to a conversation-wide secret
would let any device (or a leak of that secret) read everything, and
would not match the Phase-1 sealed-box model (one independent box per
recipient device).

Fan-out helpers in `lib/messageFanout.ts` require one
`message_envelopes` row per recipient device in the same transaction as
the `messages` insert.

## Decision

DM / sealed-box / Signal sends carry **per-device envelopes**:
`recipientDeviceId` + `ciphertext` (+ `protocol`). Sibling-device
coverage is checked on send (`fetchSiblingDeviceIds`). The conversation
room may get a ciphertext-free `new_message` for UI; the decryptable
bytes go to `device:${deviceId}`.

MLS group messages are the documented exception: one group ciphertext
on `messages`, no per-device envelopes (ADR 0006). Mixing `envelopes`
with `mlsEpoch` is rejected in `validateMessagePayload`.

## Alternatives rejected

**One shared conversation ciphertext (static group key).** Rejected for
1:1 and multi-device DMs: every device would share a decrypt key, add
device / revoke device would require rotating that key for all history
or leaving old devices able to read new mail, and it collapses to a
server-visible or single-leak plaintext if the key is ever exposed. The
repo does not record a long design thread against a static shared key
beyond the envelope requirement itself; the implemented model is
per-device boxes.

**Encrypting only to the recipient user, not each of their devices.**
Rejected: a newly linked laptop would not be able to decrypt a message
already stored for the phone. Sibling envelopes exist so every active
device of the sender (and each recipient) can open the message locally
(#188).

## Consequences

Send payload size grows with the device set. A missing sibling envelope
fails the send rather than silently dropping a device. Group chats that
need one copy of the bytes use MLS instead of multiplying envelopes
(ADR 0006).
