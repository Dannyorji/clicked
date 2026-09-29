# ADR 0006: MLS for groups

- Status: accepted
- Date: 2026-09-29

## Context

Per-device envelopes (ADR 0004) scale linearly with members × devices.
Group file sharing and group text need one ciphertext the epoch's
members can open, without giving the server the key. The chosen protocol
is MLS (RFC 9420) via `@openmls/wasm` on the client; the backend stores
public group state only
([mls-group-membership.md](../../apps/backend/docs/mls-group-membership.md),
[mls-integration.md](../../apps/web/src/lib/mls-integration.md)).

## Decision

Group conversations use MLS. The server holds `mls_groups`,
`mls_group_members` (membership as an epoch interval), `mls_commits`,
`mls_welcomes`, and `mls_key_packages`. `messages.mls_epoch` names the
epoch that encrypted `ciphertext`. A newly joined device cannot decrypt
pre-join epochs; those messages render as unavailable, not as errors.

File keys for group files ride inside that single MLS message so object
storage stores one object, not one copy per device
([mls-group-files.md](../../apps/backend/docs/mls-group-files.md)).

## Alternatives rejected

**Per-device envelope fan-out for group content.** Rejected on cost and
churn: "in a 30-person group with 3 devices each that is 90 copies of
the same bytes, and it grows again on every new device."

**Sender keys / a static group key on the server.** Rejected: the server
would be in the key-distribution path, and removal would not have MLS's
epoch forward secrecy. Not implemented; no extra rationale is recorded
beyond the MLS membership doc.

**Re-encrypting history to a joining device.** Deferred (would need an
existing member to resend plaintext). Until that exists, "no history for
a new device" is the behaviour.

**Mixing `envelopes` and `mlsEpoch` on one message.** Rejected in
`validateMessagePayload` so recipients have one rule for which bytes to
decrypt.

## Consequences

DMs stay on sealed box / Signal (ADR 0004, ADR 0005). Groups accept
weaker history for new devices in exchange for one ciphertext and
epoch-based membership. The backend remains an untrusted relay.
