# Code Style

This page covers:

- the formatters and linters in force for each language
- how to run them locally
- which ones CI blocks on
- the conventions no tool enforces, which reviewers are expected to check

There are no pre-commit hooks (no husky, lint-staged or pre-commit). Run
the commands below before pushing.

## Summary

| Language          | Tool                            | Config                                                                                           | Command                                                  | CI gate                                                              |
| ----------------- | ------------------------------- | ------------------------------------------------------------------------------------------------ | -------------------------------------------------------- | -------------------------------------------------------------------- |
| TS, TSX, MD, JSON | Prettier `3.9.1` (pinned)       | [`.prettierrc.json`](../.prettierrc.json), [`.prettierignore`](../.prettierignore)               | `pnpm format:check` / `pnpm format` (repo root)          | **Partial.** Only `apps/backend/src/**/*.ts` (Backend CI).           |
| TS (backend)      | ESLint 9 + `typescript-eslint`  | [`apps/backend/eslint.config.js`](../apps/backend/eslint.config.js)                              | `pnpm --filter backend lint`                             | **Errors only** (Backend CI, PR Check)                               |
| TS/TSX (web)      | ESLint 9 + `eslint-config-next` | [`apps/web/eslint.config.mjs`](../apps/web/eslint.config.mjs)                                    | `pnpm --filter web lint`                                 | **Errors only** (Frontend CI, PR Check)                              |
| TS type check     | `tsc` (`strict: true`)          | `apps/*/tsconfig.json`                                                                           | `pnpm --filter backend build`; `pnpm --filter web build` | **Web only.** `next build` type-checks; Backend CI never runs `tsc`. |
| Python            | ruff `0.15.20` (lint + format)  | [`apps/ai_agent/pyproject.toml`](../apps/ai_agent/pyproject.toml) `[tool.ruff]`                  | `uv run ruff check .` / `uv run ruff format --check .`   | **Yes** (AI Agent CI)                                                |
| Python            | mypy `2.1.0`                    | `pyproject.toml` `[tool.mypy]`                                                                   | `uv run mypy main.py`                                    | **Yes**, `main.py` only (AI Agent CI)                                |
| Rust              | rustfmt (default style)         | none; `rustfmt` component in [`contracts/rust-toolchain.toml`](../contracts/rust-toolchain.toml) | `cargo fmt --check` / `cargo fmt` (in `contracts/`)      | **No**                                                               |
| Rust              | clippy, `-D warnings`           | [`contracts/Cargo.toml`](../contracts/Cargo.toml) `[workspace.lints]` + CI flags                 | see [Rust](#rust-rustfmt--clippy)                        | **Yes**, zero warnings (Contracts CI)                                |

Python versions are pinned by `apps/ai_agent/uv.lock`. Prettier is pinned
exactly in both the root and backend `package.json`. The Rust toolchain is
**not** pinned: it tracks `stable`, both locally and in CI.

## Prettier

- **Pinned** to `3.9.1` (exact, no caret) in the root `package.json` and in
  `apps/backend/package.json`. A Prettier upgrade reformats code, so it gets a
  PR of its own and never rides along with other changes.
- **One shared config** at the repo root, `.prettierrc.json`, which every
  package inherits: `semi`, `singleQuote`, `trailingComma: all`,
  `printWidth: 100`, `tabWidth: 2`. Packages must not add their own Prettier
  config.
- **Commands:**
  - `pnpm format:check` and `pnpm format` at the root cover
    `**/*.{ts,tsx,md,json}`.
  - `pnpm --filter backend format:check` covers only `apps/backend/src/**/*.ts`.
    This is the only Prettier check CI runs.
- **CI gap:** web `.tsx`, Markdown and JSON are not format-checked in CI.
  Run the root `pnpm format:check` before opening a PR that touches them.

### `.prettierignore`: generated output is never formatted

**Prettier formats source that people edit. It never formats files a tool
generates.** Reformatting generated files produces noisy diffs that hide real
changes, and the next build or test run rewrites them anyway. Each ignore entry
follows from that rule:

| Entry                                                                 | Why it is ignored                                                                               |
| --------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------- |
| `node_modules/`, `dist/`, `.next/`, `.turbo/`, `coverage/`, `target/` | Dependency and build output                                                                     |
| `apps/backend/drizzle/meta/`                                          | drizzle-kit migration snapshots and `_journal.json`, rewritten by `db:generate`                 |
| `**/*.d.ts`                                                           | TypeScript declaration output (for example `drizzle.config.d.ts`)                               |
| `apps/web/next-env.d.ts`                                              | Next.js ambient types, regenerated by `next dev` and `next build`                               |
| `contracts/**/test_snapshots/`                                        | Soroban test snapshots, regenerated by `cargo test` (also gitignored in `contracts/.gitignore`) |
| `contracts/target/`, `apps/ai_agent/.venv/`                           | Rust and Python build output                                                                    |

When you add a tool that writes files into the tree, add its output to
`.prettierignore` in the same PR, under the "Generated artifacts" heading and
with a comment naming the tool. The generated SQL files in
`apps/backend/drizzle/*.sql` are not listed because the root glob does not
match `.sql`. Do not hand-edit them either.

## ESLint (TypeScript / TSX)

**Backend** ([`eslint.config.js`](../apps/backend/eslint.config.js)) uses the
flat config: `@eslint/js` recommended plus `typescript-eslint` recommended,
with these overrides:

- `@typescript-eslint/no-unused-vars`: **error**. Parameters prefixed with `_`
  are exempt, so name deliberately unused arguments `_req`, `_next` and so on.
- `@typescript-eslint/no-explicit-any`: **warn**.
- Node globals (`process`, `Buffer`, `NodeJS`, …) are declared explicitly,
  because `no-undef` cannot see `@types/node`.

**Web** ([`eslint.config.mjs`](../apps/web/eslint.config.mjs)) uses
`eslint-config-next` (`core-web-vitals` + `typescript`) with the default
Next.js ignores.

**Commands:** `pnpm --filter backend lint` and `pnpm --filter web lint` (add
`lint:fix` to auto-fix). `pnpm lint` at the root runs both through Turborepo.

**CI:** ESLint exits non-zero only on **errors**, so errors block the build and
warnings do not. The linters run in Backend CI, Frontend CI, and in `pr.yml`
(`pnpm run lint`) on every PR.

## Python: ruff + mypy

Config lives in [`apps/ai_agent/pyproject.toml`](../apps/ai_agent/pyproject.toml):

- **ruff:** `target-version = "py312"`, `line-length = 100` (the same as
  Prettier's `printWidth`), and lint rules `E`, `F`, `I` (import sorting) and
  `W`. `ruff format` is the formatter; there is no black or isort.
- **mypy:** `python_version = "3.12"`, `warn_return_any`,
  `warn_unused_configs`, `ignore_missing_imports`.

Run from `apps/ai_agent/`:

```bash
uv sync --group dev
uv run ruff check .            # lint (add --fix for safe fixes)
uv run ruff format --check .   # format check (drop --check to apply)
uv run mypy main.py            # type check
```

**CI:** all three block the build (AI Agent CI). mypy only checks `main.py`,
so `tests/` is not type-checked.

## Rust: rustfmt + clippy

Run from `contracts/`:

```bash
cargo fmt --check        # format check (drop --check to apply)
cargo clippy --workspace --target wasm32-unknown-unknown -- \
  -D warnings -A dead_code -A clippy::too-many-arguments   # exactly what CI runs
```

- **rustfmt** uses the default style (there is no `rustfmt.toml`). It is **not
  checked in CI**. Run `cargo fmt` before pushing.
- **clippy** runs with **`-D warnings`**, so any warning fails Contracts CI.
  Lint against the `wasm32-unknown-unknown` target, because that is how the
  contracts are built and warnings can differ from a host build.
- **The gate is looser than `-D warnings` suggests.** `[workspace.lints]` in
  `contracts/Cargo.toml` sets `dead_code`, `unused_variables`, `unused_mut`,
  `unused_imports` and `clippy::too_many_arguments` to `allow`. The CI command
  allows `dead_code` and `too_many_arguments` again. None of those ever fail
  the build.
- **The toolchain is `stable` and unpinned** (`rust-toolchain.toml` and
  `dtolnay/rust-toolchain@stable`). A new Rust release can add clippy lints
  that fail CI without any code change. Fix those in a dedicated PR.
- `cargo audit` also blocks the build in Contracts CI.

## Warning counts: non-blocking, but must not grow

Measured on 2026-09-29 against code identical to `main` at `23eaf29`.
clippy could not be run on the measuring machine; its zero comes from the CI
gate.

| Check                                   | Count                                                             | Blocking?      |
| --------------------------------------- | ----------------------------------------------------------------- | -------------- |
| Backend ESLint                          | **58 warnings**, all `@typescript-eslint/no-explicit-any`         | No             |
| Web ESLint                              | **4 warnings**, all `@typescript-eslint/no-unused-vars`           | No             |
| Backend Prettier (`src/**/*.ts`)        | 0                                                                 | Yes            |
| Root Prettier (`**/*.{ts,tsx,md,json}`) | 1 file (`apps/ai_agent/docs/concepts-rag-search-architecture.md`) | No (not in CI) |
| ruff check / ruff format                | 0 / 0                                                             | Yes            |
| rustfmt                                 | 0 diffs                                                           | No (not in CI) |
| clippy                                  | 0 (enforced by `-D warnings`)                                     | Yes            |

These warnings are tolerated, not accepted:

- **A PR must not increase any count.** A new `any` or unused variable gets
  fixed in the PR that introduces it. Reviewers compare the lint output
  against this table.
- **Touching a file is a good time to fix its warnings**, but do it in a
  separate commit so the behaviour change stays reviewable.
- When a count reaches zero, lock it in: pass `--max-warnings 0` to that
  package's `lint` script, or raise the rule to `error`. Then update this
  table.

## Conventions lint cannot enforce

### Comments explain _why_ and cite the issue behind them

This is the house style throughout `apps/backend`: 56 of the 83 non-test
source files cite an issue in their comments. Every comment worth writing
does two things:

1. **Explains why, not what.** The code already says what it does. The
   comment records the constraint, threat or failure that makes it look this
   way, the thing a reader could not work out from the code.
2. **References the issue number that motivated the code**, so the reader can
   find the full discussion.

It takes one of two forms: a leading `#N —` or a trailing `(#N)`.

```ts
// #330 — dev/test only: serves the fs-backed object store so presigned URLs
// issued locally are real, working URLs. Deliberately outside requireAuth
// (a presigned URL carries its own HMAC + expiry, exactly like S3), and never
// mounted in production, where the real object store answers these requests.
```

```ts
// Refuse to boot a non-dev gateway that would accept plaintext transport (#374).
assertTransportSecurityConfig();
```

Module-level JSDoc headers follow the same pattern: a title with the issue
number, then the design rationale. See `lib/transportSecurity.ts` (#374) and
`config/rateLimits.ts` (#375). Rust follows it too, for example the `///` on
`token_transfer::upgrade` (#44).

Things reviewers push back on:

- A comment that restates the code, such as `// increment counter`.
- A bare issue number with no explanation. The issue tracker may not outlive
  the code, so the comment must make sense on its own.
- An explanation with no issue number, when an issue exists.
- A comment that no longer matches the code. Update the comment in the same
  PR that changes the behaviour.

### Other conventions

- **Section banners.** Long files are split into sections with box-drawing
  banners, for example `// ── Socket events ───` in TypeScript and
  `# ── Helpers ───` in Python. Use the same style in new long files.
- **Environment reads take a `source` parameter** that defaults to
  `process.env`, parse with a safe fallback, and never throw on a bad value.
  Examples are `getRateLimitRule` and `isTlsEnforced`. This keeps them
  testable without mutating the global environment. Anything that must crash
  on start belongs in `config.ts`. See
  [environment-variables.md](environment-variables.md).
- **Never log message content.** Log IDs, counts and durations only. The
  pino redaction in `lib/logger.ts` is a backstop, not permission to log
  payloads.
- **The backend uses NodeNext import specifiers.** Relative imports end in
  `.js` (`'./lib/redis.js'`) even though the source file is `.ts`. `tsc`
  enforces this, but Backend CI does not run `tsc`, so run
  `pnpm --filter backend build` locally.
- **Commit messages** follow Conventional Commits (`feat:`, `fix:`, `docs:`,
  optionally scoped, like `fix(major): …`), as the README shows.
