# ADR 0005: Phase-1 sealed box → Phase-2 Signal, per device pair

- Status: accepted
- Date: 2026-09-29

## Context

The product shipped on a Phase-1 sealed box: ECDH ephemeral key + HKDF +
AES-256-GCM, one box per recipient device per message. No ratchet, no
forward secrecy. Phase 2 is the Signal Double Ratchet
([signal-integration.md](../signal-integration.md),
[signal-migration.md](../../apps/backend/docs/signal-migration.md)).

Clients upgrade at different times. A conversation will contain both
protocols until the last device moves.

## Decision

- **History is never re-encrypted.** `message_envelopes.protocol`
  (`sealed_box` | `signal` | `mls`, default `sealed_box`) records what
  actually produced that ciphertext. Old rows keep decrypting on the
  Phase-1 path.
- **Negotiation is per device pair**, not per conversation.
  `selectProtocol` picks the strongest protocol both devices advertise
  in `devices.capabilities`. A Signal-capable pair is not held back
  because another member is still on sealed box.
- A sender claiming a weaker protocol than both sides support is
  refused (stops a client from pinning a peer on sealed box forever).

The web Signal adapter (`signalClient.ts`) still throws if invoked; the
migration path is accepted in the data model and not fully activated in
the running client.

## Alternatives rejected

**Conversation-wide readiness gate** ("everyone must advertise Signal
before anyone uses it"). Rejected: it delays forward secrecy for every
pair until the slowest member upgrades, which in a large group may be
never.

**Re-encrypt existing history on cutover.** Rejected: the server has no
plaintext, and rewriting history would break devices that only know
sealed box. The `protocol` column exists so history stays labelled.

**Inferring the construction from ciphertext bytes or send time.**
Rejected: capabilities change; last month's envelope is not "whatever
this device supports now".

## Consequences

Capability advertisement and the per-envelope `protocol` column are both
required. Rollout can be incremental. Pairwise Signal and group MLS
(ADR 0006) are different constructions and must not be mixed on one
message (ADR 0004).
