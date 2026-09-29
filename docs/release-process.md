# Release Process

How a change gets from a contributor's pull request to production, for all
four deployables:

- the backend service (`apps/backend`)
- the Next.js web app (`apps/web`)
- the AI agent (`apps/ai_agent`)
- the Soroban contracts (`contracts/`)

For checks after a deploy and for incident handling, use the
[operator runbook](runbook.md). For every setting mentioned here, see
[environment variables](environment-variables.md).

## What is automated today

Be clear about this before reading further: **the repository contains no
deploy automation.** CI verifies code; people deploy it.

| Stage | Automated? | Where |
| --- | --- | --- |
| Lint, test, build on every PR and push | Yes | `.github/workflows/*-ci.yml`, filtered by app path |
| Security regression and crypto-dependency CVE audit | Yes, on PRs and on push to `main` | `security-ci.yml` |
| Closing PRs to `main` that a non-maintainer opened | Yes | `guard-main-branch.yml` |
| Closing linked issues when a PR merges to `dev` | Yes | `close-linked-issues.yml` |
| Version bumps, tags, changelog | **No** | Manual (see [Versioning](#versioning)) |
| Database migrations | **No** | Manual (`pnpm --filter backend db:migrate`) |
| Deploying the backend, web app or AI agent | **No** | Manual. There are no Dockerfiles or hosting configs in the repo. |
| Contract deploys | **No** | Manual scripts in `contracts/scripts/`, testnet only |

## Branch flow: contributor PR → `dev` → `main`

```
contributor fork ──PR──▶ dev ──(maintainer PR)──▶ main ──▶ deploy
```

1. **A contributor opens a PR against `dev`.** PRs against `main` from anyone
   other than the maintainer are closed automatically by
   `guard-main-branch.yml`, with a comment asking the author to retarget `dev`.
2. **CI must pass.** The workflows that run depend on the paths the PR touches:
   Backend CI, Frontend CI, AI Agent CI, Contracts CI, Security CI, and the
   repo-wide lint in `pr.yml`.
3. **A maintainer reviews and merges into `dev`.** Any `Closes #N` / `Fixes #N`
   issues in the PR are closed at that point by `close-linked-issues.yml`,
   because GitHub only auto-closes them on merges to the default branch.
4. **The maintainer promotes `dev` to `main`.** They open a PR from `dev` into
   `main`, wait for CI on the combined changes, and merge it. **Only the
   maintainer merges to `main`.** In this repo, "maintainer" means the repo
   owner or a collaborator with `admin` or `maintain` permission, which is the
   same check `guard-main-branch.yml` uses.
5. **A merge to `main` is the release candidate.** Nothing deploys
   automatically. The maintainer tags the release and runs the
   [release checklist](#release-checklist) below.

### Gaps to close before this flow is real

- **There is no `dev` branch on `origin` yet.** At the time of writing, only
  `main` (plus feature branches) exists, and recent contributor PRs were merged
  straight into `main`. Create `dev` from `main` and make it the base for new
  contributor PRs.
- **The guard only checks who *opened* a PR, not who *merges* it.** It closes
  non-maintainer PRs to `main`, but it does not stop a collaborator with write
  access from merging a PR that the maintainer opened. To enforce "only the
  maintainer merges to `main`", add a branch-protection rule (or ruleset) on
  `main` that restricts who can push and merge to the maintainer, and requires
  the CI checks to pass.

## Versioning

**Today:** there are no git tags, no changelog, and no release automation. The
package versions have never been bumped: backend `1.0.0`, web `0.1.0`,
AI agent `0.1.0`, and the contracts have no versions. The backend reports its
`package.json` version from `GET /health`, so that value is what operators see.

**Convention from now on:**

- **One tag per promotion to `main`**, in the form `vMAJOR.MINOR.PATCH` (SemVer),
  on the merge commit of the `dev → main` PR. The whole monorepo shares one
  version, because the apps are deployed together and depend on each other's
  API.
- **How to choose the bump:**
  - **MAJOR** for any change that breaks a deployed client or peer: a
    WebSocket event or REST contract change that old web or mobile clients
    cannot handle, a destructive database migration, or a contract redeploy
    that changes a contract ID.
  - **MINOR** for new features and additive migrations.
  - **PATCH** for fixes only.
- **Bump `apps/backend/package.json` to the tag version** in the promotion PR,
  so that `/health` reports what is actually running. Bumping the web and
  AI agent versions to match is optional.
- **Write release notes in the GitHub Release for the tag.** List the merged PRs,
  migrations, new or changed environment variables, and any contract ID changes.

## Release checklist

Deploy in this order. Each step depends on the ones before it:

1. **Contracts**, only if the release changes them. Contract IDs feed the
   backend's environment and are baked into the web build. See
   [Soroban contracts](#soroban-contracts).
2. **Database migrations.**
3. **Backend.**
4. **AI agent.** It is independent of the others and can go at any point
   after step 3.
5. **Web app.** It goes last because it calls the new backend API and has
   contract IDs compiled in.
6. **Verify.** See [Post-deploy verification](#post-deploy-verification).

## Backend service

### Migrations and their order relative to the rollout

Migrations live in `apps/backend/drizzle/` and are applied with
`drizzle-kit migrate`. **Run them before rolling out the new backend code**, and
write them so that the *previous* backend version still works against the
migrated schema:

- The gateway runs as several instances coordinated through Redis (see
  [runbook → Scaling gateways](runbook.md#scaling-gateways)). During a rolling
  deploy, old and new instances serve traffic against the same database at the
  same time.
- **Additive changes go in the same release as the code.** Examples: a new
  table, a new nullable column, a new index. Migrate first, then roll out.
- **Destructive changes need two releases** (expand, then contract). Examples:
  dropping or renaming a column, adding `NOT NULL` without a default. Release
  *N* stops reading and writing the column. Release *N+1* ships the migration
  that removes it, once no running instance uses it.
- **Never run `db:push` against a shared database.** It diffs the schema and
  applies changes directly, can drop columns, and records no migration.

```bash
# From the repo root, with the target database in the environment.
# drizzle.config.ts reads DATABASE_URL from process.env; if it is unset,
# the URL is an empty string.
export DATABASE_URL='postgres://…'
pnpm --filter backend db:generate   # only when authoring: commit the generated SQL in the PR
pnpm --filter backend db:migrate    # at release time: applies pending migrations
```

Take a database backup or snapshot immediately before `db:migrate`. It is the
only rollback path for a migration (see below).

### Deploy

```bash
pnpm install --frozen-lockfile
pnpm --filter backend build          # tsc → apps/backend/dist
NODE_ENV=production pnpm --filter backend start   # node dist/index.js
```

- **Set configuration through the real process environment, not a `.env`
  file.** The backend loads `.env` after its modules have read their settings,
  so values that exist only in `.env` are silently ignored. `JWT_SECRET` then
  falls back to `'test-secret'`. See the caveat in
  [environment-variables.md](environment-variables.md#caveat-the-backend-loads-env-too-late-for-module-reads).
- **`NODE_ENV=production` is required.** Without it, the backend uses the
  local-disk object store and turns TLS enforcement off (unless `APP_ENV` is
  set).
- **Roll instances one at a time with `SIGTERM`.** The shutdown handler keeps
  presence intact for clients that reconnect to another instance (see
  [runbook → Scaling gateways](runbook.md#scaling-gateways)). Sticky sessions
  are not required.
- If the startup check in `config.ts` fails, the process exits with code 1 and
  lists the missing or invalid variables. Treat that as a failed deploy and
  keep the old instances running.

### Rollback

- **Code: yes.** Redeploy the previous tag's build. This is safe **only if**
  that release's migrations were additive, which the expand/contract rule
  above guarantees.
- **Migrations: no automated rollback.** `drizzle-kit` has no down-migration
  runner, and the repo contains no down scripts.
  `apps/backend/docs/message-encryption-migration.md` refers to
  `drizzle/rollback/0003_…down.sql`, but that file does not exist. If a
  migration must be undone, the options are:
  1. Write and ship a forward migration that reverses it (preferred).
  2. Restore the pre-migration backup. **This loses every write since the
     backup**, including messages and prekey consumption.
- **The Stellar listener's position is lost on restart.** The listener keeps
  its event cursor in memory only. After a deploy or rollback, check that
  transfer and treasury events are still arriving (see
  [Post-deploy verification](#post-deploy-verification)).

## Next.js web app

### Deploy

```bash
pnpm install --frozen-lockfile
# NEXT_PUBLIC_* values are compiled into the bundle at build time
pnpm --filter web build
pnpm --filter web start          # or deploy the build output to your host
```

- **Every `NEXT_PUBLIC_*` value is fixed at build time.** This includes the
  API and socket URLs and `NEXT_PUBLIC_TOKEN_TRANSFER_CONTRACT`. Changing one
  means **rebuilding**, not restarting. Put the values in `apps/web/.env.production`
  or the host's build environment. They must not come from the repo root.
- Never set `NEXT_PUBLIC_AUTH_TOKEN` for a deployed build. It would be sent to
  every visitor.
- The repo has no hosting configuration. `apps/web/README.md` only contains
  the default Next.js "Deploy on Vercel" note.

### Rollback

- **Yes, but only as good as your build artifacts.** If the host keeps previous
  builds (for example, Vercel's instant rollback), promote the previous build.
- If you have to rebuild an old tag instead, you must rebuild with **the same
  `NEXT_PUBLIC_*` values** that tag was built with. After a contract redeploy,
  the old build points at the old contract ID. That is only correct if you
  are also rolling the contract ID back in the backend.
- Browsers that already loaded the new bundle keep running it until they
  reload.

## AI agent

### Deploy

```bash
cd apps/ai_agent
uv sync --frozen            # installs from uv.lock
OPENAI_API_KEY=… uv run python main.py    # uvicorn on 0.0.0.0:8000 (hard-coded)
```

- The only setting is `OPENAI_API_KEY`, and it must be in the process
  environment because the agent does not read `.env` files.
- The agent connects to Weaviate via `connect_to_local()`, which is hard-coded
  to `localhost:8080`, so Weaviate must run on the same host or network
  namespace.
- `/health` only confirms that the process is up. It does not check OpenAI
  or Weaviate.

### Rollback

- **Code: yes.** The service is stateless. Redeploy the previous tag.
- **Indexed data: no.** Message vectors in Weaviate's `Message` collection are
  not versioned. A release that changes the embedding model
  (`text-embedding-3-small`) makes existing vectors incompatible, and rolling
  the code back does not un-index them. Such a release needs a planned
  re-index and should be treated as MAJOR.

## Soroban contracts

Contract deploys are **a separate path with no rollback**. Treat them as their
own change, reviewed and scheduled separately from app releases. The
step-by-step commands are in the
[contract build, deployment & invocation guide](../contracts/docs/api-deployment-invocation.md).
This section covers how contract deploys fit into a release.

### How a deploy works

Each `contracts/scripts/deploy_*.sh` script builds the WASM, uploads it,
**creates a new contract instance**, and calls `initialize`. Run the scripts
from `contracts/` (see guide §4.4). The `make deploy-contracts` target runs
them from the repo root, where the relative `cargo build` and WASM paths do not
resolve. It also skips `proposals`.

- **Every run produces a new contract ID with empty state.** Balances,
  treasury members, proposals and votes do not move across. Funds held by the
  old `group_treasury` instance stay in the old instance.
- **The scripts are hard-coded to `NETWORK="testnet"`.** There is no scripted
  mainnet path. A mainnet deploy needs its own reviewed procedure.
- **New IDs must be propagated by hand**, in this order:
  1. Backend environment: `TOKEN_TRANSFER_CONTRACT_ID`,
     `GROUP_TREASURY_CONTRACT_ID`, and `STELLAR_RPC_URL` (not `RPC_URL`). Then
     restart the backend.
  2. Web build environment: `NEXT_PUBLIC_TOKEN_TRANSFER_CONTRACT`. Then
     **rebuild** the web app.

  If the two sides disagree, the backend listener watches one contract while
  users transact on another (guide §6).

### Upgrading in place

Only `token_transfer` exposes `upgrade(new_wasm_hash)`. It is admin-only and
keeps the same contract ID and storage (guide §5.1). `group_treasury` and
`proposals` have no upgrade entry point. **The only way to change their code is
to deploy a new instance**, with all the consequences above.

### Rollback: there is none

- Contract transactions are final. Transfers, deposits, withdrawals and votes
  executed against a bad contract cannot be reverted.
- For `token_transfer`, you can call `upgrade` with the previous WASM hash.
  That is a *roll forward to old code*, not a rollback: storage the new code
  wrote stays as written. If the bad WASM breaks `upgrade` itself or the
  admin key check, the contract cannot be changed again.
- For `group_treasury` and `proposals`, "rolling back" means pointing the
  apps at the old contract ID again. That works only if the old instance is
  still valid and nobody has moved state to the new one.
- Because of this, before any contract deploy that users depend on:
  - run Contracts CI and `cargo test` on the exact commit;
  - deploy to testnet first and exercise the flows in guide §5;
  - record the WASM hash, contract ID and deployer account in the release notes.

### Known script defects to fix before the next contract deploy

- `deploy_group_treasury.sh` calls `initialize` without the required
  `threshold` argument, so the invoke fails. **The instance is then deployed
  but uninitialised, and anyone can call `initialize` on it and become admin.**
  Do not publish that contract ID. Fix the script, or initialise the instance
  immediately by hand.
- `deploy_proposals.sh` requires `TREASURY_CONTRACT_ID` and `MIN_VOTES`, but
  `initialize` only takes `admin`, so neither value is applied.

## Rollback summary

| App | Code rollback | State rollback | Notes |
| --- | --- | --- | --- |
| Backend | Yes: redeploy the previous tag | **No.** Forward-fix or restore a backup (loses writes since the backup). | Safe only if migrations followed expand/contract. |
| Web app | Yes: promote the previous build | n/a | Old builds keep old `NEXT_PUBLIC_*` values. Loaded tabs keep the new bundle until reload. |
| AI agent | Yes: redeploy the previous tag | **No** for Weaviate vectors | An embedding-model change needs a re-index. |
| Contracts | `token_transfer` only, via `upgrade` to an old hash | **None.** On-chain transactions are final. | `group_treasury` and `proposals` can only be replaced, not changed. |

## Post-deploy verification

Run these after every release. The [operator runbook](runbook.md) has the
diagnosis and recovery steps for anything that fails.

1. **Backend health.** `GET /health` returns `200` with `"db": "connected"` and
   the new `version` from each instance. A `503` or `"db": "unreachable"` means
   Postgres is unreachable. See
   [runbook → Storage outage (Postgres)](runbook.md#storage-outage-postgres).
2. **Redis adapter.** The boot logs show `[socket.io] Redis adapter attached`,
   not `Redis unavailable … single-instance mode`. See
   [runbook → Redis / message bus down](runbook.md#redis--message-bus-down).
3. **Object storage.** `/health` does not check it. Do a manual upload and
   download through the app, as described in
   [runbook → Rotating VAPID / storage credentials](runbook.md#rotating-vapid--storage-credentials).
   Also see
   [runbook → Storage outage (S3-compatible object store)](runbook.md#storage-outage-s3-compatible-object-store).
4. **Stellar listener.** The boot logs do *not* show
   `[stellar-listener] … listener disabled`, and a test transfer shows up in
   the chat.
5. **Metrics.** `GET /metrics` is being scraped, and the dashboards in
   [observability.md](observability.md) show no spike in errors, disconnects
   or backpressure after the rollout.
6. **Web app.** Sign in, open a conversation, send a message, and check that
   the socket connects over `wss://`.
7. **AI agent.** `GET /health` returns `ok`, and one `/chat` call returns a
   reply. That call is what proves `OPENAI_API_KEY` and outbound access work.
