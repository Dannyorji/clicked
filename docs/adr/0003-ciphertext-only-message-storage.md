# ADR 0003: Ciphertext-only message storage

- Status: accepted
- Date: 2026-09-29

## Context

`messages` once had a plaintext `content` column (and a GIN index for
server-side search) beside the E2EE work. The server cannot search
ciphertext it cannot read. The cutover is recorded in
[message-encryption-migration.md](../../apps/backend/docs/message-encryption-migration.md).

## Decision

Live `messages` rows store `ciphertext` (and optional per-device
envelopes). There is no plaintext `content` column. Pre-cutover plaintext
was copied to `message_content_archive` and then purged from `messages`
(archive-then-purge). `GET /conversations/:id/search` returns `410 Gone`.
Search is client-side over decrypted history.

System rows use `systemPayload` JSON, never ciphertext; a check constraint
keeps the two from mixing.

## Alternatives rejected

**In-place tombstone** (keep a nulled `content` / `legacyPlaintext` column
forever). Rejected because every future query and migration would carry a
legacy-shaped column for a closed set of old rows, and a future path could
accidentally serve that field as if it were ciphertext.

**Re-encrypting old plaintext to look like E2EE.** Rejected: archived rows
with no ciphertext surface as `{ unavailable: true }`. The product does
not backdate an "encrypted" claim.

**Server-side search over ciphertext.** Not implemented; there is no
searchable-encryption scheme in this tree. The 410 is the recorded
refusal.

## Consequences

Hot-path serializers only have a ciphertext (or envelope, or unavailable)
branch. The archive table has no API. Legal/compliance access to old
plaintext is out of band.

See also ADR 0004 (how ciphertext is keyed per device) and ADR 0006
(group ciphertext via MLS).
