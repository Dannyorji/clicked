# Environment Variables

Every environment variable read by the four apps in this repo: `apps/backend`,
`apps/web`, `apps/ai_agent` and `apps/tests`. Sources:

- [`.env.example`](../.env.example)
- [`apps/backend/src/config.ts`](../apps/backend/src/config.ts) (boot-time schema)
- [`apps/backend/src/config/rateLimits.ts`](../apps/backend/src/config/rateLimits.ts)
- every direct `process.env` / `os.environ` read under `apps/`

`apps/tests` reads no environment variables. The backend test suite sets its
own values in `apps/backend/src/__tests__/setup.ts`.

## How to read the table

**Loaded** says when the value is read, and so how a bad value shows up:

| Loaded | Meaning | If the value is missing or malformed |
| --- | --- | --- |
| `boot` | Validated by `loadEnv()` in `config.ts`, called at the top of `apps/backend/src/index.ts`. | **Crashes on start:** logs `Missing or invalid environment variables: …` and exits 1. |
| `boot (TLS)` | Checked by `assertTransportSecurityConfig()` right after `loadEnv()`. | **Crashes on start** for the contradictions described in the row. Other cases only log a warning. |
| `module` | Read once when the module is first imported. No validation. | **Fails silently.** Usually the default is used, but a few `parseInt` reads become `NaN` (see the row). A restart is needed to pick up a change. |
| `lazy` | Read on every call or job run. Parsed with a fallback. | **Fails silently.** An invalid value falls back to the default, sometimes with a `console.warn`. |
| `build` | Inlined into the web client bundle by `next build`. | **Fails silently** in the browser. A rebuild is needed to pick up a change. |
| `unused` | Declared in `.env.example` or `config.ts` but read by no code. | Nothing. Setting it has no effect. |

`boot` only checks that a value is **present and well-formed**. A
`DATABASE_URL` that points at the wrong host still passes and only fails on
the first query.

**Secret** 🔒 marks credentials. See [Secrets](#secrets).

**Public** 🌐 marks values compiled into the browser bundle. See
[`NEXT_PUBLIC_*` values are public](#next_public_-values-are-public).

## Caveat: the backend loads `.env` too late for `module` reads

`apps/backend/src/index.ts` calls `dotenv.config()` **after** its `import`
statements. The backend is an ES module (`"type": "module"`), so every imported
module has already run its top-level code before dotenv loads the file. As a
result:

- Values supplied **only** through a `.env` file are invisible to every
  `module` row except those read in `index.ts` itself (`PORT`, `REDIS_URL` for
  the Socket.IO adapter, `STELLAR_RPC_URL`, `TOKEN_TRANSFER_CONTRACT_ID`,
  `GROUP_TREASURY_CONTRACT_ID`). The other `module` readers fall back to their
  defaults.
- **`JWT_SECRET` is affected.** `lib/jwt.ts` runs first, finds no value, falls
  back to `'test-secret'` and writes that back into `process.env`. dotenv then
  refuses to overwrite it, and `loadEnv()` passes because the variable is now
  set. **Tokens are signed with `'test-secret'`**, whatever the `.env` file
  says.
- `DATABASE_URL` (`db/index.ts`), `REDIS_URL` (`lib/redis.ts`) and the
  `VAPID_*` keys (`services/pushNotification.ts`) are affected the same way.

Values injected by the process environment (Docker `environment:`, Kubernetes,
systemd, `export …`) are not affected. **In any deployed environment, set
variables through the real process environment, not a `.env` file**, until
dotenv is loaded before the other imports (for example with
`node --import dotenv/config`).

## All variables

| Variable | App | Required | Default | Loaded | What breaks if it is wrong |
| --- | --- | --- | --- | --- | --- |
| **Auth** | | | | | |
| `JWT_SECRET` 🔒 | backend | **Yes** | none (`lib/jwt.ts` falls back to `'test-secret'`, see caveat) | `boot` + `module` | A weak or leaked value lets anyone forge session tokens for any user. Changing it invalidates every existing session. |
| **Core infrastructure** | | | | | |
| `DATABASE_URL` 🔒 | backend | **Yes** | none (`db/index.ts` falls back to `postgres://user:password@localhost:5432/testdb`) | `boot` + `module` | Boot passes if the value is non-empty. A wrong host or credentials only show up as failed queries at runtime. It is also read by `drizzle.config.ts` for migrations (empty string if unset). 🔒 because it contains the DB password. |
| `REDIS_URL` 🔒 | backend | **Yes** | none (the Socket.IO adapter falls back to `redis://localhost:6379`) | `boot` + `module` | The Socket.IO Redis adapter logs a warning and **degrades to single-instance mode**: multi-instance rooms and presence break. If the value is missing at import time, `lib/redis.ts` sets `redis = null`, which disables caching, presence reconciliation and device-revocation fan-out. 🔒 when the URL embeds a password. |
| `PORT` | backend | **Yes** | none (the `3001` fallback in `index.ts` is never reached because `loadEnv()` exits first) | `boot` | Must be a positive integer, or boot fails. A mismatch with the web app's URLs means the web app cannot reach the API. It is also used to build local-disk presigned URLs in dev. |
| `NODE_ENV` | backend | Required in prod | unset → treated as `development` | `lazy` | `production` switches to the real S3 object store and unmounts `/local-storage`. **If it is unset in production**, uploads go to the container's local disk (`.local-storage/`) with `http://localhost` URLs. If `APP_ENV` is also unset, TLS enforcement, HSTS and the origin check are off as well. `test` silences `morgan` request logging. |
| `APP_ENV` | backend | Optional | falls back to `NODE_ENV`, then `development` | `lazy` | Selects the transport-security posture. Only `development` and `test` allow plaintext. **Any other value is treated as production.** |
| **Transport security** (see [`security/tls-and-pinning.md`](security/tls-and-pinning.md)) | | | | | |
| `ENFORCE_TLS` | backend | Optional | on outside `development`/`test` | `boot (TLS)` + `lazy` | Accepts `true/1/yes/false/0/no`; any other value is ignored. `false` outside dev logs a warning and **accepts plaintext `http`/`ws`**. |
| `ALLOWED_ORIGINS` | backend | Required in prod | empty | `boot (TLS)` + `lazy` | **Boot fails** if TLS is enforced and the list contains an `http://` origin. If empty outside dev, any `https://` origin can call the API and open a socket (a warning is logged). If it omits the web app's origin, CORS and the socket handshake are refused. |
| `TRUST_PROXY` | backend | Optional | `1` | `lazy` | Non-integer → `1`. `1` with the gateway exposed directly lets clients forge `X-Forwarded-Proto: https`. `0` behind a load balancer makes every request look plaintext, so TLS enforcement refuses all traffic. |
| `HSTS_MAX_AGE` | backend | Optional | `31536000` | `lazy` | Invalid → default. `0` removes the HSTS header. |
| `HSTS_PRELOAD` | backend | Optional | `true` | `lazy` | Adds `preload` to HSTS. Hard to undo once browsers have preloaded the domain. |
| `TLS_PINNED_HOSTS` | backend | Optional | empty | `lazy` | Hostnames advertised at `GET /security/transport-policy`. Public, not secret. |
| `TLS_PINNED_SPKI_SHA256` | backend | Optional | empty | `lazy` | Malformed pins are dropped with a warning. Mobile clients enforce pins only when a backup pin is also set. **A wrong pin plus a backup locks every installed client out.** |
| `TLS_BACKUP_SPKI_SHA256` | backend | Optional (required with the above) | empty | `lazy` | Without it, pinning is advertised as not enforced. |
| `TLS_PIN_MAX_AGE_SECONDS` | backend | Optional | `5184000` (60 days) | `lazy` | Invalid → default. How long clients cache the pin set. Too long makes rotation slow. |
| `TLS_PIN_REPORT_URI` | backend | Optional | none | `lazy` | Where clients report pin failures. |
| **Blockchain** | | | | | |
| `TOKEN_TRANSFER_CONTRACT_ID` | backend | **Yes** | none | `boot` | Also turns on the Stellar transfer listener, together with `STELLAR_RPC_URL`. **A wrong ID passes boot**; the listener just watches the wrong contract and transfer events never arrive. Must match `NEXT_PUBLIC_TOKEN_TRANSFER_CONTRACT`. |
| `STELLAR_RPC_URL` | backend | Optional | none | `module` (index.ts) | **Not in `.env.example`** (which lists `RPC_URL` instead). If unset, the transfer and treasury listener is disabled with a single log line. |
| `GROUP_TREASURY_CONTRACT_ID` | backend | Optional | `'stub'` in `routes/treasury.ts` | `module` + `lazy` | If unset, treasury events are not watched and the treasury route reports contract `stub`. |
| `RPC_URL` | — | — | — | `unused` | Listed in `.env.example`, but the backend reads `STELLAR_RPC_URL`. Setting only `RPC_URL` leaves the listener off. |
| `PROPOSALS_CONTRACT_ID` | — | — | — | `unused` | No code reads it. |
| **Object storage** (used only when `NODE_ENV=production`, but validated in every environment) | | | | | |
| `OBJECT_STORE_ENDPOINT` | backend | **Yes** | `.env.example`: `http://localhost:9000` | `boot` | A wrong endpoint passes boot. Presigned upload and download URLs then fail when a client uses them. |
| `OBJECT_STORE_BUCKET` | backend | **Yes** | `.env.example`: `clicked` | `boot` | Same as above: uploads fail at runtime. |
| `OBJECT_STORE_ACCESS_KEY` 🔒 | backend | **Yes** | `.env.example`: `clicked` (local MinIO only) | `boot` | Presigned URLs are rejected by the store (`SignatureDoesNotMatch`). |
| `OBJECT_STORE_SECRET_KEY` 🔒 | backend | **Yes** | `.env.example`: `clickedsecret` (local MinIO only) | `boot` | Same as above. A leak gives full read/write access to every stored attachment. |
| `OBJECT_STORE_REGION` | backend | **Yes** | `.env.example`: `us-east-1` | `boot` | Signature mismatch on region-strict providers. |
| `OBJECT_STORE_FORCE_PATH_STYLE` | backend | **Yes** | `.env.example`: `true` | `boot` | Must be `true/false/1/0`, or boot fails. The wrong style for the provider produces unreachable URLs (use `true` for MinIO, `false` for AWS S3 and R2). |
| `S3_ENDPOINT`, `S3_REGION`, `S3_ACCESS_KEY_ID`, `S3_SECRET_ACCESS_KEY` 🔒, `S3_BUCKET`, `S3_FORCE_PATH_STYLE` | — | — | — | `unused` | Declared optional in `config.ts` but never read. Use the `OBJECT_STORE_*` variables instead. |
| `LOCAL_STORAGE_DIR` | backend | Optional (dev) | `<cwd>/.local-storage` | `lazy` | Dev and test only: where the local-disk object store writes files. |
| `STORAGE_ENDPOINT` | backend | Optional (dev) | `http://localhost:$PORT/local-storage` | `lazy` | Dev and test only: the base URL of local presigned URLs. If wrong, uploads from another device or container fail. |
| **Push notifications (VAPID)** | | | | | |
| `VAPID_PUBLIC_KEY` | backend | Optional | none | `boot` (optional, unchecked) + `module` + `lazy` | If unset, `GET /push/vapid-public-key` returns `configured: false` and the web app skips push. If it does not pair with the private key, push delivery fails silently. |
| `VAPID_PRIVATE_KEY` 🔒 | backend | Optional | none | `boot` (optional, unchecked) + `module` | If unset, push is silently disabled. A leak lets anyone send push notifications to your subscribers. |
| `VAPID_SUBJECT` | backend | Optional | `mailto:admin@clicked.app` | `boot` (optional, unchecked) + `module` | Push services may reject a subject that is not a `mailto:` or `https:` URL. |
| **Messaging and sync** | | | | | |
| `REPLAY_PROTECTION_TTL_SECONDS` | backend | Optional | `300` | `lazy` | **Not in `.env.example`.** This is the window that actually controls replay and duplicate `eventId` rejection. Values outside `1`–`86400` fall back to the default. |
| `IDEMPOTENCY_TTL_SECONDS` | backend | Optional | `.env.example` says `86400` | `boot` | A non-positive or non-integer value **crashes boot**, but the parsed value is **never used**. Replay protection reads `REPLAY_PROTECTION_TTL_SECONDS` instead. |
| `PRESENCE_OFFLINE_GRACE_MS` | backend | Optional | `5000` | `lazy` | Invalid → default. Too high delays "offline" status. `0` produces offline/online flapping on brief disconnects. |
| `PREKEY_LOW_THRESHOLD` | backend | Optional | `20` | `module` | Invalid → default. Too low and devices run out of one-time prekeys before they are told to replenish. |
| `SOCKET_EVENT_MAX_AGE_MS` | backend | Optional | `300000` | `module` | **Not in `.env.example`.** A non-numeric value becomes `NaN` and **every socket event is rejected as stale**. |
| `SOCKET_EVENT_MAX_FUTURE_SKEW_MS` | backend | Optional | `30000` | `module` | **Not in `.env.example`.** Same `NaN` failure as above. |
| `ENVELOPE_TTL_SECONDS` | backend | Optional | `604800` (7 days) | `module` | **Not in `.env.example`.** How far back `/sync` returns envelopes. A non-numeric value becomes `NaN` and breaks the sync cutoff. |
| `SYNC_PAGE_SIZE` | backend | Optional | `50` | `module` | **Not in `.env.example`.** Maximum `/sync` page size. A non-numeric value becomes `NaN`. |
| `MAX_PAYLOAD_SIZE` | backend | Optional | `16384` bytes | `lazy` | **Not in `.env.example`.** Too low and legitimate socket messages are rejected. |
| `MAX_ENVELOPE_SIZE` | backend | Optional | `4096` bytes | `lazy` | **Not in `.env.example`.** Too low and legitimate encrypted envelopes are rejected. |
| `FIRST_CONTACT_HOUR_LIMIT` | backend | Optional | `5` per hour | `module` | **Not in `.env.example`.** A non-numeric value becomes `NaN` and **every first-contact message is blocked**. |
| `GROUP_INVITE_HOUR_LIMIT` | backend | Optional | `10` per hour | `module` | **Not in `.env.example`.** A non-numeric value becomes `NaN` and **every group invite is blocked**. |
| `XMTP_ENV` | — | — | — | `unused` | Listed in `.env.example`; no code reads it. |
| **Socket backpressure** | | | | | |
| `SOCKET_BUFFER_THRESHOLD` | backend | Optional | `65536` bytes | `lazy` | **Not in `.env.example`.** A socket whose send buffer exceeds this is disconnected. |
| `SOCKET_SHED_THRESHOLD` | backend | Optional | `32768` bytes | `lazy` | **Not in `.env.example`.** Above this, the socket is marked "shed" and a warning and metric are emitted. Nothing reads the flag yet (`isSocketShed` has no callers), so no traffic is actually dropped. Should stay below `SOCKET_BUFFER_THRESHOLD`. |
| **Background GC jobs** (none in `.env.example`) | | | | | |
| `DEVICE_GC_INTERVAL_MS` | backend | Optional | `3600000` (1 h) | `lazy` (at job start) | Invalid → default. |
| `PREKEY_CONSUMED_RETENTION_DAYS` | backend | Optional | `30` | `lazy` | Invalid → default. How long consumed prekeys are kept for audit. |
| `PREKEY_UNCONSUMED_MAX_AGE_DAYS` | backend | Optional | `90` | `lazy` | Invalid → default. Too low deletes prekeys that peers still need to start sessions. |
| `DEVICE_STALE_AFTER_DAYS` | backend | Optional | `180` | `lazy` | Invalid → default. |
| `ENVELOPE_GC_INTERVAL_MS` | backend | Optional | `1800000` (30 min) | `lazy` (at job start) | Invalid → default. |
| `ENVELOPE_DELIVERED_RETENTION_DAYS` | backend | Optional | `7` | `lazy` | Invalid → default. |
| `ENVELOPE_MAX_AGE_DAYS` | backend | Optional | `30` | `lazy` | Invalid → default. Too low and **messages to offline devices are deleted before they are delivered**. |
| `FILE_GC_INTERVAL_MS` | backend | Optional | `300000` (5 min) | `lazy` | Invalid → default. |
| `FILE_HARD_DELETE_GRACE_MS` | backend | Optional | `0` | `lazy` | Invalid → default. |
| `PENDING_UPLOAD_TTL_MS` | backend | Optional | `86400000` (24 h) | `lazy` | Invalid → default. Too low deletes uploads that are still in progress. |
| **Rate limits** (see [`security/rate-limits.md`](security/rate-limits.md)). Format: `<limit>[/<windowSeconds>]`. A malformed value logs `[rateLimit] ignoring malformed …` and uses the default. All are `lazy`: read on every check, so no restart is needed. | | | | | |
| `RATE_LIMIT_DISABLED` | backend | Optional | `false` | `lazy` | Only the exact string `true` disables **every** limit. **Never set it in production**, because it re-opens enumeration and resource exhaustion. |
| `RATE_LIMIT_GLOBAL_IP` | backend | Optional | `600/60` | `lazy` | Per-IP ceiling across every HTTP endpoint. Too low throttles clients behind shared NAT. |
| `RATE_LIMIT_AUTH_CHALLENGE` | backend | Optional | `10/60` | `lazy` | Wallet challenge nonce issuance. |
| `RATE_LIMIT_AUTH_VERIFY` | backend | Optional | `5/60` | `lazy` | Signature verification attempts. Too high weakens brute-force protection. |
| `RATE_LIMIT_DEVICE_LINK_CHALLENGE` | backend | Optional | `10/60` | `lazy` | Device-link challenge issuance. |
| `RATE_LIMIT_DEVICE_LINK_VERIFY` | backend | Optional | `5/60` | `lazy` | Device-link verification attempts. |
| `RATE_LIMIT_KEY_BUNDLE` | backend | Optional | `30/60` | `lazy` | X3DH prekey bundle fetches. |
| `RATE_LIMIT_KEY_BUNDLE_DAILY` | backend | Optional | `200/86400` | `lazy` | Daily bundle quota. Too high allows draining one-time prekeys. |
| `RATE_LIMIT_UPLOAD_SLOT` | backend | Optional | `20/60` | `lazy` | Presigned upload slot requests. |
| `RATE_LIMIT_UPLOAD_BYTES_DAILY` | backend | Optional | `2147483648/86400` (2 GiB) | `lazy` | Daily upload volume per user. |
| `RATE_LIMIT_FILE_DOWNLOAD` | backend | Optional | `120/60` | `lazy` | Presigned download URL issuance. |
| `RATE_LIMIT_PUSH_SUBSCRIBE` | backend | Optional | `10/60` | `lazy` | Web-push subscription registration. |
| `RATE_LIMIT_SOCKET_DEFAULT` | backend | Optional | `10/1` | `lazy` | Any socket event without its own bucket. |
| `RATE_LIMIT_SOCKET_SEND_MESSAGE` | backend | Optional | `30/10` | `lazy` | `send_message` and `send_file_message`. |
| `RATE_LIMIT_SOCKET_TYPING` | backend | Optional | `20/5` | `lazy` | `typing_start` and `typing_stop`. |
| `RATE_LIMIT_SOCKET_ASK_ASSISTANT` | backend | Optional | `5/60` | `lazy` | AI assistant invocations, which drive the OpenAI bill. |
| `SOCKET_RATE_LIMIT_PER_SEC` | backend | Optional | none | `lazy` | Legacy setting. Used only when `RATE_LIMIT_SOCKET_DEFAULT` is unset, and sets that bucket to `<n>/1`. |
| **Logging** | | | | | |
| `LOG_LEVEL` | backend | Optional | `info` | `module` | **Not in `.env.example`.** An unknown level makes pino throw at import, which **crashes on start**. |
| **AI agent** | | | | | |
| `OPENAI_API_KEY` 🔒 | ai_agent | **Yes** (for AI endpoints) | none | `lazy` (per request) | Listed in `.env.example` under "AI Service", but **only the AI agent reads it**; the backend does not. The agent does not load `.env` files, so the key must be in its process environment. If unset or invalid, the service still starts and `/health` is fine, but `/chat`, `/proposals/summarise` and sub-threshold `/transfers/analyse` return 500, and `/index/message` and `/search` return 503. A leak lets others spend on your OpenAI account. The agent has no other settings: its port (`8000`) and Weaviate (`connect_to_local()`, `localhost:8080`) are hard-coded. |
| **Web client** (all 🌐 public, `build`) | | | | | |
| `NEXT_PUBLIC_API_URL` 🌐 | web | Required outside local dev | **`http://localhost:4000`** in `lib/api.ts`, but `http://localhost:3001` in `NewConversationModal.tsx` | `build` | The two fallbacks disagree, and the backend defaults to port 3001, so **leaving it unset breaks most REST calls in local dev**. In production it must be the `https://` API origin. |
| `NEXT_PUBLIC_SOCKET_URL` 🌐 | web | Required outside local dev | `http://localhost:3001` | `build` | Socket never connects. Precedence is inconsistent: `hooks/useSocket.ts` prefers this variable over `NEXT_PUBLIC_BACKEND_URL`, while `lib/socket.ts` prefers `NEXT_PUBLIC_BACKEND_URL`. Set both to the same value. It must be `https://` when TLS is enforced. |
| `NEXT_PUBLIC_BACKEND_URL` 🌐 | web | Required outside local dev | `http://localhost:3001` | `build` | See `NEXT_PUBLIC_SOCKET_URL`. |
| `NEXT_PUBLIC_SOROBAN_RPC_URL` 🌐 | web | Optional | `https://soroban-testnet.stellar.org` | `build` | If unset in production, transfers are built against testnet. |
| `NEXT_PUBLIC_NETWORK_PASSPHRASE` 🌐 | web | Optional | `Networks.TESTNET` | `build` | If it does not match the RPC's network, Freighter-signed transactions are rejected. |
| `NEXT_PUBLIC_TOKEN_TRANSFER_CONTRACT` 🌐 | web | **Yes** for transfers | `'REPLACE_WITH_TOKEN_TRANSFER_CONTRACT_ID'` | `build` | If unset, every in-chat transfer fails. Must equal the backend's `TOKEN_TRANSFER_CONTRACT_ID`, or the backend listener never sees the transfer. |
| `NEXT_PUBLIC_NETWORK` 🌐 | web | Optional | `test` | `build` | If wrong, transfer cards link to the wrong network on the Stellar explorer. |
| `NEXT_PUBLIC_AUTH_TOKEN` 🌐 | web | **Never set outside local dev** | none | `build` | A fallback session token for when none is stored. Because it is compiled into the bundle, **every visitor receives this token and is signed in as that user**. |
| `NEXT_PUBLIC_VAPID_PUBLIC_KEY` 🌐 | web | — | `''` | `build` (`next.config.ts`) | Effectively unused: declared in `next.config.ts`, but no code reads it. The web app fetches the key from `GET /push/vapid-public-key` at runtime (#349). |

## Secrets

These values are credentials: 🔒 `JWT_SECRET`, `DATABASE_URL`, `REDIS_URL`
(when it embeds a password), `OBJECT_STORE_ACCESS_KEY`,
`OBJECT_STORE_SECRET_KEY`, `VAPID_PRIVATE_KEY`, `OPENAI_API_KEY`, and the
unused `S3_SECRET_ACCESS_KEY`.

- **Never commit them.** `.env` is ignored by the root `.gitignore`, and
  `.env*` by `apps/web/.gitignore`. Only `.env.example` is tracked, and it must
  hold placeholders or local-only values.
- The `OBJECT_STORE_*` values in `.env.example` (`clicked` / `clickedsecret`)
  match the local MinIO in `infra/docker-compose.yml`. They are for local use
  only and must never be reused anywhere reachable.
- Never give a secret a `NEXT_PUBLIC_` prefix. That publishes it (see below).
- If a secret is committed or leaked, rotate it; deleting the commit is not
  enough. Rotating `JWT_SECRET` signs out every user.

## `NEXT_PUBLIC_*` values are public

Next.js replaces every `process.env.NEXT_PUBLIC_*` reference with its literal
value at `next build`. The value ends up in the JavaScript sent to every
browser, so anyone can read it with view-source. It follows that:

- Every `NEXT_PUBLIC_*` variable above is **public**. None of them may hold a
  secret. Most are harmless (URLs, network passphrase, contract ID), but
  `NEXT_PUBLIC_AUTH_TOKEN` is a live credential and must never be set for a
  shared or deployed build.
- Changing one requires a **rebuild**. Restarting `next start` is not enough.
- Next.js loads `.env*` files from `apps/web/`, not the repo root. The root
  `.env.example` does not list any web variables.
