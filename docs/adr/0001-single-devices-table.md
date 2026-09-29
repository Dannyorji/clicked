# ADR 0001: One canonical `devices` table

- Status: accepted
- Date: 2026-09-29

## Context

Auth, prekeys, realtime delivery, push, and MLS all need a stable device
identity. An earlier split stored some of that on a separate
`user_devices` table. Two registries can silently disagree about whether
a device exists, is revoked, or owns a given identity key.

`apps/backend/src/db/schema.ts` and `routes/devices.ts` record the merge:
`devices` is the single canonical device registry. `messages.senderDeviceId`,
`messageEnvelopes.recipientDeviceId`, and `pushSubscriptions.deviceId` all
FK here. `revokedAt` keeps the row for audit instead of deleting it.

## Decision

One `devices` table per physical client. Identity public key, platform,
`lastSeenAt`, `pushEnabled`, `capabilities`, and revocation live on that
row. `GET /user-devices/:id/public-key` remains as a lookup alias; it is
not a second store (`routes/userDevices.ts`).

## Alternatives rejected

**A separate `user_devices` table** (auth/prekeys vs realtime/push).
Rejected because the two copies drifted: a device could be live for
messaging and missing for prekeys, or revoked in one table and not the
other. Issue #103 / #107 and the header on `routes/devices.ts` are the
review record of that merge.

**Deleting revoked rows.** Rejected so fingerprint history, envelope
foreign keys, and "when was this revoked" stay queryable. `revokedAt` plus
the GC `staleFlaggedAt` flag cover retirement without a hard delete.

## Consequences

Every device-shaped feature adds columns or child tables that FK
`devices`, not a parallel registry. Linking a new device and revoking an
old one are the same row lifecycle. Clients must treat `/user-devices` as
a compatibility path only.

See [api-devices.md](../../apps/backend/docs/api-devices.md).
