# Presence, heartbeat & connection liveness

This document explains how the backend knows whether a user is online: the
Redis-backed per-device/per-user presence model in
[`services/presence.ts`](../src/services/presence.ts), the heartbeat watchdog
in [`services/heartbeat.ts`](../src/services/heartbeat.ts), and how the two
are wired together in the Socket.IO connection handler in
[`index.ts`](../src/index.ts). Verified against the current implementation of
all three files.

For the WebSocket event contract (`heartbeat`, `presence_update`,
`user_online`/`user_offline`) from a client's point of view, see
[`api-websocket-events.md`](api-websocket-events.md). For the connection
lifecycle these services plug into, see
[`concepts-gateway-architecture.md`](concepts-gateway-architecture.md). For
the `users`/`devices` table columns referenced throughout (`presenceVisible`,
`lastSeenVisible`, `devices.lastSeenAt`, `devices.revokedAt`), see
[`contracts-database-schema.md`](contracts-database-schema.md).

## Table of contents

- [Two independent signals of "online"](#two-independent-signals-of-online)
- [Redis key structure](#redis-key-structure)
- [The device → user presence model](#the-device--user-presence-model)
- [Heartbeat timing contract](#heartbeat-timing-contract)
- [Connection lifecycle, step by step](#connection-lifecycle-step-by-step)
- [Multi-device aggregation](#multi-device-aggregation)
- [Transition-broadcast behavior](#transition-broadcast-behavior)
- [Privacy gating](#privacy-gating)
- [Known caveats, verified against the current code](#known-caveats-verified-against-the-current-code)

## Two independent signals of "online"

The backend actually has **two** ways to decide if a user is online, and it
prefers the first but falls back to the second:

1. **Redis presence registry** (`services/presence.ts`) — live, socket-driven,
   second-granularity. This is the fast path and the one that drives
   real-time `presence_update`/`user_online`/`user_offline` broadcasts.
2. **Postgres `devices.lastSeenAt`** (`deriveDevicePresence`) — a
   database-backed fallback with a 90-second recency window, used when Redis
   is unreachable, and as the source for the `lastSeen` timestamp shown once
   a user is confirmed offline.

`GET /users/:id/presence` (`src/routes/users.ts`) tries Redis first
(`isOnline`) and only calls `deriveDevicePresence` if Redis says the user is
not online (or Redis is not configured at all) — see
[Known caveats](#known-caveats-verified-against-the-current-code) for what
this ordering does and does not guarantee.

## Redis key structure

Every key below is per-process-shared (visible to every gateway node), so
presence is correct in a horizontally-scaled deployment without any
in-process state. `PRESENCE_TTL = 90` (seconds) is the single TTL constant
used throughout `presence.ts`.

| Key pattern | Type | Holds | TTL | Set/refreshed by |
|---|---|---|---|---|
| `presence:user:{userId}` | hash | `{ [deviceId]: lastSeenEpochMs }` — one field per currently-online device | **none** — not expired directly; see [caveat](#known-caveats-verified-against-the-current-code) | `setOnline`, `refreshPresence` (hset); `setOffline`/`markDeviceOffline` (hdel field, delete key once empty) |
| `presence:user:{userId}:device:{deviceId}` | hash | `{ lastSeen: epochMs }` | `PRESENCE_TTL` (90s), refreshed on every `setOnline`/`refreshPresence` | `setOnline`, `refreshPresence` |
| `presence:sockets:{userId}` | set | socket ids currently open for this user, across all their devices | `PRESENCE_TTL` (90s), refreshed on register | `registerPresenceSocket`/`refreshPresenceSocket`; drained by `unregisterPresenceSocket` |
| `presence:device_sockets:{userId}:{deviceId}` | set | socket ids for this specific `(user, device)` pair | `PRESENCE_TTL` (90s) | `registerPresenceSocket`; drained by `unregisterPresenceSocket` |
| `presence:device_sockets_by_device:{deviceId}` | set | socket ids for this device, keyed by **device id alone** (no `userId`) | `PRESENCE_TTL` (90s) | `registerPresenceSocket`; drained by `unregisterPresenceSocket` |
| `presence:socket:{socketId}` | hash | `{ userId, deviceId }` — reverse lookup from a raw socket id | `PRESENCE_TTL` (90s) | `registerPresenceSocket`; deleted by `unregisterPresenceSocket` |

Why `presence:device_sockets_by_device:{deviceId}` exists as a *separate*
index from `presence:device_sockets:{userId}:{deviceId}`: some callers only
ever have a `deviceId` and no `userId` in hand — the device-revocation
pub/sub listener (`services/deviceRevocation.ts`) and the push-notification
online check are the two cited in the source comments — and this index lets
them resolve connected sockets without an extra DB round trip to look up the
owning user first. It replaced an in-process `Map` that was only visible to
sockets connected to the local gateway node (issue #341 in the source
comments).

`cleanupStaleSockets` and `reconcileBoot` (called on gateway boot, after the
Redis adapter attaches) walk `presence:sockets:*`/`presence:sockets:{userId}`
and use `io.in(socketId).fetchSockets()` to detect socket ids that are no
longer connected anywhere in the cluster, removing them via the same
`removeStaleSocketMapping` helper used by normal unregistration.

## The device → user presence model

Presence is tracked **per device first**, then aggregated to a user:

- `presence:user:{userId}` is a hash, not a boolean — one field per device
  that is currently considered online, keyed by `deviceId`.
- `isOnline(redis, userId)` is simply `HLEN presence:user:{userId} > 0` — a
  user is online if **any** device entry exists in that hash, regardless of
  which device or how recently (within whatever the hash currently holds).
- A single device going offline (`setOffline`/`markDeviceOffline`) removes
  only that device's field; the user stays online as long as at least one
  other device's field remains. The user transitions to fully offline only
  once the hash becomes empty, at which point the whole key is deleted.

This is why the module is organized around **device-keyed** operations
(`setOnline(redis, userId, deviceId, ...)`, not `setOnline(redis, userId,
...)`) even though the externally-visible presence state (`isOnline`) is
per-user.

## Heartbeat timing contract

Two constants in `services/heartbeat.ts` define the contract, plus one in
`services/presence.ts`:

| Constant | File | Value | Purpose |
|---|---|---|---|
| `HEARTBEAT_TIMEOUT_MS` | `heartbeat.ts` | **90,000 ms (90s)** | Server-side watchdog: if no `heartbeat` event arrives from a socket within this window, the server force-disconnects it and marks the device offline. |
| `LAST_SEEN_THROTTLE_MS` | `heartbeat.ts` | **30,000 ms (30s)** | Minimum spacing between `devices.lastSeenAt` DB writes for the same device — every heartbeat refreshes Redis, but the Postgres write is throttled to at most once per 30s per device. |
| `PRESENCE_TTL` | `presence.ts` | **90 seconds** | TTL on every per-device/per-socket Redis key (see table above); matches the heartbeat timeout so a device's Redis footprint expires at roughly the same time the watchdog would have fired anyway. |

The client-side recommended send interval is **30 seconds** (documented in
[`api-websocket-events.md`](api-websocket-events.md#heartbeat)), giving a 3×
margin before the 90-second server timeout fires — a single missed or
delayed heartbeat does not disconnect the socket.

> The module-level comment block at the top of `presence.ts` describes the
> per-device key's TTL as "60s" in prose. The actual constant in the same
> file is `PRESENCE_TTL = 90`. Treat the constant, not the comment, as
> authoritative — this documentation uses 90s throughout because that is
> what the code enforces.

### What one `heartbeat` event does, server-side

`handleHeartbeat(socket, userId, deviceId, redis)` runs, in order:

1. Clears and does **not** immediately reschedule the existing disconnect
   timer for this socket (the reschedule happens at the end, step 4).
2. If Redis is configured: `refreshPresence` (bumps the per-device hash entry
   and its TTL) and `refreshPresenceSocket` (re-registers the socket mapping,
   refreshing all four socket-related key TTLs from the table above).
3. Throttled DB write: if at least `LAST_SEEN_THROTTLE_MS` (30s) has elapsed
   since the last write for this `deviceId` (tracked in an in-process `Map`,
   per gateway node), updates `devices.lastSeenAt` and `devices.updatedAt`.
   A failed write is swallowed (`catch { /* non-critical */ }`) — a heartbeat
   is never allowed to fail the DB write path visibly to the client.
4. Reschedules the 90-second disconnect timer via the stored `schedule()`
   closure (set up by `startHeartbeatTimer` at connect time).

### What a timeout does, server-side

If no heartbeat (and no other activity — the timer is reset by `schedule()`,
which is only called from `startHeartbeatTimer` and `handleHeartbeat`, so an
otherwise-active socket that never sends `heartbeat` will still time out)
arrives within 90 seconds, the scheduled callback:

1. Logs the timeout.
2. If Redis is configured: `unregisterPresenceSocket` (remove this socket
   from all socket-mapping keys) and, only if that was the device's **last**
   remaining socket, `markDeviceOffline` (remove the device's field from
   `presence:user:{userId}`).
3. If the device became fully offline and the socket is still marked
   connected, emits `user_offline` and `presence_update` (volatile, i.e.
   best-effort/no-buffering) to every room the socket was in.
4. Force-disconnects the socket (`socket.disconnect(true)`) if still
   connected.

## Connection lifecycle, step by step

On a new Socket.IO connection (`index.ts`'s `io.on('connection', ...)`):

1. `startHeartbeatTimer(socket, userId, deviceId, redis, io)` — arms the
   90-second watchdog described above.
2. `devices.lastSeenAt` is updated immediately (unthrottled — this is a
   one-time write at connect, separate from the heartbeat throttle).
3. The socket joins its device room (`device:{deviceId}`), user room, and
   every conversation room it's a member of.
4. If Redis is configured:
   - `registerPresenceSocket` — writes the socket-mapping keys.
   - `cleanupStaleSockets` — prunes any dead socket ids left over from a
     previous session for this user.
   - `cancelPendingOfflineBroadcast(userId)` — see
     [Transition-broadcast behavior](#transition-broadcast-behavior).
   - `setOnline(redis, userId, deviceId)` — returns `true` only if the user's
     hash was empty before this call, i.e. this connection is what brought
     the user online (as opposed to e.g. a second device connecting while
     the user was already online via a first device).
   - If the user *became* online (and the pending-offline broadcast wasn't
     already cancelling a flap — see below) and `presenceVisible` is true,
     emits `user_online`/`presence_update` to every conversation room the
     user belongs to, and records the transition on co-members' resume
     streams for offline replay (`recordPresenceForCoMembers`).

On disconnect (`socket.on('disconnect', ...)`):

1. `clearHeartbeatTimer(socket.id)` — cancels the watchdog; a clean
   disconnect does not need the 90-second timeout to fire separately.
2. `devices.lastSeenAt` is updated again (unthrottled, same as connect).
3. **Gateway-shutdown short-circuit**: if the process is shutting down
   (`SIGTERM`/`SIGINT` received, or Socket.IO's own shutdown reasons), the
   handler returns immediately *without* touching presence state at all. The
   comment in `index.ts` is explicit about why: "we must NOT wipe presence —
   surviving devices re-assert via heartbeat and Redis TTLs." A rolling
   restart should not cause every connected user to flicker offline.
4. Otherwise: `unregisterPresenceSocket`, `cleanupStaleSockets`, and — only
   if this was the device's last socket — `setOffline`. If that leaves the
   user's hash empty (`fullyOffline`), the offline broadcast is scheduled
   (not fired immediately — see next section).

## Multi-device aggregation

Because `presence:user:{userId}` is a per-device hash, a user with two
connected devices (e.g. phone + laptop) has two fields in that one key. Each
device's connect/heartbeat/disconnect only ever touches its own field (and
its own `presence:user:{userId}:device:{deviceId}` key and own socket-mapping
keys) — devices never read or write each other's entries.

The aggregation rule is purely "any field present ⇒ online": there is no
weighting, no "most recent device wins," and no per-device online/offline
event exposed externally — only the user-level transition (hash
empty↔non-empty) is ever broadcast. A client reconnecting a second device
while the first is already connected does **not** re-trigger a `user_online`
broadcast, since `setOnline` returns `false` (`wasOnline` was already `true`)
in that case.

The Postgres fallback (`deriveDevicePresence`) aggregates differently: it
checks whether **any** non-revoked device for the user has `lastSeenAt`
within the last `DEVICE_PRESENCE_WINDOW_MS` (90,000 ms — the same 90-second
figure as `PRESENCE_TTL`, defined as its own local constant in `presence.ts`
rather than shared), and if none qualify, returns the single most recent
`lastSeenAt` across all of that user's non-revoked devices as `lastSeen`.

## Transition-broadcast behavior

Going offline is **debounced**, going online is not. This asymmetry exists
specifically to avoid a disconnect-then-immediately-reconnect blip (page
refresh, brief network hiccup, mobile app backgrounding) producing a visible
`user_offline` followed by `user_online` pair to every other member of a
shared conversation.

- `scheduleOfflineBroadcast(userId, broadcastFn)` — called only once the
  caller has already confirmed zero remaining live devices/sockets for the
  user (the underlying Redis state removal via `setOffline` has already
  happened synchronously; only the *broadcast* is deferred). It sets a timer
  for `PRESENCE_OFFLINE_GRACE_MS` (env-configurable, **default 5,000 ms**,
  falling back to the default on a missing/invalid/negative value), replacing
  any previous pending timer for the same user. The timer is `.unref()`'d so
  it never keeps the process alive on its own.
- `cancelPendingOfflineBroadcast(userId)` — called at the **start** of the
  connect handler, before the new connection's own online-check runs. If a
  reconnect lands within the grace window of a very recent disconnect (same
  user, any device — not necessarily the same device that disconnected), the
  pending offline broadcast is cancelled and the reconnect produces **no**
  `presence_update` at all: nothing was ever announced offline, so there is
  nothing to announce online again either (`becameOnline && !cancelledPendingOffline`
  gates the online broadcast specifically to suppress this).

When the offline broadcast does fire (grace window elapsed with no
reconnect), it re-reads conversation memberships fresh (not the stale
membership list captured at disconnect time) and calls
`deriveDevicePresence` to compute the `lastSeen` value to report, rather than
reusing any value cached from the live connection.

## Privacy gating

Two per-user boolean columns (`users.presenceVisible`,
`users.lastSeenVisible` — see
[`contracts-database-schema.md`](contracts-database-schema.md#users)) gate
what's ever broadcast or returned, independent of the Redis/DB mechanics
above:

- `presenceVisible` (default `false`) gates **all** presence broadcasting for
  that user — both the connect-time `user_online`/`presence_update` emission
  and the disconnect-time scheduled offline broadcast check this flag before
  emitting anything. `GET /users/:id/presence` returns `{ online: "unknown" }`
  for a user with `presenceVisible: false`, regardless of actual connection
  state.
- `lastSeenVisible` (default `false`) gates only whether a `lastSeen`
  timestamp is **included** in an otherwise-permitted presence payload —
  a user can be visibly online/offline without ever disclosing exactly when
  they were last active.

Both checks read the *current* value of these flags at the moment of
broadcast/query, not at connect time — a user who flips `presenceVisible` off
mid-session stops appearing in future presence broadcasts without needing to
reconnect (see the dedicated visibility-change broadcast in
`routes/users.ts`'s profile-update handler, which emits a synthetic
online/offline transition when `presenceVisible` itself changes).

## Known caveats, verified against the current code

- **`presence:user:{userId}` has no TTL of its own.** Every other
  presence-related Redis key in the table above carries `PRESENCE_TTL`.
  This one does not — it is only ever shrunk/deleted by an explicit
  `hdel`/`del` inside `setOffline`, `markDeviceOffline`, or
  `cleanupStaleSockets`'s indirect effects. If a gateway process crashes (as
  opposed to a graceful shutdown or a normal socket disconnect) between
  `setOnline` and any of those cleanup calls, that device's field can persist
  in the hash indefinitely — there is no background sweep that expires it by
  age. `isOnline()` would then report the user online based on a stale field
  with no live socket or process behind it, until some other event
  (`reconcileBoot` on the next gateway restart, a later `setOffline` call for
  the same device, or a device revocation) happens to clear it.
  `reconcileBoot` itself only rebuilds room membership from
  `presence:sockets:*` and prunes *socket-mapping* keys for dead sockets — it
  does not independently audit or repair the `presence:user:{userId}`
  device-presence hash.
- **The per-device TTL (`PRESENCE_TTL`) is the only thing that would
  eventually make a truly-dead device's own hash key disappear**, but that
  key (`presence:user:{userId}:device:{deviceId}`) expiring does **not**
  cascade into removing the corresponding field from
  `presence:user:{userId}` — the two are updated independently by the same
  call sites, not linked by Redis expiry notifications. A production
  deployment that cares about this edge case exactly would need either a
  periodic reconciliation pass or Redis keyspace-notification-driven cleanup,
  neither of which exists in the current code.
- **`isOnline()` does not re-validate recency.** It is `HLEN > 0`, not a
  check that any field's value is within `PRESENCE_TTL` of now — recency
  enforcement is delegated entirely to the per-device key's TTL and the
  explicit cleanup paths, not re-checked at read time.
- **The in-file header comment block in `presence.ts` is stale** — it
  describes a 60-second TTL and a more simplified "add socketId to a set"
  model closer to an earlier version of this module. The actual
  implementation (device-hash + per-device key + three tiers of
  socket-mapping keys, 90-second TTL) supersedes it; this document reflects
  the code, not the comment.
