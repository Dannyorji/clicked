# `group_treasury` Contract API Reference

Every public function exposed by the `group_treasury` Soroban contract: full signature,
parameters, return value, authorization requirements, state mutations, and panic
conditions — verified against the current
[`contracts/contracts/group_treasury/src/lib.rs`](../contracts/group_treasury/src/lib.rs).

This is the function-by-function reference. For the storage layout behind these functions
(the `DataKey` inventory, the `WithdrawProposal` struct, the proposal state diagram), see
[`contracts-group-treasury-storage.md`](contracts-group-treasury-storage.md). For the
authorization model and known limitations in prose form, see
[`concepts-treasury-multisig-model.md`](concepts-treasury-multisig-model.md). For the
exact panic message catalog across all three contracts, see
[`contracts-errors.md`](contracts-errors.md).

## Table of contents

- [Shared auth helper: `require_admin`](#shared-auth-helper-require_admin)
- [Shared vote-validation helper: `require_votable`](#shared-vote-validation-helper-require_votable)
- [Setup & configuration](#setup--configuration)
  - [`initialize`](#initialize)
  - [`get_threshold`](#get_threshold)
- [Membership](#membership)
  - [`add_member`](#add_member)
  - [`remove_member`](#remove_member)
  - [`is_member`](#is_member)
  - [`get_members`](#get_members)
- [Funds movement](#funds-movement)
  - [`deposit`](#deposit)
  - [`withdraw`](#withdraw)
  - [`balance`](#balance)
- [Multisig withdrawal proposals](#multisig-withdrawal-proposals)
  - [`propose_withdraw`](#propose_withdraw)
  - [`approve_withdraw`](#approve_withdraw)
  - [`reject_withdraw`](#reject_withdraw)
  - [`get_proposal`](#get_proposal)
  - [`list_proposals`](#list_proposals)
  - [`get_pending_proposals`](#get_pending_proposals)
- [Worked example: multisig withdrawal flow end-to-end](#worked-example-multisig-withdrawal-flow-end-to-end)

---

## Shared auth helper: `require_admin`

Not a public contract function, but every admin-gated entrypoint below routes through it,
so its behavior is documented once here instead of repeated per function.

```rust
fn require_admin(env: &Env) -> Address {
    let admin: Address = env
        .storage()
        .instance()
        .get(&DataKey::Admin)
        .expect("not initialized");
    admin.require_auth();
    admin
}
```

- Loads `DataKey::Admin`. Panics with `"not initialized"` (an `.expect()` panic, not a
  `panic!` string literal, but the message is the same string) if the contract has never
  been initialized.
- Calls `admin.require_auth()` — the transaction must carry a valid authorization entry for
  the stored admin address, or the host aborts the invocation with an authorization failure
  (no contract-level message).
- Returns the admin `Address` to the caller for use in event payloads.

## Shared vote-validation helper: `require_votable`

Likewise not public, but both `approve_withdraw` and `reject_withdraw` delegate to it, so
their failure conditions are identical and documented once:

```rust
fn require_votable(env: &Env, voter: &Address, proposal_id: u32) -> WithdrawProposal {
    voter.require_auth();

    if !Self::is_member(env.clone(), voter.clone()) {
        panic!("not a member");
    }

    let proposal: WithdrawProposal = env
        .storage()
        .instance()
        .get(&DataKey::Proposal(proposal_id))
        .expect("proposal not found");

    if proposal.status != ProposalStatus::Active {
        panic!("proposal is not pending");
    }
    if proposal.status == ProposalStatus::Expired
        || env.ledger().timestamp() >= proposal.expires_at
    {
        panic!("proposal expired");
    }
    if env
        .storage()
        .instance()
        .has(&DataKey::Vote(proposal_id, voter.clone()))
    {
        panic!("already voted");
    }

    proposal
}
```

Checks run in this order: `require_auth()` → membership → proposal existence → status →
expiry → duplicate-vote. The `status != Active` check runs before the expiry check, so once
a proposal's `status` field would read as anything other than `Active`, the caller sees
`"proposal is not pending"` rather than `"proposal expired"` even on a proposal whose
deadline has also passed.

---

## Setup & configuration

### `initialize`

One-time initialization. Sets the admin, the approval threshold, and sets up the empty
balances map and members list.

**Full signature:**

```rust
pub fn initialize(env: Env, admin: Address, _token: Address, threshold: u32)
```

**Parameters:**

- `admin: Address` — the address granted admin rights (`add_member`, `remove_member`,
  `withdraw`).
- `_token: Address` — accepted but unused by the function body; the contract tracks
  balances per-token in a `Map`, not a single fixed token, so this parameter has no effect
  on behavior. Kept for interface/ABI compatibility.
- `threshold: u32` — number of approvals a withdraw proposal needs to pass. Must be `>= 1`.

**Authorization requirements:** **none.** `initialize` does not call `require_auth()` on
`admin` or any other address — anyone can call it on a freshly deployed, uninitialized
contract and name themselves admin. Deployment and initialization must happen atomically
(see [`api-deployment-invocation.md`](api-deployment-invocation.md)); the `"already
initialized"` guard below is the only protection against a takeover after the fact.

**State mutations:**

- `DataKey::Admin` ← `admin`
- `DataKey::Threshold` ← `threshold`
- `DataKey::ProposalCount` ← `0u32`
- `DataKey::Balances` ← empty `Map<Address, i128>`
- `DataKey::Members` ← empty `Vec<Address>`

**Panics:**

| Condition | Message |
|---|---|
| `DataKey::Admin` already present in instance storage | `"already initialized"` |
| `threshold == 0` | `"threshold must be at least 1"` |

There is no upper bound and no cross-check against member count — a treasury can be
initialized with a threshold higher than it will ever have members (members don't exist yet
at init time, since they're added afterward via `add_member`). See
[`concepts-treasury-multisig-model.md`](concepts-treasury-multisig-model.md) for the
consequences.

---

### `get_threshold`

Returns the configured approval threshold.

**Full signature:**

```rust
pub fn get_threshold(env: Env) -> u32
```

**Return value:** `u32` — the value stored at `DataKey::Threshold`.

**Authorization requirements:** none (read-only).

**Panics:**

| Condition | Message |
|---|---|
| Contract not initialized | `"not initialized"` |

This is the only accessor that reliably detects an uninitialized contract — `is_member` and
`get_members` both return harmless defaults (`false` / empty vector) instead of panicking.

---

## Membership

### `add_member`

Admin-only: add a new member to the treasury.

**Full signature:**

```rust
pub fn add_member(env: Env, member: Address)
```

**Parameters:** `member: Address` — the address to add.

**Authorization requirements:** admin-only, via `require_admin` (`admin.require_auth()`).

**State mutations:**

- Appends `member` to `DataKey::Members`.
- Publishes event topic `"member_added"` with a `MemberAddedEvent { member, added_by: admin }`.

**Panics:**

| Condition | Message |
|---|---|
| Contract not initialized | `"not initialized"` |
| Caller is not the admin | host auth failure (no contract message) |
| `member` is already in `DataKey::Members` | `"member already exists"` |

Duplicate detection is a linear scan over the current members vector before insertion —
there is no faster membership index.

---

### `remove_member`

Admin-only: remove a member from the treasury.

**Full signature:**

```rust
pub fn remove_member(env: Env, member: Address)
```

**Parameters:** `member: Address` — the address to remove.

**Authorization requirements:** admin-only, via `require_admin`.

**State mutations:**

- Rebuilds `DataKey::Members` without `member` (not swap-remove — relative order of the
  remaining members is preserved).
- Publishes event topic `"member_removed"` with a `MemberRemovedEvent { member, removed_by: admin }`.

**Panics:**

| Condition | Message |
|---|---|
| Contract not initialized | `"not initialized"` |
| Caller is not the admin | host auth failure |
| `member` is not currently in `DataKey::Members` | `"member not found"` |

**Important side effect not visible from the signature:** removal touches only
`DataKey::Members`. It does **not** touch any `DataKey::Proposal` or `DataKey::Vote` entry —
a removed member's prior votes stay counted in `approvals`/`rejections` on any proposal they
already voted on, they just can no longer cast *new* votes (`require_votable`'s `is_member`
check uses current membership, not membership at proposal-creation time). Removing a member
also shrinks `member_count`, which changes the blocking-minority arithmetic in
`reject_withdraw` for every currently-open proposal.

---

### `is_member`

Check whether an address is a current member.

**Full signature:**

```rust
pub fn is_member(env: Env, member: Address) -> bool
```

**Return value:** `bool` — `true` if `member` is present in `DataKey::Members`.

**Authorization requirements:** none (read-only).

**Panics:** none. Uses `unwrap_or_else(|| Vec::new(&env))` when reading `DataKey::Members`,
so it returns `false` on an uninitialized contract rather than panicking — **this function
cannot be used to detect initialization state.**

---

### `get_members`

Return the full current membership list.

**Full signature:**

```rust
pub fn get_members(env: Env) -> Vec<Address>
```

**Return value:** `Vec<Address>` — the current contents of `DataKey::Members`, in insertion
order (minus any removed addresses).

**Authorization requirements:** none (read-only).

**Panics:** none. Returns an empty vector on an uninitialized contract (same
`unwrap_or_else` pattern as `is_member`).

---

## Funds movement

### `deposit`

Transfer `amount` tokens from `from` into the treasury.

**Full signature:**

```rust
pub fn deposit(env: Env, from: Address, token: Address, amount: i128)
```

**Parameters:**

- `from: Address` — the depositor; **not required to be a treasury member.**
- `token: Address` — the SEP-41 token contract address being deposited.
- `amount: i128` — amount to deposit, must be `> 0`.

**Authorization requirements:** `from.require_auth()` only. Deposit is deliberately **not**
member-gated — any address may deposit into the treasury.

**State mutations:**

- Calls `TokenClient::new(&env, &token).transfer(&from, &env.current_contract_address(), &amount)`
  — an external cross-contract call that moves the actual tokens.
- Increments the treasury's internally tracked balance for `token` in `DataKey::Balances`.
- Publishes event topic `"deposit"` with a `DepositEvent { from, amount }`.

**Panics:**

| Condition | Message |
|---|---|
| `amount <= 0` | `"amount must be positive"` |
| `from` did not authorize the call | host auth failure |
| The underlying token's `transfer` call fails (e.g. insufficient balance) | delegated — the token contract's own failure, not a `group_treasury` message |

The amount check runs **before** `from.require_auth()`, so an invalid amount fails without
ever prompting a signature.

---

### `withdraw`

Admin-only: transfer `amount` tokens from the treasury to `to`.

**Full signature:**

```rust
pub fn withdraw(env: Env, to: Address, token: Address, amount: i128)
```

**Parameters:**

- `to: Address` — withdrawal recipient.
- `token: Address` — the SEP-41 token contract address.
- `amount: i128` — amount to withdraw, must be `> 0`.

**Authorization requirements:** admin-only, via `require_admin`. **This function does not
consult the proposal/voting system at all** — no threshold, no approvals, no member vote.
It is a separate, parallel path to moving funds alongside `propose_withdraw` →
`approve_withdraw`. See
[`concepts-treasury-multisig-model.md`](concepts-treasury-multisig-model.md#two-independent-authorization-surfaces)
for how the two relate (they don't — reaching `Passed` via the proposal system does not
itself call this function; there is no execute entrypoint today).

**State mutations:**

- Calls `TokenClient::new(&env, &token).transfer(&env.current_contract_address(), &to, &amount)`.
- Decrements the treasury's tracked balance for `token`.
- Publishes event topic `"withdraw"` with a `WithdrawEvent { to, amount }`.

**Panics:**

| Condition | Message |
|---|---|
| `amount <= 0` | `"amount must be positive"` |
| Contract not initialized | `"not initialized"` |
| Caller is not the admin | host auth failure |
| Tracked balance for `token` is less than `amount` | `"insufficient funds"` |
| The underlying token's `transfer` call fails | delegated |

Order: amount check → `require_admin` → balance check → token transfer. The `insufficient
funds` check reads the **internally tracked** `Balances` map, not the token contract's
actual on-chain balance — tokens sent directly to the treasury address without going
through `deposit` are invisible to this check.

---

### `balance`

Returns the token balance currently held by this treasury, as tracked internally.

**Full signature:**

```rust
pub fn balance(env: Env, token: Address) -> i128
```

**Parameters:** `token: Address` — the token contract to query.

**Return value:** `i128` — the tracked balance for `token`, or `0` if the token has never
been deposited or the contract is uninitialized (`unwrap_or(0)`).

**Authorization requirements:** none (read-only).

**Panics:** none. A `0` return is ambiguous between "no funds", "unknown token", and
"uninitialized contract" — call `get_threshold` first if that distinction matters.

---

## Multisig withdrawal proposals

### `propose_withdraw`

Member-only: create a new withdraw proposal. The proposer's own approval is recorded
automatically at creation time.

**Full signature:**

```rust
pub fn propose_withdraw(
    env: Env,
    proposer: Address,
    to: Address,
    token: Address,
    amount: i128,
    ttl_ledgers: u32,
) -> u32
```

**Parameters:**

- `proposer: Address` — must be a current member.
- `to: Address` — intended withdrawal recipient.
- `token: Address` — token contract to withdraw from.
- `amount: i128` — amount requested, must be `> 0`.
- `ttl_ledgers: u32` — caller-supplied lifetime for the proposal, converted to an absolute
  expiry timestamp as `env.ledger().timestamp() + (ttl_ledgers as u64 * 5)` (an
  approximate 5-seconds-per-ledger conversion). Despite the parameter name, `expires_at` is
  stored and checked as a **timestamp**, not a ledger count.

**Return value:** `u32` — the new proposal's id (the pre-increment value of
`DataKey::ProposalCount`; ids start at `0`).

**Authorization requirements:** `proposer.require_auth()`, plus a membership check
(`is_member`).

**State mutations:**

- Increments `DataKey::ProposalCount`.
- Writes a new `WithdrawProposal` to `DataKey::Proposal(id)` with `approvals: 1`,
  `rejections: 0`, `status: Active`.
- Writes `DataKey::Vote(id, proposer)` ← `true` — **the proposer auto-approves their own
  proposal** at creation; they do not call `approve_withdraw` separately (doing so
  afterward hits the "already voted" panic).
- Publishes event topic `"proposal_created"` with a `ProposalCreatedEvent { id, proposer, to, token, amount, expires_at }`.

**Panics:**

| Condition | Message |
|---|---|
| `proposer` did not authorize | host auth failure |
| `proposer` is not a current member | `"proposer is not a member"` |
| `amount <= 0` | `"amount must be positive"` |
| Tracked balance for `token` is less than `amount` | `"insufficient funds"` |

Order: `require_auth` → membership → amount → balance. The signing prompt appears **before**
the membership check, so a non-member is asked to sign a transaction that then fails. The
balance check happens at proposal-creation time only — funds are not escrowed, so a proposal
that was fundable when created may not be when a later execution step (if one existed)
attempted to move funds.

---

### `approve_withdraw`

Member-only: approve a pending withdraw proposal. Transitions the proposal to `Passed` once
the approval count reaches `threshold`.

**Full signature:**

```rust
pub fn approve_withdraw(env: Env, approver: Address, proposal_id: u32)
```

**Parameters:**

- `approver: Address` — must be a current member who has not yet voted on this proposal.
- `proposal_id: u32` — target proposal.

**Authorization requirements:** delegated to [`require_votable`](#shared-vote-validation-helper-require_votable):
`approver.require_auth()` + current membership + proposal must be `Active`, unexpired, and
not already voted on by `approver`.

**State mutations:**

- Writes `DataKey::Vote(proposal_id, approver)` ← `true`.
- Increments `proposal.approvals`.
- If `approvals >= threshold` (read from `DataKey::Threshold`): sets `status` to `Passed`
  and publishes event topic `"proposal_approved"` with a
  `ProposalApprovedEvent { id, approvals, threshold }`.
- Writes the updated proposal back to `DataKey::Proposal(proposal_id)`.
- Always publishes event topic `"withdraw_vote"` with a
  `WithdrawVoteCastEvent { id, voter: approver, approve: true }`, regardless of whether
  threshold was reached.

**Panics:** (all via `require_votable`, plus one of its own)

| Condition | Message |
|---|---|
| `approver` did not authorize | host auth failure |
| `approver` is not a current member | `"not a member"` |
| No proposal with `proposal_id` | `"proposal not found"` |
| Proposal status is not `Active` | `"proposal is not pending"` |
| `env.ledger().timestamp() >= proposal.expires_at` | `"proposal expired"` |
| `approver` already has a recorded vote on this proposal | `"already voted"` |
| `DataKey::Threshold` unset | `"not initialized"` |

Reaching `Passed` is a **terminal state as far as this contract's own functions are
concerned** — there is no execute entrypoint in `lib.rs` that reads a `Passed` proposal and
moves funds. See
[`concepts-treasury-multisig-model.md`](concepts-treasury-multisig-model.md#withdrawal-authorization-why-proposal--approvals--threshold-not-one-signature).

---

### `reject_withdraw`

Member-only: reject a pending withdraw proposal. Transitions the proposal to `Rejected` once
rejections reach the blocking minority.

**Full signature:**

```rust
pub fn reject_withdraw(env: Env, rejecter: Address, proposal_id: u32)
```

**Parameters:**

- `rejecter: Address` — must be a current member who has not yet voted on this proposal.
- `proposal_id: u32` — target proposal.

**Authorization requirements:** same as `approve_withdraw`, via `require_votable`.

**State mutations:**

- Writes `DataKey::Vote(proposal_id, rejecter)` ← `false`.
- Increments `proposal.rejections`.
- Computes `blocking_minority = member_count.saturating_sub(threshold) + 1` — the smallest
  number of rejections that makes it mathematically impossible for the remaining members to
  still supply `threshold` approvals.
- If `rejections >= blocking_minority`: sets `status` to `Rejected` and publishes event
  topic `"proposal_rejected"` with a `ProposalRejectedEvent { id, rejections }`.
- Writes the updated proposal back to storage.
- Always publishes event topic `"withdraw_vote"` with
  `WithdrawVoteCastEvent { id, voter: rejecter, approve: false }`.

**Panics:** identical trigger set to `approve_withdraw` (shared `require_votable`):

| Condition | Message |
|---|---|
| `rejecter` did not authorize | host auth failure |
| `rejecter` is not a current member | `"not a member"` |
| No proposal with `proposal_id` | `"proposal not found"` |
| Proposal status is not `Active` | `"proposal is not pending"` |
| `env.ledger().timestamp() >= proposal.expires_at` | `"proposal expired"` |
| `rejecter` already has a recorded vote on this proposal | `"already voted"` |
| `DataKey::Threshold` unset | `"not initialized"` |

`Rejected` is terminal — no function reads or acts on a `Rejected` proposal afterward, and
there is no separate "cancel" function.

---

### `get_proposal`

Returns the withdraw proposal with the given id.

**Full signature:**

```rust
pub fn get_proposal(env: Env, proposal_id: u32) -> WithdrawProposal
```

**Return value:** the full `WithdrawProposal` struct (see
[`contracts-group-treasury-storage.md`](contracts-group-treasury-storage.md#withdrawproposal)
for field-by-field documentation).

**Authorization requirements:** none (read-only).

**Panics:**

| Condition | Message |
|---|---|
| No proposal exists at `proposal_id` | `"proposal not found"` |

---

### `list_proposals`

Returns every proposal ever created.

**Full signature:**

```rust
pub fn list_proposals(env: Env) -> Vec<WithdrawProposal>
```

**Return value:** `Vec<WithdrawProposal>`, built by iterating ids `1..=count` (where `count`
is `DataKey::ProposalCount`) and collecting whichever ones still have a stored entry.

**Authorization requirements:** none (read-only).

**Panics:** none — missing entries are skipped via `if let Some(...)`.

**Known off-by-one:** ids are assigned starting at `0` (see `propose_withdraw`), but this
function iterates `1..=count`. **Proposal `0` — the very first proposal ever created — is
never included in the result**, while the loop's final iteration probes a nonexistent id
past the end (harmlessly skipped by the `Option` guard). Fetch proposal `0` directly via
`get_proposal(0)` if completeness matters.

---

### `get_pending_proposals`

Returns every proposal whose status is `Active`.

**Full signature:**

```rust
pub fn get_pending_proposals(env: Env) -> Vec<WithdrawProposal>
```

**Return value:** `Vec<WithdrawProposal>`, same `1..=count` iteration as `list_proposals`,
filtered to `status == ProposalStatus::Active`.

**Authorization requirements:** none (read-only).

**Panics:** none.

Shares the same off-by-one as `list_proposals` (proposal `0` excluded). It also filters only
on `status`, not on `expires_at` — a proposal whose deadline has passed but which nothing
has yet voted into a terminal state still shows up here, even though `require_votable` would
reject any further vote on it with `"proposal expired"`.

---

## Worked example: multisig withdrawal flow end-to-end

This walks `propose_withdraw` → `approve_withdraw` × N through to a `Passed` proposal, then
shows the rejection path, using the same setup pattern as
[`src/test.rs`](../contracts/group_treasury/src/test.rs)'s `voting_setup` helper. All
addresses are illustrative; `amount`s are in a token's smallest unit.

### Setup

```rust
// threshold = 2, 3 members
let admin = Address::generate(&env);
let token_id = env.register(mock_token::MockToken, ());
let contract_id = env.register(GroupTreasuryContract, ());
let client = GroupTreasuryContractClient::new(&env, &contract_id);

client.initialize(&admin, &token_id, &2); // threshold = 2

let member_a = Address::generate(&env);
let member_b = Address::generate(&env);
let member_c = Address::generate(&env);
client.add_member(&member_a);
client.add_member(&member_b);
client.add_member(&member_c);
```

`get_threshold()` now returns `2`; `get_members()` returns `[member_a, member_b, member_c]`.

### 1. Fund the treasury

```rust
let token = mock_token::MockTokenClient::new(&env, &token_id);
token.mint(&member_a, &500_000);
client.deposit(&member_a, &token_id, &500_000);
```

`balance(token_id)` now returns `500_000`. (Deposit is not member-gated, but `member_a`
happens to be a member here.)

### 2. Create the proposal

```rust
let recipient = Address::generate(&env);
let id = client.propose_withdraw(&member_a, &recipient, &token_id, &100_000, &100);
// id == 0
```

Immediately after this call:

- `get_proposal(0).approvals == 1` (the proposer's auto-approval).
- `get_proposal(0).status == ProposalStatus::Active`.
- `DataKey::Vote(0, member_a) == true` — `member_a` cannot vote again on proposal `0`.
- `expires_at = env.ledger().timestamp() + (100 * 5)` — roughly 500 seconds of voting
  window from a `ttl_ledgers` of `100`.

### 3. First additional approval — still below threshold

```rust
client.approve_withdraw(&member_b, &0);

let proposal = client.get_proposal(0);
assert_eq!(proposal.approvals, 2);
assert_eq!(proposal.status, ProposalStatus::Active); // not yet Passed... 
```

Wait — with `threshold = 2` and the proposer's auto-approval already counted as `1`, a
single additional approval (`member_b`) brings `approvals` to `2`, which **meets**
`threshold`. So in this concrete 2-of-3 setup, `status` flips to `Passed` right here, not
after a third vote:

```rust
assert_eq!(proposal.status, ProposalStatus::Passed);
```

A `proposal_approved` event with `{ id: 0, approvals: 2, threshold: 2 }` is published on
this call. `member_c` never needs to vote for this proposal to pass — exactly `threshold`
distinct approvals are required, not unanimity.

### 4. Attempting to vote after `Passed`

```rust
client.approve_withdraw(&member_c, &0);
// panics: "proposal is not pending"
```

`require_votable` rejects any further vote once `status != Active`, approval or rejection,
from any member — including one who never voted.

### 5. What `Passed` does — and does not — do

```rust
let treasury_balance = client.balance(&token_id);
assert_eq!(treasury_balance, 500_000); // unchanged
```

No funds moved. There is no function in `lib.rs` that reads a `Passed` proposal and executes
a transfer — `propose_withdraw` / `approve_withdraw` / `reject_withdraw` is a complete,
self-contained voting record, but this contract's own `Passed` state is not wired to its
`withdraw` function. (The separate `proposals` contract calls `is_member` / `balance` /
`withdraw` on this contract after running its *own* independent proposal cycle — see
[`api-proposals.md`](api-proposals.md) — but that is not triggered by this contract reaching
`Passed`.) If funds are to move for this specific proposal today, the admin must call
`withdraw(to, token, amount)` separately and manually, using the proposal's `to`/`token`/
`amount` fields as a reference — `withdraw` does not read or require a `Passed` proposal to
exist.

### 6. The rejection path (a second proposal, same treasury)

```rust
let id2 = client.propose_withdraw(&member_a, &recipient, &token_id, &50_000, &100);
// id2 == 1, approvals == 1 (member_a's auto-approval), status == Active

client.reject_withdraw(&member_b, &1);
// rejections == 1; blocking_minority = 3 - 2 + 1 = 2; still Active

client.reject_withdraw(&member_c, &1);
// rejections == 2 == blocking_minority -> status flips to Rejected
```

A `proposal_rejected` event with `{ id: 1, rejections: 2 }` is published on the second
rejection. From here, proposal `1` is terminal — no function acts on a `Rejected` proposal
again, and `member_a`'s own approval cannot be withdrawn or converted.

### Reading back the full picture

```rust
let all = client.list_proposals();       // note: starts scanning at id 1 — see caveat above
let pending = client.get_pending_proposals(); // empty here, since both proposals are terminal
let p0 = client.get_proposal(0); // Passed
let p1 = client.get_proposal(1); // Rejected
```

Because both proposals in this walkthrough are ids `0` and `1`, and `list_proposals`/
`get_pending_proposals` both start their scan at `1`, `all` in this example contains only
proposal `1` — proposal `0` is silently excluded by the off-by-one documented under
[`list_proposals`](#list_proposals). Always fetch id `0` directly with `get_proposal(0)` if
your integration cares about the very first proposal a treasury ever created.

---

## Related documents

- [`contracts-group-treasury-storage.md`](contracts-group-treasury-storage.md) — storage key
  inventory, `WithdrawProposal` field reference, and the proposal state diagram.
- [`concepts-treasury-multisig-model.md`](concepts-treasury-multisig-model.md) — the
  authorization model in prose, including the two independent ways funds can move and the
  documented limitations (unreachable threshold, stale proposals, membership changes mid-vote).
- [`contracts-errors.md`](contracts-errors.md) — panic catalog across all three contracts,
  with client-presentation guidance per failure.
- [`api-proposals.md`](api-proposals.md) — the separate `proposals` contract, which calls
  `is_member` / `balance` / `withdraw` on this contract via cross-contract invocation.
- [`contracts-events.md`](contracts-events.md) — the full event catalog, including every
  event this contract publishes.
- [`testing.md`](testing.md) — how the test suite referenced in the worked example above
  exercises these functions, including `#[ignore]`d stubs for not-yet-implemented execution
  and expiry-finalization paths.
