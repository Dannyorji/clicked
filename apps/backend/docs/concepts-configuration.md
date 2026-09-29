# Backend configuration loading and validation

How the gateway reads environment variables: which ones fail boot, which ones
are read later, and how that interacts with the object-store singleton.

This page is about **when** a variable is consumed. The catalog of names,
defaults, and meanings lives in the environment-variable reference
([`.env.example`](../../../.env.example)). Rate-limit buckets and override
syntax are in [`docs/security/rate-limits.md`](../../../docs/security/rate-limits.md).
Do not treat the tables below as a second copy of those lists.

## Boot-validated set (`src/config.ts`)

`index.ts` calls `loadEnv()` immediately after `dotenv.config()`. `loadEnv()`
parses `process.env` against `EnvSchema` (Zod). On success it returns the
typed object and prints nothing. On failure it logs the offending names and
**exits the process with code 1** — the HTTP server is never bound.

Required (missing or empty → hard startup failure):

| Variable | Constraint |
| --- | --- |
| `DATABASE_URL` | non-empty string |
| `REDIS_URL` | non-empty string |
| `JWT_SECRET` | non-empty string |
| `PORT` | positive integer (string is coerced) |
| `TOKEN_TRANSFER_CONTRACT_ID` | non-empty string |
| `OBJECT_STORE_ENDPOINT` | non-empty string |
| `OBJECT_STORE_BUCKET` | non-empty string |
| `OBJECT_STORE_ACCESS_KEY` | non-empty string |
| `OBJECT_STORE_SECRET_KEY` | non-empty string |
| `OBJECT_STORE_REGION` | non-empty string |
| `OBJECT_STORE_FORCE_PATH_STYLE` | `true` / `false` / `1` / `0` |

Optional in the same schema (validated only when present; absence does not
fail boot): `VAPID_PUBLIC_KEY`, `VAPID_PRIVATE_KEY`, `VAPID_SUBJECT`, the
legacy `S3_*` aliases, and `IDEMPOTENCY_TTL_SECONDS`.

`assertTransportSecurityConfig()` runs next. That is a separate check
(`ENFORCE_TLS`, `ALLOWED_ORIGINS`) and is not part of `EnvSchema`.

### Failure output

If `DATABASE_URL` is missing, stderr looks like this and the process exits 1:

```text
Missing or invalid environment variables: DATABASE_URL
  - DATABASE_URL: DATABASE_URL is required
```

An empty environment lists every required name, one `- name: message` line
each. A non-numeric `PORT` is reported the same way (`PORT must be an integer`).

## Lazy vs eager

**Eager (boot).** `EnvSchema` variables above. The process does not start
without them. `index.ts` then immediately calls `getObjectStore()`, so the
object-store singleton is constructed on the boot path too — but the
construction itself is still lazy at the *module* level (see below).

**Lazy (first use, no boot failure).** Everything else is read from
`process.env` at the call site, with a hardcoded default when unset or
malformed:

- Rate-limit buckets in `src/config/rateLimits.ts` via `getRateLimitRule()`.
  Each call re-reads `RATE_LIMIT_<BUCKET>` (and the legacy
  `SOCKET_RATE_LIMIT_PER_SEC` for `socket_default`). A malformed override is
  warned and ignored; the default stands. `RATE_LIMIT_DISABLED=true` is a
  kill switch, also read on demand.
- Backpressure thresholds (`SOCKET_BUFFER_THRESHOLD`, `SOCKET_SHED_THRESHOLD`)
  — see [concepts-backpressure.md](./concepts-backpressure.md).
- Stellar RPC wiring (`STELLAR_RPC_URL`, `GROUP_TREASURY_CONTRACT_ID`) —
  missing values disable the listener rather than crashing; see
  [concepts-stellar-listener.md](./concepts-stellar-listener.md).
- Sync/envelope knobs such as `ENVELOPE_TTL_SECONDS` and `SYNC_PAGE_SIZE`.

Rate-limit rules are deliberately not cached at import time so tests and the
runbook can change a limit without restarting the module graph.

## `loadEnv()` and the object-store singleton

`lib/objectStore.ts` keeps a process-wide singleton behind `getObjectStore()`.
The client is **not** built at import time. The first call constructs it:

- `NODE_ENV === 'production'` → `createObjectStore(loadEnv())` (real S3 /
  MinIO / R2 client, credentials from the boot-validated `OBJECT_STORE_*`
  fields).
- otherwise → `getLocalObjectStore()` (fs-backed store; no live S3 needed).

Building the singleton in `objectStore.ts` (instead of exporting a client
from `index.ts`) avoids a circular import:

```text
index.ts → app.ts → routes → lib/storage.ts → index.ts
```

`storage.ts` and `services/fileCleanup.ts` both call `getObjectStore()`, so
there is exactly one client/bucket pair per process. `loadEnv()` is invoked
again inside that first production call; by then boot validation has already
run, so it is a typed re-parse, not a second chance to start with a broken
env.

`resetObjectStoreForTests()` clears the memo so tests can change env.

## Production vs development in `lib/storage.ts`

`generatePresignedPut` / `generatePresignedGet` branch on
`NODE_ENV === 'production'`:

| Branch | Implementation |
| --- | --- |
| production | `getObjectStore()` — the S3-compatible client from `OBJECT_STORE_*` |
| any other `NODE_ENV` | `getLocalObjectStore()` — local-disk store, URLs served by `routes/localStorage.ts` |

The same `NODE_ENV` split exists inside `getObjectStore()` itself. The
storage helpers short-circuit to the local store in non-production so
presigned-URL callers never touch the S3 constructor on a laptop or in CI.

Cross-referenced from [`IMPLEMENTATION_DOCS.md`](../../../IMPLEMENTATION_DOCS.md).
