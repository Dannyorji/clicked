# Data Retention

For every kind of data the system stores, this page gives:

- what is kept, and whether the server can read it
- how long it is kept
- what deletes it
- what a user-initiated erasure can and cannot reach

It describes the code as it is, not as intended. Where the two differ, the gap is
called out; see [Known gaps](#known-gaps) for the full list.

Stores covered: Postgres (backend), Redis (backend), the S3-compatible object
store, backend process memory, and Weaviate (AI agent). Retention windows that
can be tuned are listed with their environment variable; defaults come from
[environment-variables.md](environment-variables.md).

## Content versus metadata

The server stores two very different kinds of data. See
[threat-model.md](threat-model.md) for the full visibility analysis.

- **Content is end-to-end encrypted and the server cannot read it.** This
  covers message ciphertext (per-device envelopes, and MLS group ciphertext on
  `messages`), file blobs in the object store, and opaque MLS commit and
  Welcome material. It is encrypted on the client. The keys never reach the
  server: file keys travel inside envelopes, and private keys and ratchet state
  stay on the client
  ([threat-model → What the server cannot see](threat-model.md#what-the-server-cannot-see)).
  Deleting content is still worth doing, because it limits what an attacker who
  later gets a client key could decrypt.
- **Metadata is visible to the server.** This covers who is in which
  conversation, who sent what to whom and when, sizes, content types, device
  names and platforms, presence, IP addresses and user agents in audit logs,
  push endpoints, and wallet addresses. **Most metadata outlives the content it
  describes.** Deleting a message removes its ciphertext, but the row recording
  that the message existed stays
  ([threat-model → Residual metadata risk](threat-model.md#residual-metadata-risk)).
- **Exceptions, where the server can read content:**
  - **AI assistant prompts and replies.** Prompts are sent to the backend as
    plaintext and forwarded to the AI agent and OpenAI. Replies are stored in
    `messages.ciphertext` as **plaintext**, because the server writes them.
  - **System messages** (`contentType = 'system'`), which carry a plaintext
    `systemPayload`.
  - **Weaviate `Message` objects** in the AI agent, which hold plaintext
    `content` (see [AI agent](#ai-agent-weaviate)).

## Retention table

Legend: **C** is content (encrypted), **M** is metadata (visible), **P** is
plaintext content that the server can read.

| Data                                                                                                       | Store                                            | Kind                    | Retention                                                                                                                                                                                                        | Removed by                                                                                                |
| ---------------------------------------------------------------------------------------------------------- | ------------------------------------------------ | ----------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------- |
| **Messages and envelopes**                                                                                 |                                                  |                         |                                                                                                                                                                                                                  |                                                                                                           |
| Message row: sender, conversation, timestamps, `contentType`, `fileId`, `mlsEpoch`                         | Postgres `messages`                              | M                       | **Indefinite**, even after the message is deleted                                                                                                                                                                | Only a cascade when the conversation row is deleted                                                       |
| Message ciphertext (MLS group messages)                                                                    | Postgres `messages.ciphertext`                   | C                       | Until the sender deletes the message                                                                                                                                                                             | Sender deletes the message (sets the column to `NULL`)                                                    |
| Per-device envelope ciphertext                                                                             | Postgres `message_envelopes`                     | C                       | **7 days after delivery**, or **30 days after creation** if never delivered                                                                                                                                      | Envelope GC. Sender deletion removes them immediately.                                                    |
| AI assistant replies                                                                                       | Postgres `messages.ciphertext` + envelopes       | **P**                   | Row: **indefinite**. Envelopes: as above.                                                                                                                                                                        | Nothing (see [cannot reach](#what-erasure-cannot-reach))                                                  |
| System messages                                                                                            | Postgres `messages.systemPayload`                | M                       | Indefinite                                                                                                                                                                                                       | Conversation cascade only                                                                                 |
| **Files**                                                                                                  |                                                  |                         |                                                                                                                                                                                                                  |                                                                                                           |
| File blob (encrypted)                                                                                      | Object store                                     | C                       | While any live message references it                                                                                                                                                                             | File GC hard-deletes it once every referencing message is deleted and the grace period has passed         |
| File row: uploader, conversation, size, MIME type, sha256, storage key                                     | Postgres `files`                                 | M                       | **Indefinite**. The row is kept with `hardDeletedAt` set after the blob is gone.                                                                                                                                 | Conversation cascade only                                                                                 |
| Unconfirmed upload (`status = 'pending'`)                                                                  | Object store + `files`                           | C + M                   | **24 hours**                                                                                                                                                                                                     | File GC deletes both the blob and the row                                                                 |
| **Devices and keys**                                                                                       |                                                  |                         |                                                                                                                                                                                                                  |                                                                                                           |
| Device row: identity public key, name, platform, `lastSeenAt`, capabilities                                | Postgres `devices`                               | M                       | **Indefinite**. Revocation sets `revokedAt`; after 180 days the GC sets `staleFlaggedAt`.                                                                                                                        | Nothing. Rows are never hard-deleted.                                                                     |
| One-time prekeys                                                                                           | Postgres `device_prekeys`                        | M (public keys)         | Consumed: **30 days** after creation. Unconsumed: **90 days** after creation.                                                                                                                                    | Device GC. Device revocation deletes them immediately.                                                    |
| Signed prekey                                                                                              | Postgres `device_prekeys`                        | M (public key)          | Until replaced by the next upload                                                                                                                                                                                | Replaced on upload. Device revocation deletes it.                                                         |
| MLS KeyPackages                                                                                            | Postgres `mls_key_packages`                      | M (public)              | Same windows as one-time prekeys                                                                                                                                                                                 | Device GC. **Not** removed by revocation.                                                                 |
| Identity-key change log                                                                                    | Postgres `device_key_history`                    | M                       | **Indefinite, by design.** It exists to detect silent key swaps.                                                                                                                                                 | Nothing                                                                                                   |
| **Conversations and groups**                                                                               |                                                  |                         |                                                                                                                                                                                                                  |                                                                                                           |
| Conversation and membership                                                                                | Postgres `conversations`, `conversation_members` | M                       | Membership: until the user leaves. Group: until its last member leaves. **DMs: indefinite.**                                                                                                                     | Leaving a group. The last member leaving deletes the conversation and cascades.                           |
| MLS group state, commits, Welcomes; group control log                                                      | Postgres `mls_*`, `group_control_events`         | C (opaque payloads) + M | Indefinite                                                                                                                                                                                                       | Conversation cascade only                                                                                 |
| **Accounts**                                                                                               |                                                  |                         |                                                                                                                                                                                                                  |                                                                                                           |
| User profile and privacy settings                                                                          | Postgres `users`                                 | M                       | **Indefinite**                                                                                                                                                                                                   | Nothing. There is no account-deletion path.                                                               |
| Wallet addresses                                                                                           | Postgres `wallets`                               | M                       | Indefinite                                                                                                                                                                                                       | Nothing                                                                                                   |
| **Audit**                                                                                                  |                                                  |                         |                                                                                                                                                                                                                  |                                                                                                           |
| Security audit events: actor, subject, target, IP, user agent, metadata                                    | Postgres `audit_logs`                            | M                       | **Indefinite. Append-only by design.**                                                                                                                                                                           | Only a deliberate, privileged manual prune (see [below](#audit-logs))                                     |
| **Payments and governance**                                                                                |                                                  |                         |                                                                                                                                                                                                                  |                                                                                                           |
| Token transfers (mirrors on-chain `transfer` events)                                                       | Postgres `token_transfers`                       | M                       | Indefinite                                                                                                                                                                                                       | Conversation cascade (the chain copy is permanent)                                                        |
| Treasury proposals and votes                                                                               | Postgres `treasury_proposals`, `proposal_votes`  | M                       | Indefinite                                                                                                                                                                                                       | Nothing (the chain copy is permanent)                                                                     |
| On-chain transactions, balances, proposals, votes                                                          | Stellar ledger                                   | M (public)              | **Permanent**                                                                                                                                                                                                    | Nothing, ever                                                                                             |
| **Push**                                                                                                   |                                                  |                         |                                                                                                                                                                                                                  |                                                                                                           |
| Push subscription: endpoint, `p256dh`, `auth`                                                              | Postgres `push_subscriptions`                    | M                       | Until unsubscribed or the push service reports it dead                                                                                                                                                           | `DELETE /push/subscriptions`, or automatic pruning on HTTP 410/404. **Not** removed by device revocation. |
| **Redis (ephemeral)**                                                                                      |                                                  |                         |                                                                                                                                                                                                                  |                                                                                                           |
| Presence: `presence:user:*`, `presence:sockets:*`, `presence:device_sockets*`, `presence:socket:*`         | Redis                                            | M                       | Per-device and socket keys: **90 s TTL**, refreshed by heartbeats. The `presence:user:{userId}` hash has no TTL; its entry is removed when the device goes offline, and the key is deleted with the last device. | TTL expiry, disconnect or offline handling                                                                |
| Resume stream (`resume:events:{userId}`): typing, presence and other ephemeral events for reconnect replay | Redis stream                                     | M                       | **300 s** after the last event, capped at about 500 entries                                                                                                                                                      | TTL + `XADD MAXLEN ~ 500`                                                                                 |
| Replay-protection markers (`replay:{deviceId}:{eventId}`)                                                  | Redis                                            | M                       | **300 s** (`REPLAY_PROTECTION_TTL_SECONDS`)                                                                                                                                                                      | TTL                                                                                                       |
| Rate-limit counters (`rl:*`), abuse counters (`abuse:*`)                                                   | Redis                                            | M                       | Bucket window; `abuse:*` 1 h                                                                                                                                                                                     | TTL                                                                                                       |
| Prekeys-low latch                                                                                          | Redis                                            | M                       | 30 days                                                                                                                                                                                                          | TTL. Released on device revocation.                                                                       |
| Conversation list cache                                                                                    | Redis                                            | M                       | 30 s                                                                                                                                                                                                             | TTL, invalidated on writes                                                                                |
| **Process memory**                                                                                         |                                                  |                         |                                                                                                                                                                                                                  |                                                                                                           |
| Auth and device-link nonces                                                                                | Backend memory                                   | M                       | 5 min / 2 min, single use                                                                                                                                                                                        | Consumed on read, or lost on restart                                                                      |
| **AI agent**                                                                                               |                                                  |                         |                                                                                                                                                                                                                  |                                                                                                           |
| Indexed messages and embeddings                                                                            | Weaviate `Message`                               | **P**                   | **Indefinite**                                                                                                                                                                                                   | Nothing. There is no delete endpoint.                                                                     |
| Prompts and replies                                                                                        | OpenAI                                           | **P**                   | Governed by OpenAI's API data policy                                                                                                                                                                             | Outside this system                                                                                       |

Out of scope, owned by the operator: application logs (stdout/pino), database
and object-store backups, and metrics. Their retention is whatever the
deployment configures. **Backups keep erased data until they expire.**

## Garbage-collection jobs

Three in-process jobs enforce the windows above. Every backend instance starts
them from `apps/backend/src/index.ts` with `setInterval(...).unref()`.

- **The first pass runs one interval after boot, not at boot.**
- On a multi-instance deployment, every instance runs every job. The passes
  are plain idempotent `DELETE … WHERE` / `UPDATE … WHERE` statements, so
  concurrent runs are safe.
- Failures are logged (`[device-gc]`, `[envelope-gc]`, `[file-cleanup]`) and
  retried on the next tick.

| Job              | Source                    | Schedule (default)                           | What each pass does                                                                                                                                                                                                                      | Windows                                                                                                                                           |
| ---------------- | ------------------------- | -------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Device GC**    | `services/deviceGc.ts`    | Every **1 h** (`DEVICE_GC_INTERVAL_MS`)      | 1. Deletes one-time prekeys past their window.<br>2. Deletes MLS KeyPackages past the same window.<br>3. Sets `staleFlaggedAt` on long-revoked devices (flag only, no delete).                                                           | Consumed 30 d (`PREKEY_CONSUMED_RETENTION_DAYS`)<br>Unconsumed 90 d (`PREKEY_UNCONSUMED_MAX_AGE_DAYS`)<br>Stale 180 d (`DEVICE_STALE_AFTER_DAYS`) |
| **Envelope GC**  | `services/envelopeGc.ts`  | Every **30 min** (`ENVELOPE_GC_INTERVAL_MS`) | Deletes envelopes that were delivered longer ago than the delivered window, **or** created longer ago than the max-age window.                                                                                                           | Delivered 7 d (`ENVELOPE_DELIVERED_RETENTION_DAYS`)<br>Max age 30 d (`ENVELOPE_MAX_AGE_DAYS`)                                                     |
| **File cleanup** | `services/fileCleanup.ts` | Every **5 min** (`FILE_GC_INTERVAL_MS`)      | 1. Hard-deletes soft-deleted blobs that have no live message reference, then sets `hardDeletedAt`.<br>2. Deletes pending uploads (blob and row) past their TTL.<br>3. Re-enables push subscriptions whose 5-minute back-off has expired. | Grace 0 (`FILE_HARD_DELETE_GRACE_MS`)<br>Pending 24 h (`PENDING_UPLOAD_TTL_MS`)                                                                   |

Details that affect the windows:

- **Prekey and KeyPackage age is measured from `createdAt`**, not from
  consumption. `device_prekeys` has no `consumedAt` column, so a key consumed
  on day 29 is deleted on day 30.
- **The envelope max-age applies to undelivered envelopes.** A device that
  stays offline for more than 30 days loses those messages permanently. Keep
  `ENVELOPE_MAX_AGE_DAYS` at or above the longest offline period you support.
- **The `/sync` window is separate from GC.** `/sync` returns envelopes from
  the last 7 days (`ENVELOPE_TTL_SECONDS`). That controls what is served, not
  what is kept.
- **File hard-delete is idempotent.** `hardDeletedAt` is set only after the
  object-store delete succeeds, so a crash between the two steps is retried.
  Before each delete, the job re-checks that no live message references the
  file.
- **Push dead-endpoint pruning is not a GC job.** It happens inline when a
  send returns 410/404. A transient failure sets `disabledAt` for 5 minutes,
  and the file-cleanup tick re-enables the subscription.

## Per-category notes

### Messages and envelopes

A message is stored as one `messages` row plus one `message_envelopes` row per
recipient device. For MLS groups, the single group ciphertext lives on the
`messages` row instead. When the **sender** deletes a message (`DELETE
/messages/:id`, or the `delete_message` socket event):

1. `messages.deletedAt` is set and `messages.ciphertext` is set to `NULL`.
2. Every envelope for the message is deleted.
3. The attached file, if any, is soft-deleted, but only once no other live
   message references it.
4. `message_deleted` is broadcast so clients drop their copy.

The `messages` row itself stays: sender, conversation, timestamps, content
type, file link and MLS epoch. Edits are separate `messages` rows linked by
`editsMessageId`. Deleting the original does not delete its edits, so each
edit must be deleted on its own.

### Files

The lifecycle is `pending → ready → deleted`:

- **Unconfirmed uploads** are deleted after 24 hours.
- **Soft delete** (`deletedAt`) happens when the last live message referencing
  the file is deleted.
- **Hard delete** removes the blob from the object store on the next
  file-cleanup pass after the grace period (default: immediately, so within
  about 5 minutes). The `files` row survives with `hardDeletedAt` set, keeping
  its size, MIME type, sha256 and uploader. The file key never touches the
  server: it lives inside the envelope ciphertext.

### Devices and prekeys

Revoking a device (`DELETE /devices/:id`, or log-out-everywhere):

- sets `revokedAt`
- deletes all of the device's `device_prekeys`
- releases the prekeys-low latch
- disconnects the device's sockets on every instance

It does **not** delete the device row, its MLS KeyPackages, its push
subscriptions, its undelivered envelopes, or its key history. Those fall
under their own windows in the table above. The device row is kept on
purpose, to preserve audit history. The GC only flags it as stale after 180
days, and there is no hard delete.

### Audit logs

`audit_logs` records security events: device linked or revoked,
log-out-everywhere, key-bundle drains, failed auth, file access denied, group
membership changes. It is **append-only by design**, and the actor and subject
columns are **deliberately not foreign keys**
([security/audit-logging.md](security/audit-logging.md#append-only)):

- A cascade would delete a user's history along with the account it
  incriminates.
- `ON DELETE SET NULL` would issue an `UPDATE` that the append-only rule
  refuses.

As a result, **no user action and no GC job removes audit rows.** Removing
them is a deliberate, privileged operation: drop the trigger, prune, then
recreate the trigger. Rows hold identifiers, IP address, user agent and
sanitised metadata, and never message content.

> **Gap:** the trigger that enforces append-only (`audit_logs_no_mutation`) is
> described in `security/audit-logging.md` and `db/schema.ts`, but it is **not
> present in any migration**. `apps/backend/drizzle/` contains only
> `0000_lean_scrambler.sql`, and the `0001_audit_logs.sql` the audit doc cites
> does not exist. Until it is added, audit rows are append-only by application
> convention only, and the database account can update or delete them.

### Presence and resume streams (Redis)

Everything in Redis is short-lived metadata with a TTL:

- **Presence keys** expire 90 seconds after the last heartbeat and are deleted
  on disconnect.
- **The resume stream** holds ephemeral events (typing, presence and similar,
  not message content) for replay after a reconnect. It expires 300 seconds
  after its last write and is trimmed to about 500 entries.

The durable part of presence is `devices.lastSeenAt` in Postgres, which is kept
as long as the device row: indefinitely. If Redis is flushed, all of this is
lost with no user-visible effect beyond a presence blip.

### Push subscriptions

A subscription stores the browser's push endpoint and its encryption keys
(`p256dh`, `auth`). It is removed when:

- the device calls `DELETE /push/subscriptions`, or
- the push service returns 410 Gone or 404, in which case it is pruned on the
  next send.

It is **not** removed when the device is revoked, and the FK cascade never
fires because device rows are never deleted. The push provider (the browser
vendor) keeps its own delivery metadata, outside this system.

### AI agent (Weaviate)

`POST /index/message` in `apps/ai_agent` stores the **plaintext** message
`content` plus its embedding in Weaviate. `GET /search` reads it back. There is
no delete endpoint and no retention job. No code in this repo calls
`/index/message` today. **Wiring it up would store server-readable plaintext
indefinitely**, which contradicts the end-to-end encryption model in
[threat-model.md](threat-model.md). Any caller needs an explicit opt-in, a
retention window, and deletion that follows message deletion.

## User-initiated erasure

There is **no account-deletion or "erase my data" endpoint.** Users can only
remove data object by object:

| User action                          | What it removes                                                                                                                             | What it leaves                                                                                                                                                                   |
| ------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Delete own message                   | Its ciphertext, all its envelopes, and the file blob if no other live message uses it (within about 5 minutes)                              | The message row (metadata), edit rows, the `files` row, and copies already on recipients' devices                                                                                |
| Revoke a device / log out everywhere | The device's prekeys and live sockets                                                                                                       | The device row, key history, MLS KeyPackages (until GC), push subscriptions, undelivered envelopes (until GC), and an audit row recording the revocation                         |
| Leave a group                        | The membership row. If they are the last member, the whole conversation cascades (messages, envelopes, files rows, MLS state, control log). | The user's past messages in a group that still has members. A `member_left` control event and an audit row. **For the last member: file blobs** (see [Known gaps](#known-gaps)). |
| Unsubscribe from push                | That push subscription                                                                                                                      | Nothing else                                                                                                                                                                     |
| Change privacy settings              | Future presence and read-receipt exposure to _other users_                                                                                  | What the server already holds. The server sees presence regardless.                                                                                                              |

## What erasure cannot reach

These records survive every action a user can take. Some survive by design,
some because of gaps in the code:

1. **Audit logs, by design.** They are append-only and not FK-linked, so that
   deleting an account cannot erase the history of what was done with it. Only
   an operator can prune them, deliberately.
2. **On-chain data, permanently.** Token transfers, treasury deposits and
   withdrawals, proposals and votes live on the Stellar ledger. They are
   public, and no one can delete them: not the user, not the operator. The
   Postgres mirrors (`token_transfers`, `treasury_proposals`,
   `proposal_votes`) can be removed, but the chain copy cannot. Wallet
   addresses tie those records to the user.
3. **Account and device records.** There is no deletion path for `users`,
   `wallets` or `devices`. `device_key_history` is permanent by design.
4. **Message metadata.** Deleted messages leave rows that record sender,
   conversation and time. DM conversations cannot be left, so their messages,
   `files` rows and membership are kept indefinitely.
5. **AI assistant replies.** They are stored as server-readable plaintext
   under the assistant's user ID. The delete route only lets the _sender_
   delete, so no user can remove them.
6. **Copies outside the server:** recipients' devices, the push provider,
   OpenAI (assistant prompts), and operator backups and logs.

## Known gaps

These are the places where the code falls short of what this page, or other
docs, would lead a reader to expect:

- **No account deletion.** Building one has to decide:
  - what cascades (`users` cascades widely through FKs)
  - what is tombstoned instead (audit rows must stay; see above)
  - how to handle DMs, where the other party's copy is legitimately theirs
- **Audit append-only is not enforced in the database.** The trigger is
  missing from migrations (see [Audit logs](#audit-logs)).
- **Orphaned file blobs on conversation deletion.** When the last member
  leaves a group, `files` rows are cascade-deleted by the database, so the
  file-cleanup job never sees them and **their encrypted blobs stay in the
  object store forever**.
- **Revocation leaves push subscriptions and KeyPackages behind.** Push
  subscriptions for a revoked device are never removed unless the push service
  reports them dead.
- **Weaviate stores plaintext with no retention.** See
  [AI agent](#ai-agent-weaviate).
