# Stellar chain listener and on-chain reconciliation

How `services/stellarListener.ts` watches Soroban contract events, writes
`token_transfers` / `treasury_proposals`, tracks its place in the ledger, and
behaves after downtime or an unreachable RPC.

The listener is started from `index.ts` only when both `STELLAR_RPC_URL` and
`TOKEN_TRANSFER_CONTRACT_ID` are set. Missing either logs

```text
[stellar-listener] STELLAR_RPC_URL or TOKEN_TRANSFER_CONTRACT_ID unset; listener disabled.
```

and leaves the API up. `GROUP_TREASURY_CONTRACT_ID` is optional: when set, a
second fetcher is attached for treasury events.

`runForever` never rethrows. RPC and DB errors are logged inside the loop so
a dead chain cannot take the HTTP/WebSocket process down.

## Watched events and database writes

Two independent pollers share the same loop (default interval 5s).

### `token_transfer` — topic `transfer`

`buildRpcFetcher` calls Soroban RPC `getEvents` filtered to
`TOKEN_TRANSFER_CONTRACT_ID` and topic `transfer`. Each event is mapped to:

| RPC field | Persisted column (`token_transfers`) |
| --- | --- |
| `txHash` | `tx_hash` (unique) |
| `value.to` | `recipient_address` |
| `value.amount` | `amount` (decimal string) |
| `value.memo` | `memo` (hex, optional) |

`conversation_id` / `sender_id` are filled by decoding `memo` as a message
UUID and looking up `messages`. If that misses, the writer falls back to an
arbitrary existing conversation and user; if the database has neither, the
event is dropped.

Write path: `INSERT … ON CONFLICT (tx_hash) DO UPDATE SET created_at = now()`.
A replayed ledger page refreshes `created_at` on the same row; it does not
insert a second transfer.

### `group_treasury` — proposal lifecycle

`buildTreasuryRpcFetcher` watches `GROUP_TREASURY_CONTRACT_ID` for:

| Contract event | Row status |
| --- | --- |
| `proposal_created` | `active` |
| `proposal_approved` | `approved` |
| `proposal_rejected` | `rejected` |
| `proposal_executed` | `executed` |
| `proposal_expired` | `expired` |

Persistence is an **update** on `(contract_id, proposal_id)` (unique index
`treasury_proposals_contract_proposal_idx`), copying `approvals` /
`rejections` when present. If no matching row exists, the event is ignored
(`if (!row) return`) — the listener does not insert treasury proposals. After
a successful update it emits `treasury_proposal_updated` to the linked
conversation Socket.IO room.

## Cursor / position tracking

Both pollers keep an **in-process** cursor (`pagingToken` from the last
successfully persisted event). There is no cursor table and nothing is
written to Redis or Postgres.

- First poll after process start: `cursor` is `null`. `getEvents` is called
  with `startLedger` and `cursor` both unset (the fetcher comment: "resume
  on cursor only").
- After each successful persist, the cursor advances to that event's
  `pagingToken`. A persist failure leaves the cursor unmoved so the next
  poll retries the same page.
- Token-transfer and treasury streams have separate cursors.

Cursors die with the process. A restart is a cold start: both cursors are
`null` again.

## Catch-up after downtime

Two different "down" cases:

1. **Process was down (restart / deploy).** Cursors reset. The next
   `getEvents` page is whatever the RPC returns without a cursor. Events
   still inside the RPC's retention window are re-delivered and upserted.
   Events that have aged out of that window are not replayed by this
   listener — those mirrored rows stay missing until something else writes
   them.
2. **Process stayed up but a poll failed.** `consecutiveFailures`
   increments. The loop waits `min(1000 * 2^(n-1), 30000)` ms and retries
   **from the last good in-memory cursor**, so it continues where it left
   off rather than rewinding.

## Idempotency

A replayed ledger event must not double-write:

- Token transfers: unique `tx_hash` + `ON CONFLICT DO UPDATE`. The same
  hash can be persisted any number of times; there is still one row.
- Treasury proposals: unique `(contract_id, proposal_id)`. A repeated
  event updates status / counts on the existing row.

Because the cursor only advances after a successful persist, a crash
mid-page can re-offer the same events; the unique keys absorb that.

## RPC unreachable — drift from on-chain state

When `getEvents` throws (timeout, 5xx, network partition):

- The error is logged as `fetch failed; reconnecting after backoff`.
- The API and WebSocket server keep running.
- No new rows are written for the duration of the outage.

**Yes, on-chain state can drift from the mirrored tables.** The listener is
a best-effort projection, not a consensus participant. During an RPC outage
the chain continues; `token_transfers` and `treasury_proposals` do not.
After reconnect, catch-up depends on RPC retention and on the treasury
update-only rule (a `proposal_*` event for a row that was never inserted
is silently skipped). There is no compensating backfill job in this
service.

Cross-referenced from [`IMPLEMENTATION_DOCS.md`](../../../IMPLEMENTATION_DOCS.md).
