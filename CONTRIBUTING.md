# Contributing to clicked

## Pull requests must target `dev`

**Open every pull request against `dev`, not `main`.**

If you open a PR against `main` and you are not the repository maintainer (or a collaborator
with admin/maintain permission), the `guard-main-branch.yml` workflow automatically:

1. Posts a comment on the PR explaining that PRs must target `dev`.
2. Closes the PR (`state: closed`).

**This is not a rejection of your work.** It is an automated branch-policy check, not a review
outcome — nobody looked at your code and decided against it. To continue: open a new PR with
the same branch against `dev` (or retarget the base branch on the existing PR and reopen it,
per the bot's comment — GitHub's base-branch editing UI is available on an open or reopened
PR; once the workflow has already closed it, opening a fresh PR against `dev` is the reliable
path).

`dev` is the integration branch contributors merge into; `main` is reserved for the
maintainer. This is why the `close-linked-issues.yml` workflow (below) exists at all — GitHub's
built-in "Closes #N" auto-close only fires on merges to the *default* branch, and this repo's
default branch is `main`, not `dev`.

## Forking and branching

```bash
# 1. Fork the repo on GitHub, then clone your fork
git clone https://github.com/<your-username>/clicked.git
cd clicked

# 2. Add the upstream repo so you can pull in new dev commits later
git remote add upstream https://github.com/codebestia/clicked.git

# 3. Make sure you're branching from an up-to-date dev
git fetch upstream
git checkout -b your-feature-branch upstream/dev

# 4. ...make your changes, keeping the branch focused on one concern...

# 5. Push to your fork
git push origin your-feature-branch

# 6. Open a PR: base = codebestia/clicked:dev, compare = <your-username>/clicked:your-feature-branch
```

Keep unrelated changes out of the branch — a docs fix and a feature change belong in separate
PRs so each can be reviewed and merged independently.

## Commit conventions

This repository follows [Conventional Commits](https://www.conventionalcommits.org/), with an
optional scope naming the area touched. Examples consistent with the existing commit history:

```
feat: add protection workflow
feat(backend): lock in Signal-protocol server invariants with guards and tests
feat(web): session reset and re-handshake on peer identity key change
fix(messages): apply shared payload validation to file messages
fix(conversations): serialize list preview to prevent plaintext leaks
docs: add testing strategy, migration workflow, and security policy
docs(backend): add background GC jobs reference
docs(web): add frontend accessibility guide
docs(contracts): add contract testing guide
```

- **Format:** `<type>(<optional-scope>): <description>`.
- **Common types:** `feat`, `fix`, `docs` (also seen in history: `style`, `refactor`, `chore`,
  `test`).
- **Scopes in use:** `backend`, `web`, `contracts`, `ai_agent`, `messages`, `conversations`,
  `security`, among others — pick the one matching the app or area your change touches, or omit
  the scope for changes that span multiple areas.
- Write the description in the imperative, present tense (`add`, not `added`/`adds`), matching
  the pattern above.

## Linking issues

If your PR closes an issue, reference it with a closing keyword directly in the PR title or
body: `Closes #123`, `Fixes #123`, or `Resolves #123` (case-insensitive; `-s`/`-es`/`-d`/`-ed`
suffixes are all recognized).

When your PR merges into `dev`, the `close-linked-issues.yml` workflow scans the PR title and
body for that pattern and closes each matched issue, with a comment
(`Closed by #<PR>, merged into dev.`) — this replaces GitHub's native auto-close, which only
triggers on merges to the default branch (`main`).

**Limitation to know about:** the workflow's matching regex requires the closing keyword
immediately before each issue number. `Closes #123, #124` only closes `#123` — `#124` has no
keyword directly in front of it and will be left open. If your PR closes multiple issues, repeat
the keyword for each: `Closes #123, Closes #124`.

## PR template

Every PR uses the template at
[`.github/pull_request_template.md`](.github/pull_request_template.md), which asks for:

- A **description** of the change.
- The **type of change** (bug fix / new feature / documentation update / other).
- A **checklist**: contributing guidelines read, changes tested locally, code follows the
  project's coding standards.

Fill in the description and check off the boxes that genuinely apply — an unchecked box is a
signal to the reviewer, not just a formality.

## CI requirements

CI is split by area, using GitHub Actions `paths` filters — a workflow only runs when a PR
touches the paths it's scoped to. Two checks run on every PR regardless of what changed.

### Repository-wide (always run)

- **PR Check** (`pr.yml`) — installs dependencies and runs `pnpm run lint` at the repo root.
- **Security CI** (`security-ci.yml`) — runs on every PR (and every push to `main`):
  - `regression`: the ciphertext-only guard and secret-field scan
    (`apps/backend/src/__tests__/security.regression.test.ts`).
  - `dependency-audit`: `pnpm audit` scoped to crypto-relevant backend dependencies (`ioredis`,
    `jsonwebtoken`, `web-push`, `@stellar/stellar-sdk`, `drizzle-orm`, `socket.io`).

### Backend (`apps/backend/**`)

`backend-ci.yml` spins up Postgres, Redis, and MinIO containers, runs migrations, then:
`pnpm format:check`, `pnpm lint`, `pnpm test`. (The suite's actual tests don't dial into those
containers — see [`apps/backend/docs/testing.md`](apps/backend/docs/testing.md) — the
containers exist so CI can validate migrations against a real Postgres.)

### Web (`apps/web/**`)

`frontend-ci.yml`: `pnpm --filter web lint`, `pnpm --filter web test`,
`pnpm --filter web build`.

### AI agent (`apps/ai_agent/**`)

`ai-agent-ci.yml`, three jobs: `ruff check` + `ruff format --check` (lint), `mypy main.py`
(typecheck), and `pytest --cov=main` (test + coverage, uploaded to Codecov).

### Contracts (`contracts/**`)

`contracts-ci.yml`, matrixed over the `token_transfer`, `group_treasury`, and `proposals`
packages: `cargo test -p <package>`, then `cargo build -p <package> --target
wasm32-unknown-unknown --release` with a 100 KB per-WASM size gate reported as a PR comment.
A separate `clippy` job runs `cargo clippy --workspace --target wasm32-unknown-unknown -- -D
warnings` (with `dead_code` and `too-many-arguments` allowed), and an `audit` job runs
`cargo audit`. This workflow also runs on a weekly schedule independent of any PR.

### Running checks locally

Match the workflow's own commands where practical: `pnpm lint` / `pnpm format:check` /
`pnpm test` for the app you touched, or `cargo test -p <package>` /
`cargo clippy --workspace --target wasm32-unknown-unknown` for contracts. Running the same
commands the workflow runs, in the affected app's directory, is the most reliable way to know
CI will pass before you push.

---

See also the top-level [documentation index](docs/README.md) for architecture and per-app
references.
