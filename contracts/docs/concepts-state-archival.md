# State Archival & TTL Management

Soroban rents ledger state. Every storage entry a contract writes carries a **time-to-live
(TTL)**, measured in ledgers remaining until it is eligible for **archival** — removal from
the live ledger's BucketList into cold storage, from which it must be explicitly **restored**
before it can be read or written again. A contract that lets a live key's TTL run out does not
lose the data (archival is not deletion), but it does become unusable: any call that touches
an archived entry fails until an operator restores it.

This matters most for `group_treasury`: if `DataKey::Members` or an open `DataKey::Proposal`
archives, the treasury cannot authorize withdrawals, cannot accept votes, and cannot even
answer `get_members()` until someone restores the entry. This document covers how each of the
three contracts in this workspace uses storage, how (or whether) each extends TTLs, and what an
operator does when something archives anyway.

Source reviewed: `contracts/contracts/{token_transfer,group_treasury,proposals}/src/*.rs`,
verified by searching every contract source file for `extend_ttl`, `.bump(`, and `restore` —
none of the three contracts call any of them (see [Findings](#findings-no-contract-extends-any-ttl)).

Related: [`contracts-token-transfer-storage.md`](contracts-token-transfer-storage.md),
[`contracts-group-treasury-storage.md`](contracts-group-treasury-storage.md),
[`concepts-resource-budget.md`](concepts-resource-budget.md) (TTL/rent as one of several
metered resources), [`api-deployment-invocation.md`](api-deployment-invocation.md) (the
Stellar CLI conventions this document extends).

---

## Soroban storage and archival, briefly

Soroban's `env.storage()` exposes three *storage types*, but only **two archival durability
classes**:

| Storage type (`env.storage().X()`) | Archival durability | TTL behaviour |
| --- | --- | --- |
| `instance()` | **Persistent** | Shares one TTL with the contract's instance entry (and the contract's Wasm code entry). If it expires, the *entire* contract instance — every key stored via `instance()` — archives together. |
| `persistent()` | **Persistent** | Each keyed entry has its own independent TTL. |
| `temporary()` | **Temporary** | Expires and is permanently deleted (not archived-but-recoverable) once its TTL reaches zero. Never used to recover old data. |

A new entry starts with a TTL set by the network's minimum-TTL configuration for its
durability class. Every ledger that passes without action decrements that countdown. A
contract extends an entry's TTL explicitly, by calling `extend_ttl` (on a `Persistent`/`
Temporary` key via `env.storage().persistent().extend_ttl(key, threshold, extend_to)`, or on
the whole instance via `env.storage().instance().extend_ttl(threshold, extend_to)`) — there is
no automatic extension on read or write. If the TTL reaches zero before anything extends it,
the entry **archives**: it is still provably on the ledger (restorable, not deleted), but any
transaction that reads or writes it fails until it is restored. `temporary()` entries do not
archive — they are gone for good once their TTL expires, which is why nothing in this workspace
uses temporary storage for data that must survive.

## Findings: no contract extends any TTL

All three contracts store every key through `env.storage().instance()` exclusively. None of
them call `.extend_ttl()`, `.bump()`, or any TTL-related API, anywhere in `lib.rs` or
`storage.rs`:

```bash
grep -rniE "extend_ttl|\.bump\(|restore" contracts/{token_transfer,group_treasury,proposals}/src/*.rs
# (no matches)
```

That means TTL management for every key documented below is **entirely external to the
contract code** — it depends on whatever extends the contract's instance entry from outside
(an operator's maintenance transaction, a deploy/invoke call that happens to touch the
instance, or nothing at all). None of the deploy scripts in `contracts/scripts/` call
`extend` either, so a freshly deployed contract's only TTL activity is whatever minimum the
network assigns at creation and whatever `stellar contract invoke` calls happen to do
implicitly (simulated-and-submitted invocations typically refresh the footprint's TTL as part
of normal fee/resource accounting on write, but this is network/RPC behaviour, not something
this codebase controls or can rely on at a chosen interval).

## Per-contract storage key inventory

**Durability column meaning:** every key below is `instance()` storage, so "Durability" is
`Persistent` for all of them per the table above — the column is kept to make it explicit this
was checked per key, not assumed from the contract once.

### `token_transfer`

| Key | Durability | Where TTL is extended |
| --- | --- | --- |
| `DataKey::Admin` | Persistent (instance) | **Nowhere.** No call in `lib.rs` extends it. |
| `DataKey::TokenContract` | Persistent (instance) | **Nowhere.** |

Source: `contracts/token_transfer/src/{lib,storage}.rs`. Two keys, both instance, both never
explicitly extended — see [`contracts-token-transfer-storage.md`](contracts-token-transfer-storage.md#storage-type-usage)
for the storage-tier rationale (not the TTL gap, which that document does not cover).

### `group_treasury`

| Key | Durability | Where TTL is extended |
| --- | --- | --- |
| `DataKey::Admin` | Persistent (instance) | **Nowhere.** |
| `DataKey::Balances` | Persistent (instance) | **Nowhere.** |
| `DataKey::Members` | Persistent (instance) | **Nowhere.** |
| `DataKey::Threshold` | Persistent (instance) | **Nowhere.** |
| `DataKey::ProposalCount` | Persistent (instance) | **Nowhere.** |
| `DataKey::Proposal(u32)` | Persistent (instance) | **Nowhere.** |
| `DataKey::Vote(u32, Address)` | Persistent (instance) | **Nowhere.** |

Source: `contracts/group_treasury/src/{lib,storage}.rs`, per
[`contracts-group-treasury-storage.md`](contracts-group-treasury-storage.md#storage-tier-everything-is-instance-storage).
Because every key shares the single instance TTL, **there is no per-key risk gradient here** —
`Members` and `Proposal(id)` archive at exactly the same moment as `Admin`, because they are
the same ledger entry underneath `DataKey` variants. This is the contract named in the task
description: if the instance entry archives, `get_members()`, `propose_withdraw()`, and every
vote function fail until restored, because none of them can be called without first reading
`DataKey::Admin` or `DataKey::Members` from the same archived instance.

### `proposals`

| Key | Durability | Where TTL is extended |
| --- | --- | --- |
| `DataKey::Admin` | Persistent (instance) | **Nowhere.** |
| `DataKey::NextProposalId` | Persistent (instance) | **Nowhere.** |
| `DataKey::Proposal(u64)` | Persistent (instance) | **Nowhere.** |
| `DataKey::Vote(u64, Address)` | Persistent (instance) | **Nowhere.** |

Source: `contracts/proposals/src/{lib,storage}.rs`.

## TTL thresholds and extension amounts: none are configured

There are **no TTL threshold or extension-amount constants anywhere in this codebase** — no
`extend_ttl` call exists to carry them. This section exists to document that absence
explicitly, and to head off a likely point of confusion:

**Do not confuse storage TTL with `group_treasury`'s `ttl_ledgers` parameter.** 
`propose_withdraw(proposer, to, token, amount, ttl_ledgers)` takes a caller-supplied
`ttl_ledgers: u32` and computes `expires_at = env.ledger().timestamp() + (ttl_ledgers as u64 *
5)` — this is **proposal voting-window duration**, a business-logic field on
`WithdrawProposal`, converted from ledgers to seconds at the contract's documented ~5
seconds/ledger assumption. It has nothing to do with Soroban's storage-entry TTL or archival.
The backend's `apps/backend/docs/api-treasury.md` uses the same "TTL" word for the same
ledger→seconds conversion on its own `/treasury/propose` endpoint, which reinforces the
naming collision rather than resolving it — when reading "TTL" anywhere in this codebase,
confirm from context which of the two unrelated concepts is meant.

Because no contract calls `extend_ttl`, there is no "amount this workspace extends by" to
report, and no reasoning to document for a choice that was never made. What *would* need
choosing, if this gap were closed, and the reasoning each choice should weigh:

- **Threshold** (the `threshold` argument to `extend_ttl`, i.e. "extend once the entry has
  this many ledgers of TTL left") — should be comfortably larger than the longest expected gap
  between transactions that touch the instance, so routine inactivity never crosses it
  unnoticed.
- **`extend_to`** (how far out to push the TTL) — trades off against cost: Soroban charges
  rent proportional to entry size × extension length, so extending `group_treasury`'s instance
  (which grows without bound as proposals and votes accumulate, per
  [`concepts-resource-budget.md`](concepts-resource-budget.md#size-is-not-the-only-budget)) by
  a large amount becomes proportionally more expensive as the treasury accumulates history.
- Network-level minimum/maximum TTL bounds for persistent entries are a live, queryable
  network parameter (`ConfigSettingStateArchival` in Soroban's ledger config), not a constant
  safe to hard-code in a doc — query the current values from the target network's RPC rather
  than trusting a number written here, which could drift out of date.

## Risk: keys written but never extended

Every key in every contract in this workspace falls into this bucket — there is no subset to
carve out as "the risky ones," because none of the three contracts extend anything. Stated as
risks:

| Risk | Contract(s) affected | Consequence |
| --- | --- | --- |
| Instance entry (all storage) archives from pure inactivity | `token_transfer`, `group_treasury`, `proposals` | Every function panics/fails at the host level until an operator restores the instance. For `token_transfer`, this stops all transfers routed through it. For `group_treasury`, members cannot vote, deposit, or withdraw, and `is_member`/`get_members` — called by the external `proposals` contract's `execute_withdraw` — also fail, so a stalled treasury instance can block the `proposals` contract's execution path too. |
| No code path re-extends TTL on the operations that matter most | `group_treasury` especially | A treasury with no activity for a network-defined minimum period (an inactive group, or a group that only reads balances via simulated/`--send=no` calls, which do not extend TTL) drifts toward archival with no in-contract signal. There is no `ping()`/`keep_alive()` function a cron job could call to safely extend TTL without otherwise mutating state. |
| Unbounded instance growth raises the cost of recovery | `group_treasury`, `proposals` | Per `concepts-resource-budget.md`, both contracts keep every proposal and every vote in the same instance entry forever. If that entry archives, the restore transaction's cost scales with the *entire* accumulated entry size, not just the keys currently needed — a long-lived, heavily-used treasury is the most valuable to keep alive and the most expensive to recover if it lapses. |
| Deploy scripts do not establish a baseline extension | `contracts/scripts/deploy_*.sh` | A freshly deployed contract starts at whatever TTL the network assigns on creation and is never pushed further out by the deploy flow itself — the "someone extends this on a schedule" assumption has no owner anywhere in this repo today. |

**There is no mitigation in this codebase today.** This is an open gap, not a documented
design trade-off — nothing in `lib.rs`, `storage.rs`, or `contracts/scripts/` suggests this was
a deliberate choice. Closing it needs either an admin-gated `extend_ttl` entrypoint on each
contract (callable by automation without otherwise mutating state) or an operational process
that invokes the CLI commands below on a schedule tighter than the network's minimum persistent
TTL.

## Operator recovery: extending and restoring with the Stellar CLI

These commands follow the same invocation conventions as
[`api-deployment-invocation.md`](api-deployment-invocation.md) — `--source`, `--network`, and
`--id` resolve identities and the target network the same way. Soroban's archival durability
only has two classes (`persistent`, which covers both `instance()` and `persistent()` storage,
and `temporary()`); every key in this workspace is `persistent`.

**Extend before archival** (routine maintenance — run this periodically, well before the
network's minimum TTL would otherwise be reached, since nothing in the contracts does this
automatically):

```bash
stellar contract extend \
  --id $GROUP_TREASURY_CONTRACT_ID \
  --source deployer \
  --network testnet \
  --durability persistent \
  --ledgers-to-extend 535679
```

Omitting `--key`/`--key-xdr` extends the contract's own instance entry (and its Wasm code
entry) — the right target here, since every `DataKey` in this workspace lives inside that one
instance entry, not as independent `persistent()` keys.

**Restore after archival** (the entry has already lapsed and every call is failing):

```bash
stellar contract restore \
  --id $GROUP_TREASURY_CONTRACT_ID \
  --source deployer \
  --network testnet \
  --durability persistent
```

This resubmits the archived entry's current (last-known) state to the ledger and resets its
TTL, after which normal `invoke` calls succeed again. The account passed via `--source` pays
the restoration fee, which scales with the size of what is being restored — see the
unbounded-growth risk above for why this cost is not fixed over a treasury's lifetime.

Both subcommands' exact flags have changed across Stellar CLI releases (notably around how
`--ledgers-to-extend` and `--durability` are specified); confirm against the installed CLI
with `stellar contract extend --help` / `stellar contract restore --help` before relying on
the invocations above verbatim, rather than assuming they match every CLI version.

## What this document does not claim

- It does not assert specific network minimum/maximum TTL ledger counts — those are live
  network configuration, not a constant of this codebase, and are called out above as
  something to query rather than hard-code.
- It does not claim archival has been observed in testing or production for any of these
  contracts — the finding here is a static-analysis result (no `extend_ttl` call exists in the
  source), not an incident report.
