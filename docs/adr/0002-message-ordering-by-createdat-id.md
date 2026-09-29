# ADR 0002: Order messages by `(createdAt, id)`, not `sequenceNumber`

- Status: accepted
- Date: 2026-09-29

## Context

Offline sync and in-conversation display both need a total order. A
per-conversation monotonic `sequenceNumber` was implemented briefly, then
dropped. The schema comment on `messages` and `routes/sync.ts` state why:
the same counter cannot also be a coherent **cross-conversation** cursor
for a device catching up on every chat it belongs to, and keeping two
counters was judged not worth the complexity.

`id` (UUID) is only a same-millisecond tiebreaker, not a clock.

## Decision

Order and paginate by `(createdAt, id)`. Envelope sync encodes that pair
as the cursor (`createdAt.getTime():id`). `GET /sync` does not emit
`sequenceNumber`. Group *control* events still use an epoch/sequence log
on the conversation row; that is a different problem (see
[group-epoch-sync.md](../group-epoch-sync.md)).

## Alternatives rejected

**Per-conversation `sequenceNumber` as the only order key.** Rejected
because a device's offline cursor has to move across many conversations.
A number that restarts per chat cannot be compared globally. Maintaining
a second, cross-conversation counter alongside it was rejected as
duplicated machinery for the same `createdAt` fact the row already has.

(The schema comment cites `#137` / `#128` for the sync argument; those
issue numbers in comments do not all match their original tickets. The
behaviour is what the code does.)

## Consequences

Clients must sort and resume with timestamps + ids. The web app still
types `sequenceNumber` in several places and falls back to `0`
(DRIFT-1 in
[contracts-response-types.md](../../apps/web/docs/contracts-response-types.md));
that is a known client bug, not a revival of the column.

See `apps/backend/src/db/schema.ts` (`messages`) and
`apps/backend/src/routes/sync.ts`.
