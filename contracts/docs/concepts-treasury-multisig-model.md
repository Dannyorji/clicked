# `group_treasury` Multisig / Authorization Model

Source: [`contracts/contracts/group_treasury/src/lib.rs`](../contracts/group_treasury/src/lib.rs),
[`contracts/contracts/group_treasury/src/test.rs`](../contracts/group_treasury/src/test.rs).

For the exact storage keys, types, and the proposal state diagram, see
[`contracts-group-treasury-storage.md`](contracts-group-treasury-storage.md). For function
signatures, see [`api-proposals.md`](api-proposals.md) (the related `proposals` contract) —
`group_treasury` itself has no dedicated API reference page in `contracts/docs/` today; this
document and the storage doc above are the canonical references for its own functions, cited
by exact name from `lib.rs`.

## Two independent authorization surfaces

`group_treasury` has two ways funds can move, and they do not share logic:

1. **Admin-gated `withdraw(to, token, amount)`** — a single admin address, set once at
   `initialize`, can move funds directly. No proposal or vote is involved.
2. **Member-voted proposals** (`propose_withdraw` / `approve_withdraw` / `reject_withdraw`) —
   the subject of this document. As of the current `lib.rs`, reaching `Passed` does **not**
   move funds: there is no execute entrypoint that reads a `Passed` proposal and calls
   `withdraw` on its behalf. The external `proposals` contract calls this contract's
   `is_member` / `balance` / `withdraw` after running its *own*, separate proposal/vote/finalize
   cycle (see `contracts/contracts/proposals/src/treasury_interface.rs` and
   [`api-proposals.md`](api-proposals.md)) — it does not call `propose_withdraw` or
   `approve_withdraw` on `group_treasury`. Treat the two proposal systems as parallel, not
   layered.

Everything below describes surface 2, the `group_treasury`-internal voting system.

## Membership

- **Adding a member:** `add_member(member)`. Admin-only (`require_admin`, i.e.
  `admin.require_auth()`). Panics with `"member already exists"` if the address is already in
  `DataKey::Members` — duplicates are prevented by a linear scan before insertion.
- **Removing a member:** `remove_member(member)`. Admin-only. Panics with
  `"member not found"` if the address isn't currently a member. Implemented by rebuilding the
  members vector without the target address (not swap-remove), so relative order of remaining
  members is preserved.
- **Self-removal:** `remove_member` takes an arbitrary `member` argument but the call itself is
  gated by `require_admin`, not by the caller being `member`. An admin who is also a member
  could call `remove_member(self)`, and any member removal (including of the admin's own
  address as a member entry, if it were ever added as one) is authorized the same way — there
  is no separate "members can remove themselves" path.
- **Effect on existing proposals:** membership changes do **not** touch any
  `DataKey::Proposal(id)` record. A proposal created while an address was a member keeps
  whatever `approvals`/`rejections` it already accumulated even if that address is later
  removed — the counters are not recomputed. However, a *newly removed* address can no longer
  vote (`require_votable` calls `is_member` at vote time, checking current membership, not
  membership at proposal-creation time), and a *newly added* member can vote on any
  still-`Active` proposal that predates their membership, since eligibility is also checked
  against current membership.

## Threshold model

`Threshold` (a `u32`) is the number of approval votes a proposal needs to move from `Active` to
`Passed`. It is **not** expressed as "N of M current members" in the code — it is a fixed
number set once, independent of how membership changes afterward.

- **Storage:** `DataKey::Threshold`, instance storage (see the storage doc for tier details).
- **Initial value:** set by the `threshold` parameter to `initialize`, which panics with
  `"threshold must be at least 1"` if `threshold == 0`. There is no other default.
- **Who can set it:** only `initialize`, which itself panics with `"already initialized"` if
  called a second time. **There is no `set_threshold` function in `lib.rs`** — the threshold
  cannot be changed after initialization by any current entrypoint.
- **Valid range:** any `u32 >= 1`. The contract does not validate `threshold` against the
  member count (which is `0` at `initialize` time in any case, since members are added
  afterward via `add_member`) — a treasury could be initialized with a threshold higher than
  it will ever have members, making it permanently impossible to pass a proposal. See
  Limitations below.
- **Changing membership after the threshold is set:** since threshold is fixed and membership
  is not, the *effective* difficulty of reaching `Threshold` approvals rises and falls as
  members are added or removed. Example: `Threshold = 2`, 3 members. If one member is removed
  (2 members left), a proposal still needs 2 approvals — now unanimous among the remaining
  members instead of 2-of-3.

## Withdrawal authorization: why proposal → approvals → threshold, not one signature

A single member's approval is deliberately insufficient — `approve_withdraw` only flips
`status` to `Passed` once `proposal.approvals >= threshold`, and there is no path that lets one
vote or one admin action mark a member-voted proposal as approved early. This requires
independent agreement from multiple members before a proposal is even eligible to move funds
(when a future execution step consumes `Passed`), rather than trusting any single member's
judgment.

- **Who may create proposals:** any current member, via `propose_withdraw`
  (`proposer.require_auth()` + `is_member` check). Also requires `amount > 0` and that the
  treasury's recorded balance for `token` is `>= amount` **at creation time**.
- **Who may approve/reject:** any current member, via `approve_withdraw` / `reject_withdraw`.
  Both require `require_auth()` from the voter and `is_member` on the voter.
- **Can the proposer approve their own proposal?** Yes, implicitly and automatically —
  `propose_withdraw` sets `approvals: 1` and records `DataKey::Vote(id, proposer) = true` at
  creation. The proposer does not call `approve_withdraw` separately for this; doing so
  afterward would hit the "already voted" panic below.
- **Duplicate approval/rejection:** `require_votable` checks
  `env.storage().instance().has(&DataKey::Vote(proposal_id, voter))` and panics with
  `"already voted"` if the address has already voted in either direction. A member cannot vote
  twice, and cannot switch an approval to a rejection (or vice versa) after voting once.
- **Required number of approvals:** exactly `Threshold` (the stored value, not derived from
  member count at vote time).
- **Approvals tied to current or proposal-time membership?** Current membership, checked at
  each vote via `is_member`. Whether a given address's *earlier* vote still "counts" after that
  address is removed is not re-validated — the `approvals` counter is never decremented on
  membership change, so a vote cast while a member remains counted after removal.
- **When does execution become possible?** As of the current `lib.rs`, it doesn't — reaching
  `Passed` is a terminal state as far as `group_treasury`'s own functions are concerned.
  `src/test.rs` (`multisig_flow::multisig_execute_on_approved`, `#[ignore]`d) documents this as
  planned but unimplemented, with no issue identified in-repo at the time of writing.
- **Who may execute, and must they be an approver?** Not applicable today — there is no
  execute function.
- **What happens after execution?** Not applicable today.

## Security properties and limitations

**What the model does provide today:**
- No single member (other than the admin, via the separate `withdraw` path) can move funds
  through the proposal system — reaching `Passed` requires `Threshold` distinct members'
  authenticated approvals.
- Each member's vote is authenticated (`require_auth`) and counted at most once per proposal.
- A proposal can be blocked before reaching threshold: once `rejections` reaches the
  *blocking minority* — `member_count.saturating_sub(threshold) + 1`, i.e. the smallest number
  of rejections that makes it mathematically impossible for the remaining members to still
  supply `Threshold` approvals — `reject_withdraw` flips the proposal to `Rejected`. Example
  from `src/test.rs`: `Threshold = 2` of 3 members → blocking minority = `3 - 2 + 1 = 2`.
- Voting past `expires_at` is blocked: `require_votable` panics with `"proposal expired"` once
  `env.ledger().timestamp() >= proposal.expires_at`, even though (see below) the stored
  `status` does not itself change to `Expired`.

**Known limitations, verified against the current source (not speculative):**

| Scenario | Actual behavior |
|---|---|
| Threshold never reached, proposal not rejected either | The proposal stays `Active` indefinitely once past `expires_at`; nothing in `lib.rs` transitions it out. `get_pending_proposals` will keep returning it (it filters on `status == Active`, not on `expires_at`) even though no further approval can be recorded past expiry. |
| Fewer members than `Threshold` | `initialize` does not check members against threshold (members don't exist yet at init time), and nothing later re-validates it. A treasury can end up with `Threshold` permanently unreachable if members are removed below that count, with no function to lower `Threshold` to compensate. |
| Member removed during an active vote | Verified in `remove_member`/`require_votable`: removal only updates `DataKey::Members`. It does not touch any `DataKey::Proposal` or `DataKey::Vote` entry. A removed member's prior vote stays counted in `approvals`/`rejections`; they simply can no longer cast new votes (`is_member` check fails on their next attempt). |
| Threshold changed during an active proposal | Not reachable — there is no function to change `Threshold` after `initialize`, so this scenario cannot occur in the current contract. |
| Stale / expired proposals | `require_votable`'s expiry check blocks new votes, but no function transitions `status` to `Expired` or otherwise finalizes a stale proposal. It remains `Active` in storage and in `list_proposals`/`get_pending_proposals` output. |
| Rejected / cancelled proposals | `Rejected` is terminal — no function reads or acts on a `Rejected` proposal afterward. There is no separate "cancel" function; rejection via `reject_withdraw` is the only way to end a proposal without reaching threshold. |
| Already-executed proposals | Not reachable — no function ever sets `status` to `Executed` in the current contract, so this state cannot occur yet. |
| Expired proposals (status-level) | As above, `Expired` is defined in `ProposalStatus` but no function sets it; expiry is enforced only as a vote-time guard, not a stored transition. |

Do not read the presence of `Executed`/`Expired` in the `ProposalStatus` enum, or the
`multisig_execute_on_approved` / `multisig_finalize_expired` test names, as evidence that
execution or expiry-finalization exists — both are explicitly `#[ignore]`d in `src/test.rs`
with a stated reason of being blocked on not-yet-implemented functionality.
