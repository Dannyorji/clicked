# Implementation docs

Index of the docs that describe how this tree actually behaves. Concept
and contract pages live next to the code they cover; this file is only
the map.

## Decisions

- [docs/adr/README.md](./docs/adr/README.md) — Architecture Decision
  Records (devices table, message ordering, ciphertext-only storage,
  per-device envelopes, sealed-box → Signal, MLS for groups).

## Backend concepts

- [apps/backend/docs/concepts-configuration.md](./apps/backend/docs/concepts-configuration.md) —
  boot-validated vs lazy env, `loadEnv()`, object-store singleton.
- [apps/backend/docs/concepts-backpressure.md](./apps/backend/docs/concepts-backpressure.md) —
  socket buffer tiers; only disconnect is enforced today.
- [apps/backend/docs/concepts-stellar-listener.md](./apps/backend/docs/concepts-stellar-listener.md) —
  Soroban event watch, cursors, idempotency, RPC failure.
- [apps/backend/docs/concepts-gateway-architecture.md](./apps/backend/docs/concepts-gateway-architecture.md)
- [apps/backend/docs/concepts-delivery-fanout.md](./apps/backend/docs/concepts-delivery-fanout.md)
- [apps/backend/docs/concepts-storage-push-jobs.md](./apps/backend/docs/concepts-storage-push-jobs.md)

## Operations and security

- [docs/runbook.md](./docs/runbook.md)
- [docs/observability.md](./docs/observability.md)
- [docs/threat-model.md](./docs/threat-model.md)
- [docs/security/rate-limits.md](./docs/security/rate-limits.md)
- [`.env.example`](./.env.example) — environment-variable reference
