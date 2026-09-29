# Local Object Store (development/test)

Source: [`src/lib/localObjectStore.ts`](../src/lib/localObjectStore.ts),
[`src/lib/objectStore.ts`](../src/lib/objectStore.ts),
[`src/routes/localStorage.ts`](../src/routes/localStorage.ts),
[`src/lib/storage.ts`](../src/lib/storage.ts),
[`src/routes/uploads.ts`](../src/routes/uploads.ts).

See also: [File Uploads API](api-files-uploads.md) for the upload/confirm route contract this
store backs, and the [backend testing guide](testing.md) /
[cross-app testing strategy](../../../docs/testing.md) for the no-Docker-in-tests policy this
store exists to satisfy.

## `ObjectStoreLike` — the shared interface

Both the production `ObjectStore` (S3-compatible, via `@aws-sdk/client-s3`) and the
development-only `LocalDiskObjectStore` implement the same `ObjectStoreLike` interface, defined
in `objectStore.ts`:

```ts
export interface ObjectStoreLike {
  putObject(key: string, body: ..., contentType?: string): Promise<void>;
  getObject(key: string): Promise<unknown>;
  deleteObject(key: string): Promise<void>;
  headObject(key: string): Promise<{ exists: boolean; size?: number }>;
  getPresignedPutUrl(key: string, contentType: string | undefined, ttlSeconds: number): Promise<string>;
  getPresignedGetUrl(key: string, ttlSeconds: number): Promise<string>;
}
```

**Call sites only ever depend on `ObjectStoreLike`** (`getObjectStore()`'s return type,
imported by `routes/uploads.ts` and `lib/storage.ts`); nothing in the application imports
`ObjectStore` or `LocalDiskObjectStore` directly except the factory code itself. The backend
selected underneath is swapped without call-site changes — verified below.

| Method | Purpose | Local (`LocalDiskObjectStore`) behavior | Production (`ObjectStore`) behavior |
|---|---|---|---|
| `putObject(key, body, contentType?)` | Store bytes at `key`. | Writes the body to a file under the local storage root, plus a sidecar `<key>.meta.json` recording `contentType`. Creates parent directories as needed (`mkdir(..., { recursive: true })`). | `PutObjectCommand` against the S3-compatible bucket. |
| `getObject(key)` | Retrieve an object. | Reads the file and its sidecar meta file; returns `{ Body: Buffer, ContentType? }`. Throws (propagates the `fs` error) if the file doesn't exist. | `GetObjectCommand`; returns the raw SDK response. The two implementations' return shapes are intentionally left loosely typed (`unknown`) because nothing in the codebase consumes `getObject()` polymorphically today — callers that use it know which implementation they're calling. |
| `deleteObject(key)` | Remove an object. | `rm` on both the object file and its sidecar meta file, with `{ force: true }` (no error if already absent). | `DeleteObjectCommand`. |
| `headObject(key)` | Existence + size check, **used by upload-confirm verification** (`routes/uploads.ts`, `#356`). | `stat()`s the resolved path; returns `{ exists: true, size }` on success, `{ exists: false }` on any error (e.g. `ENOENT`). | `HeadObjectCommand`; maps `NotFound` / `NoSuchKey` names or a `404` status code to `{ exists: false }`, otherwise returns `{ exists: true, size: ContentLength }` (omitting `size` if the SDK didn't return `ContentLength`). Any other error is rethrown rather than swallowed. |
| `getPresignedPutUrl(key, contentType, ttlSeconds)` | Issue a time-limited upload URL. | Builds a URL back to this backend's own `/local-storage/<key>` route (see below) with an HMAC signature and an `expires` timestamp `ttlSeconds` seconds out. `contentType` is accepted for interface parity but not used to constrain the signature. | Real S3 presigned URL via `getSignedUrl` from `@aws-sdk/s3-request-presigner`, expiring after `ttlSeconds`. |
| `getPresignedGetUrl(key, ttlSeconds)` | Issue a time-limited download URL. | Same local-URL scheme as the PUT case, signed for `GET`. | Real S3 presigned GET URL. |

`headObject` matters specifically because `routes/uploads.ts`'s upload-confirm handler calls it
to prove an object actually landed in the store — at the expected size — before marking a file
row `ready`; both implementations must answer this the same way for that check to behave
identically in dev and production.

## Local filesystem behavior

- **Root directory:** `process.env.LOCAL_STORAGE_DIR`, falling back to `<cwd>/.local-storage`
  (`DEFAULT_ROOT_DIR` in `localObjectStore.ts`). This directory is expected to be gitignored;
  this document does not assume any specific absolute path beyond that default.
- **Key → path mapping:** `resolvePath(key)` joins the root with `key` via `node:path`'s
  `join`/`normalize`, then verifies the resolved path is still inside the root
  (`resolved === root || resolved.startsWith(root + sep)`); otherwise it throws
  `"Invalid storage key: <key>"`. This is the path-traversal guard — a key like
  `../../etc/passwd` cannot escape the root directory.
- **Directory creation:** `putObject` creates any missing parent directories with
  `mkdir(dirname(path), { recursive: true })` before writing, so nested keys (e.g.
  `uploads/<conversationId>/<hash>`, the shape `generateStorageKey` in `lib/storage.ts`
  produces) work without a separate provisioning step.
- **Metadata persistence:** yes, but only `contentType`, and only in a sidecar file
  `<resolved-path>.meta.json` next to the object (`metaPath(key)`), written by `putObject` and
  read back by `getObject`. There is no metadata store beyond this — no size, no timestamps,
  no arbitrary user metadata.
- **Missing objects:** `getObject` and `putObject`'s underlying `fs` calls throw/reject on a
  missing file; `headObject` and the exported test helper `exists()` instead catch the error
  and return `false`/`{ exists: false }`. Callers that need a non-throwing existence check use
  `headObject`.
- **Concurrent writes:** no special handling — `writeFile` is used as-is, with no locking,
  temp-file-then-rename, or write queuing. Two concurrent `putObject` calls to the same key race
  at the filesystem level exactly as plain `fs.writeFile` calls would.

## Signed local URLs

Format (both PUT and GET use the same shape, differing only in HTTP method and which key each
verifies against):

```
http://localhost:<PORT>/local-storage/<key>?expires=<unix-seconds>&sig=<hex-hmac>
```

- **Base URL:** `process.env.STORAGE_ENDPOINT` if set (trailing slashes stripped), otherwise
  `http://localhost:${process.env.PORT ?? 3001}/local-storage`.
- **Object identifier:** the storage `key`, appended as the path segment(s) after
  `/local-storage/` (e.g. `uploads/<conversationId>/<hash>`).
- **`expires`:** a Unix timestamp in seconds — `Math.floor(Date.now() / 1000) + ttlSeconds` at
  signing time.
- **`sig`:** `HMAC-SHA256(processSecret, "<METHOD>:<key>:<expires>")`, hex-encoded.
  `processSecret` is `randomBytes(32)` generated once per process at module load — it is not
  persisted or shared across processes, which is fine because the same process that signs a URL
  is the one that verifies it later in the same dev/test run.

Example (placeholders, not a real signature):

```
http://localhost:3001/local-storage/uploads/abc123/def456?expires=1780000000&sig=4f3c...e9a1
```

### Validation on the serving route

`routes/localStorage.ts` mounts `PUT` and `GET` handlers on `/*splat` under `/local-storage`
(only outside production — see below). Both call `checkSignature(req, res, method)`, which:

1. **Object/key:** extracts the key from the wildcard route params (`req.params['splat']`,
   joined with `/` if Express gives it as an array).
2. **Signature:** reads `sig` from the query string and calls
   `verifySignedRequest(method, key, expires, sig)` in `localObjectStore.ts`, which
   recomputes the expected HMAC for `(method, key, expires)` and compares it to the supplied
   signature using `timingSafeEqual` (constant-time, after confirming both buffers are the same
   length — a length mismatch is treated as a non-match rather than passed to
   `timingSafeEqual`, which throws on unequal lengths).
3. **Expiry:** `verifySignedRequest` also checks `Number.isFinite(expires)` and rejects if
   `Date.now() / 1000 > expires`.
4. **No other validation** is performed by this route beyond signature + expiry — there is no
   separate auth/session check, which is intentional: the route is mounted outside
   `requireAuth` (see `app.ts`) because a valid signed URL is meant to be the credential, the
   same way a real S3 presigned URL is.

On failure, the route responds `403 { error: 'Invalid or expired signed URL' }` without
distinguishing "bad signature" from "expired" from "missing key/sig" in the response body.

The `PUT` handler reads the raw body (`express.raw({ type: () => true, limit: '150mb' })`) and
the `content-type` header, then calls `getLocalObjectStore().putObject(key, body, contentType)`.
The `GET` handler calls `getObject(key)` and streams back `Body` with `Content-Type` set from
the stored metadata if present, returning `404 { error: 'Object not found' }` on any error
(e.g. the file doesn't exist).

## Production/development selection

Two independent switches exist in the codebase, both keyed on `process.env.NODE_ENV`:

- **`objectStore.ts`'s `getObjectStore()`** (used by `routes/uploads.ts` for `headObject`, and
  elsewhere for delete/head operations): a lazily-constructed singleton that picks
  `createObjectStore(loadEnv())` (real S3 client) when `NODE_ENV === 'production'`, otherwise
  `getLocalObjectStore()`. `resetObjectStoreForTests()` clears the singleton so tests can
  change env and re-resolve it.
- **`lib/storage.ts`'s `generatePresignedPut` / `generatePresignedGet`**: each has its own
  `isProduction()` check (`process.env.NODE_ENV === 'production'`) and calls
  `getLocalObjectStore()` directly outside production, or `getObjectStore()` in production. This
  is a second, independent branch rather than routing through `getObjectStore()`'s own switch —
  both land on the same backend selection, but it's worth knowing these are two separate
  `NODE_ENV` checks in the source rather than one shared decision point.
- **Route mounting:** `app.ts` only mounts `localStorageRouter` at `/local-storage` when
  `process.env['NODE_ENV'] !== 'production'` — in production the route doesn't exist at all, so
  a presigned local URL could never be dereferenced even if one were somehow issued.

**Call-site agnosticism, verified:** `routes/uploads.ts` calls `getObjectStore().headObject(...)`
without any environment branching of its own — it only ever sees `ObjectStoreLike`. The
environment switch lives entirely in the two factory functions above, not in application code
that consumes the store.

## Test architecture: why this exists

The [cross-app testing strategy](../../../docs/testing.md) states the project-wide rule
directly: **no test in the repository may require Redis, Postgres, or an S3/MinIO server to be
running.** `LocalDiskObjectStore` is what makes that rule achievable for object-store-dependent
code: `getObjectStore()` resolves to it automatically whenever `NODE_ENV !== 'production'`
(the default for `vitest`), so upload/presign/download/head flows exercise a real, working
implementation — actual bytes on disk, actual signature verification — without a Docker
dependency, on a fresh clone, with Docker stopped.

This does **not** mean every backend test is Docker-free for the same reason: the same
cross-app testing document also states that Postgres access goes through a hand-built `db`
mock rather than a real database in tests. `LocalDiskObjectStore` specifically replaces the
need for a MinIO/S3 dependency; it makes no claim about, and has no bearing on, how other
external dependencies (Postgres, Redis) are avoided in the test suite. Separately, the backend
CI workflow (`backend-ci.yml`) does start real Postgres, Redis, and MinIO containers — but per
the testing doc, that's to validate migrations against a real Postgres, not because the test
suite itself dials into any of those services.
