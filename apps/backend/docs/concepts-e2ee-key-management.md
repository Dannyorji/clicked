# E2EE device identity & key lifecycle (backend)

This document explains the backend's role in the end-to-end-encryption key
lifecycle: how a device gets an identity, how it publishes key material,
how other clients fetch and consume that material, and how a device is
retired. It covers the full lifecycle as one coherent story and names the
exact routes and tables involved at each stage.

**Scope.** This is a backend concepts document. It does not describe any
client-side cryptography (X3DH session establishment, Double Ratchet, MLS
epoch secrets) — only what the server stores, serves, validates, and
deletes along the way. See
[`../../apps/web/docs/concepts-e2ee-architecture.md`](../../apps/web/docs/concepts-e2ee-architecture.md)
for the client-side key model.

## What the server can and cannot see

Stated explicitly, because every section below depends on it:

- **The server never sees a private key, of any kind.** Not an identity
  private key, not a signed-prekey private key, not a one-time-prekey
  private key, not an MLS HPKE init private key, not a session/ratchet key.
  Every key-shaped column in `devices` and `device_prekeys` (and
  `mls_key_packages`) stores **public** material only —
  `identityPublicKey`, `publicKey` (signed and one-time), `keyPackage`. This
  is a property of what the schema is defined to hold, not an access-control
  claim about what a query *could* return if misused.
- **The server verifies signatures without needing the signer's private
  key** — Ed25519 signature verification is a public-key operation. The
  signed-prekey upload route (`POST /devices/:id/prekeys`) verifies that
  `signedPreKey.signature` is a valid signature over `signedPreKey.publicKey`
  made by the device's own `identityPublicKey`, entirely with public data.
  This proves the signed prekey really was published by the device that owns
  the identity key, without the server ever holding a signing key of its own
  for that device.
- **The server cannot decrypt message ciphertext.** `messages.ciphertext`
  and `message_envelopes.ciphertext` are opaque blobs to every backend code
  path — nothing in `routes/` or `services/` ever attempts to decrypt them.
  (See [`contracts-database-schema.md`](contracts-database-schema.md#messages)
  for the column, and
  [`../../apps/web/docs/concepts-e2ee-architecture.md`](../../apps/web/docs/concepts-e2ee-architecture.md)
  for where decryption actually happens — exclusively on-device.)
- **What the server *can* see and act on**: which device belongs to which
  user, when a device was last active, how many one-time prekeys a device
  has left, which device fetched which other device's bundle and when
  (`key_bundle_drained` audit events — see
  [Audit trail](#audit-trail-what-gets-logged-and-why) below), and the
  device capability set a device advertises (which protocols/ciphersuites it
  supports — see
  [`concepts-protocol-negotiation.md`](concepts-protocol-negotiation.md)).
  All of this is metadata about key *distribution*, never about message
  *content*.

## The lifecycle, stage by stage

### 1. Device identity & registration

A device's identity is its Ed25519 `identityPublicKey`, stored once per
device in `devices.identity_public_key`. There are **two independent code
paths** that can create or resolve a `devices` row for an existing account,
and they behave differently on a revoked identity key — this distinction
matters and is easy to miss:

| | `POST /auth/challenge` → `POST /auth/verify` (`routes/auth.ts`) | `POST /devices/link/challenge` → `POST /devices/link/verify` (`routes/devices.ts`) |
|---|---|---|
| Purpose | Sign in (first device, or any later device, via wallet proof) | Explicitly link an *additional* device to an already-authenticated account |
| Caller authentication | None required beforehand — this *is* the login step | Requires a valid JWT already (the caller must already be signed in on some device) |
| Proof required | Wallet signature over the sign-in challenge message | A **second, distinct** wallet signature over a device-link-specific message (`"Link device to Clicked\nUser: ...\nNonce: ..."`) — deliberately different wording from the login message so one signature can never be replayed as the other (see [`concepts-nonce-lifecycle.md`](concepts-nonce-lifecycle.md)) |
| Lookup key | `(userId, identityPublicKey)` | `(userId, identityPublicKey)` |
| New identity key, no existing row | Inserts a new `devices` row | Inserts a new `devices` row |
| Existing row, **not** revoked | Reuses it, updates `lastSeenAt`/name/platform/`registrationId`/capabilities | `409 Device already registered for this user` |
| Existing row, **revoked** | **Hard refusal**: `401 Device has been revoked` — sign-in is blocked outright, no reactivation | **Reactivates** the row in place (`revokedAt` cleared, name/platform/`registrationId` updated) — the comment in the source calls this out as a deliberately distinct audit event from a fresh link |
| Audit action recorded | `auth_failed` with `reason: device_revoked` (failure case only; success records nothing here) | `device_linked`, with `metadata.reactivatedRevokedDevice: true/false` |

`POST /devices` itself (the bare, JWT-only registration endpoint) is
**retired** — it now unconditionally returns `403` directing the caller to
the link-challenge flow (tracked in-source as issue #333, closing a gap
where a stolen JWT alone was sufficient to attach an attacker-controlled
device). If you are reading
[`api-devices.md`](api-devices.md) and its `POST /devices` section describes
reactivation-on-registration as a single unauthenticated-by-wallet step,
that section predates #333 and no longer matches `routes/devices.ts` — this
document reflects the current two-step link flow; `api-devices.md` should be
read for the routes it still accurately describes (listing, revocation,
prekey upload).

Either path writes the same columns: `identityPublicKey` (immutable per
row — there is no "update my identity key in place" operation; a client that
rotates its long-term identity key produces a **new** `devices` row, not a
mutated existing one — see [Key rotation](#7-key-rotation-and-the-unwritten-devicekeyhistory-table)
below), plus `deviceName`, `platform`, `registrationId` (the X3DH
registration id from the prekey bundle spec), and `capabilities` (the
protocol/ciphersuite/file-transfer versions the device advertises, defaulted
via `normalizeCapabilities` if omitted — see
[`concepts-protocol-negotiation.md`](concepts-protocol-negotiation.md)).

### 2. Prekey publication

`POST /devices/:id/prekeys` (ownership- and revocation-checked — see
[`api-devices.md`](api-devices.md#post-devicesidprekeys) for the full
request/response contract) writes into `device_prekeys`, discriminated by
`keyType`:

- **Signed prekey** (`key_type = 'signed'`) — exactly one live row per
  device, **upserted** on every upload (`ON CONFLICT` on the partial unique
  index over `deviceId` where `keyType = 'signed'`). The server verifies
  `signature` is a valid Ed25519 signature over `publicKey`, signed by the
  device's `identityPublicKey`, **before** writing anything — an invalid
  signature means nothing is persisted. This is the one point in the whole
  lifecycle where the server actively validates a cryptographic property of
  uploaded key material, rather than just storing it.
- **One-time prekeys** (`key_type = 'one_time'`) — many rows, added to the
  existing pool (`ON CONFLICT ... DO NOTHING` on `(deviceId, keyType,
  keyId)`, so a retried upload of the same `keyId` is a silent no-op, not an
  error). Capped at **200 unconsumed** per device (`OTP_CAP`); a batch that
  would exceed the cap is trimmed, not rejected outright, as long as some
  headroom remains — see [`api-devices.md`](api-devices.md#the-200-key-cap-and-trimming-behavior)
  for the exact trimming arithmetic.

Nothing here is decrypted or re-derived — the server stores the public
key/signature bytes exactly as submitted and never generates key material on
a device's behalf.

### 3. Key-bundle fetch & one-time-prekey consumption

`GET /users/:userId/devices/:deviceId/key-bundle` (`routes/users.ts`) is how
a sender obtains a recipient device's X3DH bundle to start a session. Three
things happen atomically inside one DB transaction:

1. The device's current **signed prekey** is read (required — `409` if the
   device has never uploaded one).
2. **At most one** unconsumed one-time prekey row is selected
   (`FOR UPDATE SKIP LOCKED`, oldest first) and flipped to `consumed = true`
   — flipped, never deleted, so the fetch history stays auditable. Two
   concurrent fetches for the same device can never be handed the same
   one-time prekey, because the row lock excludes it from the other
   transaction's candidate set.
3. The post-consumption remaining count is computed **inside the same
   transaction**, so it can never race a concurrent fetch into reporting a
   stale number.

If no one-time prekey remains, the bundle is still returned — with
`oneTimePreKey: null` — rather than erroring, so the sender can fall back to
a 3-DH session using only the signed prekey. See
[`e2ee-onboarding.md`](e2ee-onboarding.md#c-low-prekey-warning-before-exhaustion)
for the exact JSON shapes of both the normal and exhausted responses, and
for the `prekeys_low` socket event that fires (debounced, once per threshold
crossing) to warn the *owning* device to replenish before it actually runs
dry.

Every successful one-time-prekey consumption also increments a metric
(`prekeyConsumedTotal`) and records a `key_bundle_drained` audit event (see
[Audit trail](#audit-trail-what-gets-logged-and-why)) — the subject is the
device **owner** (whose prekey supply just shrank), the actor is whoever
fetched the bundle.

### 4. Replenishment signal and background pruning

Two independent mechanisms keep the one-time-prekey pool healthy without any
cross-service coordination:

- **Low-supply push**: a `prekeys_low` event to the owning device's socket
  room (`device:{deviceId}`) once a fetch drops the remaining count below a
  threshold (default `20`, `PREKEY_LOW_THRESHOLD`). Re-arms once the device
  tops back up at or above the threshold, so a later crossing warns again.
- **Background GC** (`services/deviceGc.ts`, hourly by default): prunes
  `device_prekeys` rows where `key_type = 'one_time'` and either `consumed =
  true` and older than `PREKEY_CONSUMED_RETENTION_DAYS` (default 30d), or
  unconsumed and older than `PREKEY_UNCONSUMED_MAX_AGE_DAYS` (default 90d).
  Signed prekeys are never touched by this pass — there is exactly one live
  signed prekey per device, replaced in place on upload, not garbage
  collected. See [`concepts-gc-jobs.md`](concepts-gc-jobs.md) for the full
  job reference, including the MLS KeyPackage pass that mirrors this policy.

### 5. Revocation

A device is revoked via `DELETE /devices/:id` (single device, caller-owned
only, idempotent, refuses to revoke the caller's last active device) or
`POST /devices/logout-everywhere` (every other active device on the
account). Both route through one shared helper with **identical** per-device
side effects:

1. `devices.revokedAt` is set (row kept, not deleted — revocation history is
   permanent and queryable).
2. Every row in `device_prekeys` for that device — signed **and** one-time,
   consumed or not — is **deleted outright**. A revoked device has no
   prekeys left to hand out; re-registering the same identity key later (via
   the link-verify reactivation path above) starts prekey-less and must
   re-upload.
3. Live sockets for that device are force-disconnected, same-node
   immediately and cross-node via a `device_revoked:{deviceId}` Redis
   pub/sub publish (best-effort; a publish failure does not fail the HTTP
   request).
4. A `device_revoked` system event is emitted into every conversation the
   user is a member of, so other clients learn "this user's device set
   changed" and know to refresh key bundles before encrypting to them again.

Full request/response detail and the exact ownership/idempotency rules are
in [`api-devices.md`](api-devices.md#revocation-side-effects) — that section
of the doc is still accurate (only the *registration* section is affected by
the #333 change above).

### 6. Post-revocation staleness flagging

`services/deviceGc.ts`'s third pass flags (never deletes) devices that have
been revoked longer than `DEVICE_STALE_AFTER_DAYS` (default 180d), by
setting `devices.staleFlaggedAt`. This is purely informational — it marks a
row eligible for whatever downstream archival process an operator chooses
to run; revocation audit history is preserved indefinitely regardless.

### 7. Key rotation and the unwritten `deviceKeyHistory` table

`device_key_history` exists in the schema specifically to let clients detect
a silent identity-key swap and show a safety-number-style warning: it
records `(deviceId, userId, previousKey, newKey, changeReason, recordedAt)`,
and `GET /users/:id/key-history` (`routes/users.ts`) reads it back, ordered
oldest-first.

**Verified against the current source: nothing writes to this table.**
There is no `db.insert(deviceKeyHistory, ...)` call anywhere in
`apps/backend/src`, and no test exercises one. The schema comment's claim
that it is "written whenever a device's `identityPublicKey` changes" does
not correspond to any implemented write path today — because, per
[stage 1](#1-device-identity--registration), **there is also no operation
that changes an existing device row's `identityPublicKey` in place.** A
client that rotates its long-term identity key does so by registering what
the server sees as a brand-new, unrelated `devices` row (a new `(userId,
identityPublicKey)` pair) — the old row is left exactly as it was, not
updated and not linked to the new one. `GET /users/:id/key-history` is
therefore a read endpoint over a table that the current implementation never
populates; treat it as reading back an empty history today, not as a working
key-transparency feed. Closing this gap would mean either (a) adding an
explicit identity-key-rotation endpoint that updates a row in place and logs
the transition, or (b) deriving history from the existing
add-new-row/revoke-old-row pattern after the fact — neither exists yet.

### MLS key packages — a parallel, separate mechanism

Group conversations run MLS (RFC 9420) instead of per-recipient X3DH
sessions, and MLS has its own public-key-material table,
`mls_key_packages`, populated by `POST /devices/:id/mls-key-packages` under
the same "public material only, idempotent upload, capped pool" shape as
`device_prekeys` — but it is a genuinely separate mechanism with its own
single-use-per-Add semantics, not an alternate code path over the same
table. This document's scope is the `devices`/`device_prekeys` pair
specifically; see [`mls-key-packages.md`](mls-key-packages.md) and
[`mls-group-membership.md`](mls-group-membership.md) for the MLS-side
lifecycle.

## `devices` / `device_prekeys` at a glance

Full column-by-column documentation (types, defaults, indexes, FKs) is in
[`contracts-database-schema.md`](contracts-database-schema.md#devices) — this
is just the subset load-bearing for the lifecycle above:

| Column | Table | Role in the lifecycle |
|---|---|---|
| `identityPublicKey` | `devices` | The device's long-term public identity; immutable per row (see [stage 7](#7-key-rotation-and-the-unwritten-devicekeyhistory-table)); lookup key for both registration paths. |
| `registrationId` | `devices` | X3DH registration id published alongside the bundle. |
| `capabilities` | `devices` | Advertised protocol/ciphersuite support; defaults applied for pre-existing rows. |
| `revokedAt` | `devices` | `NULL` while active; set (never cleared except by link-verify reactivation) on revocation. |
| `staleFlaggedAt` | `devices` | Set by GC once revoked past the retention window; informational only. |
| `keyType` | `device_prekeys` | `'signed'` (one live row, upserted) or `'one_time'` (many rows, pooled and capped). |
| `signature` | `device_prekeys` | Required (DB check constraint) and server-verified only for `keyType = 'signed'`; never set for one-time keys. |
| `consumed` | `device_prekeys` | Flips `true` on bundle-fetch claim; one-time keys only; never un-set. |

## Audit trail: what gets logged, and why

Every stage above that touches a device's security posture writes to
`audit_logs` (see
[`../../../docs/security/audit-logging.md`](../../../docs/security/audit-logging.md)
for the full catalog and what is deliberately excluded from it):

| Action | Fired by | Subject vs. actor |
|---|---|---|
| `device_linked` | `POST /devices/link/verify` | Actor = the authenticated caller; target = the new/reactivated device |
| `device_revoked` | `DELETE /devices/:id`, `logout-everywhere` (once per call, not per device) | Subject = device owner |
| `logout_everywhere` | `POST /devices/logout-everywhere` | Subject = the caller (revoking their own other devices) |
| `auth_failed` (`reason: device_revoked`) | `POST /auth/verify` rejecting a revoked device | Subject = the account being signed into |
| `key_bundle_drained` | `GET /users/:userId/devices/:deviceId/key-bundle` consuming the last (or any) one-time prekey | Subject = bundle/device **owner**; actor = the fetching caller |

These rows never contain key material or message content — only
identifiers, counts, and outcomes, per the append-only audit log's design
constraint.

## Related documents

- [`api-devices.md`](api-devices.md) — full request/response reference for
  listing, revocation, and prekey upload (registration section is
  superseded by [stage 1](#1-device-identity--registration) above).
- [`e2ee-onboarding.md`](e2ee-onboarding.md) — the exact JSON shapes and
  sequencing for first-time onboarding and first-DM bundle fetch, including
  the offline-recipient and prekey-exhausted paths.
- [`contracts-database-schema.md`](contracts-database-schema.md) — full
  column/index/FK reference for `devices`, `device_prekeys`, and every other
  table.
- [`concepts-protocol-negotiation.md`](concepts-protocol-negotiation.md) —
  how `capabilities` drives sender/recipient protocol agreement.
- [`concepts-nonce-lifecycle.md`](concepts-nonce-lifecycle.md) — the
  sign-in and device-link nonce stores referenced in stage 1.
- [`concepts-gc-jobs.md`](concepts-gc-jobs.md) — the full background-job
  reference, including the prekey/MLS-KeyPackage pruning and device
  stale-flagging passes.
- [`mls-key-packages.md`](mls-key-packages.md) — the parallel MLS key
  material lifecycle.
- [`../../../docs/security/audit-logging.md`](../../../docs/security/audit-logging.md) —
  the audit log format and full action catalog.
- [`../../apps/web/docs/concepts-e2ee-architecture.md`](../../apps/web/docs/concepts-e2ee-architecture.md) —
  the client-side key model this document deliberately does not cover.
