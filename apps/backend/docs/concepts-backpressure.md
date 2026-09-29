# Backpressure and slow-consumer handling

How `services/backpressure.ts` watches per-socket send-buffer occupancy, what
the two thresholds do **today**, and how a client recovers after a hard
disconnect.

Gateway architecture mentions this module in passing
([concepts-gateway-architecture.md](./concepts-gateway-architecture.md) §8).
That overview currently describes shed as "stop sending". This page is the
accurate account of what is wired.

## Monitoring

On connect, `index.ts` calls `registerForBackpressure(socket)`. The socket is
added to an in-process set. A single `setInterval(checkBuffers, 5000)` runs
while any socket is registered and is cleared when the last one
unregisters (disconnect).

Each tick reads the engine.io transport's `bufferedAmount` (bytes waiting
in the WebSocket send buffer). If that field is missing or throws, occupancy
is treated as `0` — the socket is not shed or disconnected on a read error.

This is **per socket**, not per user. Two devices of the same account are
independent. The sets live in process memory; they are not shared across
gateway instances.

## Two tiers

| Tier | Env | Default | What the code does today |
| --- | --- | --- | --- |
| Shed | `SOCKET_SHED_THRESHOLD` | `32768` bytes | Marks the socket id in `shedSockets`, increments `clicked_backpressure_events_total{action="shed"}`, logs a warning. **Does not change emit behaviour.** |
| Disconnect | `SOCKET_BUFFER_THRESHOLD` | `65536` bytes | Same mark + metric `{action="disconnect"}`, then `socket.disconnect(true)`. **This is the only tier that affects delivery.** |

Both env values are parsed on every tick (`parseInt`, must be a positive
integer). Invalid or unset values fall back to the defaults above.

### Which tier changes emit behaviour

`isSocketShed(socketId)` exists and is exported. **Nothing in the backend
calls it.** No dispatcher, fan-out, or `socket.emit` path consults
`shedSockets` before sending.

So:

- **Disconnect** is enforced. Crossing `SOCKET_BUFFER_THRESHOLD` force-closes
  the socket.
- **Shed** is telemetry and a flag only. Crossing `SOCKET_SHED_THRESHOLD`
  does **not** stop new events being queued. A reader of the gateway
  overview must not infer that the server sheds load today — the buffer
  will keep growing until the disconnect threshold (or the client catches
  up and the flag is cleared on a later tick).

When occupancy drops back below the shed threshold, the id is removed from
`shedSockets`. That only matters for the metric/flag, not for emits.

## Hard disconnect — what the client sees

`socket.disconnect(true)` is a server-initiated close (`close` / disconnect
on the client, reason typically `io server disconnect`). The socket is
removed from rooms, presence, the device-delivery subscriber, and
backpressure monitoring. In-flight emits for that connection are gone.

The client is not told "you were too slow". There is no dedicated
backpressure event. From the client's point of view this is the same shape
as any other unexpected drop: the connection is dead and it must open a
new one and authenticate again.

## How resume recovers

A new connection is a new socket. After auth it re-joins conversation /
user / device rooms and is registered for backpressure from scratch
(`shedSockets` does not survive the old socket id).

Missed **durable** chat is not in the WebSocket buffer and is not in the
resume stream. The client recovers it with envelope sync
(`GET /sync`, `syncRequired: true` on `resume_complete`). See
[api-messages-sync.md](./api-messages-sync.md) and
[concepts-delivery-fanout.md](./concepts-delivery-fanout.md).

Missed **ephemeral** events (read/delivery receipts, presence, system
notices) are recovered by the resume/replay path:

1. Client emits `resume { lastEventId }` with the last Redis stream id it
   persisted.
2. The gateway reads `resume:events:${userId}` after that id (exclusive)
   and emits `ephemeral_replay` for each entry still within the 300s TTL /
   500-entry cap.
3. It finishes with `resume_complete { lastEventId, syncRequired: true }`.

Implementation: `services/resumeStream.ts`, handler in
`socket/messaging.ts`. Protocol notes:
[concepts-gateway-architecture.md](./concepts-gateway-architecture.md) §5
and [api-websocket-events.md](./api-websocket-events.md) (`resume` /
`resume_complete` / `ephemeral_replay`).

Events that were only sitting in the killed socket's send buffer and were
never recorded to Postgres or the resume stream are not replayed. That is
the loss window a slow consumer accepts when it is hard-disconnected.

Cross-referenced from [`IMPLEMENTATION_DOCS.md`](../../../IMPLEMENTATION_DOCS.md).
