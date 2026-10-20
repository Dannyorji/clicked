# Contract Security Review Checklist

The checklist a reviewer works through on any PR that touches
`contracts/contracts/{token_transfer,group_treasury,proposals}`. Each item below is answered
for all three contracts as they stand today, with a link to the exact code or test that proves
the answer — or, where nothing proves it, marked **Open finding** rather than assumed safe.

This is a point-in-time audit, not a standing guarantee. Re-check an item against the diff
whenever a PR touches the function(s) it covers; a passing answer here does not transfer to
code written after this document.

**Read this first if you're about to write or review a test:** a test built on
`env.mock_all_auths()` proves that *some* authorization mock exists for the call — not that the
*correct* address is the one required. A `require_auth()` call can be deleted entirely from a
function and every `mock_all_auths()`-based test will keep passing. Every "Pass" below that
relies on a test cites either a scoped `mock_auths()` / `env.auths()` assertion, an unmocked
`should_panic` test, or (for the init-front-running items) a test that demonstrates the
*absence* of a check by succeeding with no mocked auth at all — never a blanket
`mock_all_auths()` test standing alone. See
[`testing.md`](testing.md#testing-authorization) (#348) for the full explanation and the
three-tier pattern this review follows.

---

## 1. `require_auth` on every state-changing entry point

| Function | `token_transfer` | `group_treasury` | `proposals` |
| --- | --- | --- | --- |
| `initialize` | ❌ No auth at all — see [§2](#2-initialization-front-running-and-re-initialization) | ❌ No auth at all — see [§2](#2-initialization-front-running-and-re-initialization) | ✅ `admin.require_auth()` — [`lib.rs:45`](../contracts/proposals/src/lib.rs#L45) |
| `transfer` | ✅ `from.require_auth()` — [`lib.rs:38`](../contracts/token_transfer/src/lib.rs#L38), proven by `test_transfer_requires_sender_auth` ([`test.rs:124`](../contracts/token_transfer/src/test.rs#L124)) | — | — |
| `set_token_contract` | ✅ `admin.require_auth()` via loaded admin — [`lib.rs:86`](../contracts/token_transfer/src/lib.rs#L86) | — | — |
| `upgrade` | ✅ `admin.require_auth()` — [`lib.rs:104`](../contracts/token_transfer/src/lib.rs#L104), proven by `test_upgrade_non_admin_panics` ([`test.rs:174`](../contracts/token_transfer/src/test.rs#L174)) | — | — |
| `add_member` / `remove_member` | — | ✅ `require_admin()` → `admin.require_auth()` — [`lib.rs:21`](../contracts/group_treasury/src/lib.rs#L21), proven by `test_non_admin_cannot_add_member` ([`test.rs:258`](../contracts/group_treasury/src/test.rs#L258)) | — |
| `deposit` | — | ✅ `from.require_auth()` — [`lib.rs:154`](../contracts/group_treasury/src/lib.rs#L154) | — |
| `withdraw` | — | ✅ `require_admin()` — [`lib.rs:180`](../contracts/group_treasury/src/lib.rs#L180), proven by `test_non_admin_cannot_withdraw` ([`test.rs:158`](../contracts/group_treasury/src/test.rs#L158)) | — |
| `propose_withdraw` | — | ✅ `proposer.require_auth()` — [`lib.rs:225`](../contracts/group_treasury/src/lib.rs#L225), proven by `test_propose_withdraw_non_member_panics` ([`test.rs:784`](../contracts/group_treasury/src/test.rs#L784)) (membership check; see note below on what this proves) | — |
| `approve_withdraw` / `reject_withdraw` | — | ✅ voter `require_auth()` via `require_votable` — [`lib.rs:434`](../contracts/group_treasury/src/lib.rs#L434), proven unmocked by `test_vote_without_auth_panics` ([`test.rs:579`](../contracts/group_treasury/src/test.rs#L579)) | — |
| `create_proposal` | — | — | ✅ `proposer.require_auth()` — [`lib.rs:64`](../contracts/proposals/src/lib.rs#L64) |
| `vote` | — | — | ✅ `voter.require_auth()` — [`lib.rs:119`](../contracts/proposals/src/lib.rs#L119), proven by `double_vote_panics` style tests combined with the auth-presence pattern in `testing.md` |
| `finalize_proposal` / `finalize_expired_proposal` | — | — | ⚪ **Intentionally no auth.** Doc comment states "callable by anyone" — [`lib.rs:154`](../contracts/proposals/src/lib.rs#L154), [`lib.rs:192`](../contracts/proposals/src/lib.rs#L192). Gated by time (`expires_at`) and by the existing vote tally, not by caller identity, and moves no funds — reviewed as a deliberate design choice, not a gap. Confirm this reasoning still holds if either function is ever changed to do more than flip a status. |
| `execute_proposal` | — | — | ✅ `executor.require_auth()` — [`lib.rs:219`](../contracts/proposals/src/lib.rs#L219). Note: authenticates *that someone signed*, not that the signer is privileged — anyone may execute a `Passed` proposal. This is load-bearing only for the MVP path, which moves no funds (see [§3](#3-admin-key-powers)). |
| `execute_withdraw` | — | — | ✅ `caller.require_auth()` plus a separate `treasury.is_member(caller)` check — [`lib.rs:246-263`](../contracts/proposals/src/lib.rs#L246-L263), proven by `execute_withdraw_non_member_panics` ([`test.rs:450`](../contracts/proposals/src/test.rs#L450)) |

**Note on `propose_withdraw`'s proof:** `test_propose_withdraw_non_member_panics` proves the
*membership* check panics for a non-member — it does not, by itself, distinguish "no auth
check exists" from "auth check exists but membership failed first." Combined with
`test_vote_without_auth_panics` existing as a *separate, deliberately unmocked* test for the
sibling voting functions, the pattern in this contract's test suite is consistent enough to
trust, but a reviewer adding a new `require_auth()`-gated function here should add an unmocked
test for it specifically, not rely on the membership check to stand in for one.

---

## 2. Initialization front-running and re-initialization

**Re-initialization** (calling `initialize` a second time) is guarded in all three contracts
by the same pattern — `if env.storage().instance().has(&DataKey::Admin) { panic!("already
initialized"); }` — and verified for two of the three:

| Contract | Re-init guard | Test |
| --- | --- | --- |
| `token_transfer` | ✅ [`lib.rs:19-21`](../contracts/token_transfer/src/lib.rs#L19-L21) | ✅ `test_double_initialize_panics` ([`test.rs:78`](../contracts/token_transfer/src/test.rs#L78)) |
| `group_treasury` | ✅ [`lib.rs:34-36`](../contracts/group_treasury/src/lib.rs#L34-L36) | ✅ `test_double_initialize_panics` ([`test.rs:80`](../contracts/group_treasury/src/test.rs#L80)) |
| `proposals` | ✅ [`lib.rs:42-44`](../contracts/proposals/src/lib.rs#L42-L44) | ⚠️ **Open finding** — no test in `proposals/src/test.rs` calls `initialize` twice; the guard exists but is unverified by this suite. |

**Initialization front-running** (someone other than the intended deployer calling
`initialize` first) is the more serious half of this item, and the three contracts are **not**
equally protected — `require_auth` on the `admin` parameter is a weaker mitigation than it
looks:

| Contract | Front-running exposure |
| --- | --- |
| `token_transfer` | ⚠️ **Open finding.** `initialize` has **no** `require_auth` call at all — [`lib.rs:18-26`](../contracts/token_transfer/src/lib.rs#L18-L26). Anyone who observes the deployed-but-uninitialized contract can call `initialize(attacker, any_token)` and become admin permanently; nothing in the function checks who is calling. Proven by the test suite itself: `test_initialize` ([`test.rs:70`](../contracts/token_transfer/src/test.rs#L70)) calls `client.initialize(...)` with **no `env.mock_all_auths()` anywhere in `setup()`**, and it succeeds — if any auth were required, this test would panic. |
| `group_treasury` | ⚠️ **Open finding**, same shape. `initialize` has no `require_auth` — [`lib.rs:33-49`](../contracts/group_treasury/src/lib.rs#L33-L49). `test_initialize` ([`test.rs:72`](../contracts/group_treasury/src/test.rs#L72)) also calls `client.initialize(...)` with no auth mocked anywhere in `setup()`, and succeeds. |
| `proposals` | ⚠️ **Partially mitigated, not closed.** `initialize` does call `admin.require_auth()` — [`lib.rs:45`](../contracts/proposals/src/lib.rs#L45) — but this only proves *the address named as admin authorized being admin*; it does not stop an attacker from calling `initialize(attacker_address)` and signing for `attacker_address` themselves. It stops a third party from being framed as admin without their consent; it does **not** stop an attacker from racing the legitimate deployer's own `initialize` call. The real mitigation for all three contracts is operational — collapsing deploy-then-initialize into the smallest possible window, ideally the same submitted transaction batch — not anything enforceable inside `initialize` itself, since the function cannot know who the "legitimate" deployer is before any state exists to record that. |

**Reviewer guidance:** treat "front-running window between deploy and initialize" as a
deployment-process risk for all three contracts, not something a code change to `initialize`
alone can fully close. For `token_transfer` and `group_treasury` specifically, adding
*some* `require_auth` to `initialize` (even a deployer-controlled contract/address passed as a
constructor-style constraint) is still strictly better than none, since today literally no
signature of any kind is required to claim admin on either.

---

## 3. Admin key powers

What the admin address can do, per contract, once it holds that role:

| Contract | Admin powers | Where |
| --- | --- | --- |
| `token_transfer` | `set_token_contract` (repoint which SEP-41 token this router transfers) ; `upgrade` (replace the contract's own Wasm via `env.deployer().update_current_contract_wasm`) | [`lib.rs:80-106`](../contracts/token_transfer/src/lib.rs#L80-L106) |
| `group_treasury` | `add_member` / `remove_member` (full control of who can vote) ; `withdraw` (move any amount of any token the treasury holds, directly, bypassing the entire member-vote system) | [`lib.rs:60-123`](../contracts/group_treasury/src/lib.rs#L60-L123), [`lib.rs:175-202`](../contracts/group_treasury/src/lib.rs#L175-L202) |
| `proposals` | Reserved but **unused**. The doc comment at [`lib.rs:38-40`](../contracts/proposals/src/lib.rs#L38-L40) states the admin slot exists for a future governance hook; no function in the current contract checks the stored admin against anything. |

**Open finding, `group_treasury`:** the admin's direct `withdraw` is a complete bypass of the
member-voting system documented in
[`concepts-treasury-multisig-model.md`](concepts-treasury-multisig-model.md#two-independent-authorization-surfaces)
— a single admin key can move 100% of treasury funds with zero member approvals, in the same
contract whose entire other surface is built around requiring multiple members to agree. There
is no way to change or revoke the admin address after `initialize` (no `set_admin` function in
any of the three contracts), so a compromised or malicious admin key is a permanent, unilateral
risk for the lifetime of the deployed instance.

**Upgrade risk (`token_transfer` only — the only contract with an `upgrade` function):** a
compromised admin key can replace the entire contract logic via `upgrade`, which is a strictly
larger blast radius than any individual function misuse. There is no timelock, multi-sig, or
second-party check on `upgrade` beyond the single `admin.require_auth()`. `group_treasury` and
`proposals` have no `upgrade` function at all, so this specific risk does not apply to them.

---

## 4. Integer overflow on `i128` amounts

No contract performs checked arithmetic (`checked_add`, `checked_mul`, etc.) anywhere —
confirmed by `grep -rn "checked_add\|checked_mul" contracts/*/src/*.rs` returning no matches.
The only protection against silent `i128` wraparound is a **workspace-level build setting**,
not contract code:

```toml
# contracts/Cargo.toml
[profile.release]
overflow-checks = true
```

This is documented and deliberately justified in
[`concepts-resource-budget.md`](concepts-resource-budget.md#release-profile-settings-that-affect-size):
*"these contracts move funds; a silent `i128` wraparound in a balance is a critical bug. This
is a deliberate trade of size for safety."* With this flag on, an overflow traps (panics) at
the host level instead of wrapping silently.

**Open finding:** this protection is a one-line Cargo profile setting that nothing in the
contract test suites exercises directly — `grep -rn "overflow" contracts/*/src/test.rs` returns
no matches in any of the three contracts. There is no test anywhere that constructs an amount
near `i128::MAX` and asserts the expected panic. A future change to `contracts/Cargo.toml`
(for example, to shrink Wasm size — see the same resource-budget document's own warning not to
do this) could silently remove this protection with no test failing to catch it. Any PR that
touches `[profile.release]` in `contracts/Cargo.toml` should be treated as security-relevant for
this reason alone, and a reviewer should not assume overflow safety is "tested" in the normal
sense — it is configured, and unverified by any test in this repo.

---

## 5. Negative or zero amounts

Every function that accepts an `i128` amount intended to be positive guards it with the same
`if amount <= 0 { panic!("amount must be positive"); }` pattern, which covers negative and zero
in one comparison:

| Function | Guard | Zero tested? | Negative tested? |
| --- | --- | --- | --- |
| `token_transfer::transfer` | [`lib.rs:33-35`](../contracts/token_transfer/src/lib.rs#L33-L35) | ✅ `test_transfer_zero_panics` ([`test.rs:113`](../contracts/token_transfer/src/test.rs#L113)) | ⚠️ **Open finding** — not tested |
| `group_treasury::deposit` | [`lib.rs:151-153`](../contracts/group_treasury/src/lib.rs#L151-L153) | ✅ `test_deposit_zero_panics` ([`test.rs:175`](../contracts/group_treasury/src/test.rs#L175)) | ⚠️ **Open finding** — not tested |
| `group_treasury::withdraw` | [`lib.rs:176-178`](../contracts/group_treasury/src/lib.rs#L176-L178) | ⚠️ **Open finding** — no zero-amount test exists for `withdraw` specifically | ⚠️ **Open finding** — not tested |
| `group_treasury::propose_withdraw` | [`lib.rs:231-233`](../contracts/group_treasury/src/lib.rs#L231-L233) | ⚠️ **Open finding** — no zero-amount test exists | ⚠️ **Open finding** — not tested |
| `proposals::create_proposal` | [`lib.rs:69-71`](../contracts/proposals/src/lib.rs#L69-L71) | ⚠️ **Open finding** — no zero-amount test exists | ⚠️ **Open finding** — not tested |

The guard condition (`<= 0`) is correct by inspection for both zero and negative values in every
one of these five call sites — this is not a correctness finding about the code. It *is* a test
coverage finding: only two of five guarded functions have any test proving the guard fires, and
**none** of the five has a test using an actual negative literal (every existing test uses
`0`). A negative amount passing one of these checks would be a critical bug (e.g., a negative
`withdraw` amount reducing `current - amount` by a negative number, i.e. *increasing* the stored
balance) and nothing in the current suite would catch a regression that weakened `<= 0` to
`== 0` by mistake.

---

## 6. Threshold edge cases

Scoped to `group_treasury`, the only contract with a fixed numeric approval `Threshold`.
(`proposals` uses simple majority of cast votes with no quorum and no threshold concept — see
[`concepts-proposal-lifecycle.md`](concepts-proposal-lifecycle.md#voting), so neither edge case
below applies to it.)

| Edge case | Status | Evidence |
| --- | --- | --- |
| `threshold == 0` at `initialize` | ✅ Rejected | `panic!("threshold must be at least 1")` — [`lib.rs:37-39`](../contracts/group_treasury/src/lib.rs#L37-L39), proven by `test_initialize_zero_threshold_panics` ([`test.rs:361`](../contracts/group_treasury/src/test.rs#L361)) |
| `threshold` greater than member count | ⚠️ **Open finding.** Not rejected anywhere — `initialize` cannot check this in any case, since members are always `0` at init time (added afterward via `add_member`), and no later function re-validates `Threshold` against `Members.len()`. A treasury can be permanently initialized in a state where no proposal can ever reach `Threshold` approvals. Documented as a known limitation, not tested as a guarded case, in [`concepts-treasury-multisig-model.md`](concepts-treasury-multisig-model.md#known-limitations-verified-against-the-current-source-not-speculative) (see the "Fewer members than `Threshold`" row). There is no `set_threshold` function to recover from this once deployed. |
| Removing members below `Threshold` | ⚠️ **Open finding.** `remove_member` ([`lib.rs:89-123`](../contracts/group_treasury/src/lib.rs#L89-L123)) has no awareness of `Threshold` at all — it can remove members down to zero while `Threshold` stays fixed, with no warning, no block, and no test covering the scenario. Same source citation as above. |

Both edge cases share one root cause: there is no function in `lib.rs` that ever changes
`Threshold` after `initialize`, and no function that validates it against the live member count
at any point afterward. Closing either finding requires either a `set_threshold` admin function
with its own validation, or a check inside `remove_member` that refuses to drop membership below
the stored threshold — neither exists today.

---

## 7. Double voting

| Contract | Guard | Test |
| --- | --- | --- |
| `group_treasury` (`approve_withdraw` / `reject_withdraw`) | `require_votable` checks `env.storage().instance().has(&DataKey::Vote(proposal_id, voter))` and panics `"already voted"` before recording a new vote — [`lib.rs:454-460`](../contracts/group_treasury/src/lib.rs#L454-L460) | ✅ `test_double_vote_panics` ([`test.rs:463`](../contracts/group_treasury/src/test.rs#L463)) and, specifically covering switching sides rather than repeating the same side, `test_approve_then_reject_same_member_panics` ([`test.rs:477`](../contracts/group_treasury/src/test.rs#L477)) — a member cannot flip from approve to reject either |
| `proposals` (`vote`) | Same pattern — `has(&vote_key)` check before recording — [`lib.rs:129-132`](../contracts/proposals/src/lib.rs#L129-L132) | ✅ `double_vote_panics` ([`test.rs:284`](../contracts/proposals/src/test.rs#L284)) |

No open finding on this item for either contract — both the guard and the "can't switch sides"
variant are covered by named, specific tests rather than inferred from the happy path.

One related, narrower gap worth a reviewer's attention even though it isn't "double voting" in
the counted sense: in `group_treasury`, a member removed after voting keeps their already-cast
vote counted in `approvals`/`rejections` — removal does not touch any `DataKey::Vote` or
`DataKey::Proposal` entry (documented in
[`concepts-treasury-multisig-model.md`](concepts-treasury-multisig-model.md#membership), "Member
removed during an active vote" row). This is a membership-consistency question, not a
double-voting one, and is called out here only so it isn't mistaken for covered by the tests
above.

---

## 8. Storage archival

Full coverage is in [`concepts-state-archival.md`](concepts-state-archival.md) — summarized
here for checklist purposes:

| Contract | Status |
| --- | --- |
| `token_transfer` | ⚠️ **Open finding.** All storage is `instance()`; no `extend_ttl`/`.bump()` call exists anywhere in the contract. |
| `group_treasury` | ⚠️ **Open finding**, same shape, and higher-impact: every `DataKey` variant (including `Members` and every open `Proposal`) shares the one instance TTL, so the whole treasury becomes unusable at once if it archives. |
| `proposals` | ⚠️ **Open finding**, same shape. |

No contract in this workspace extends any TTL. This is flagged once per contract here for
checklist completeness; see the linked document for the full per-key inventory, the risk
table, and the `stellar contract extend` / `stellar contract restore` operator commands.

---

## Summary table

| # | Item | `token_transfer` | `group_treasury` | `proposals` |
| --- | --- | --- | --- | --- |
| 1 | `require_auth` coverage | ✅ Pass | ✅ Pass (with the `propose_withdraw` caveat above) | ✅ Pass (two functions intentionally public) |
| 2 | Init front-running | ⚠️ Open — no auth at all | ⚠️ Open — no auth at all | ⚠️ Open — `require_auth` present but doesn't close the race |
| 2 | Re-initialization | ✅ Guarded & tested | ✅ Guarded & tested | ✅ Guarded, ⚠️ untested |
| 3 | Admin powers | Noted (upgrade = full logic replacement) | Noted (unilateral `withdraw` bypasses voting) | N/A — admin slot unused |
| 4 | Overflow | ⚠️ Mitigated by build flag only, untested | ⚠️ Same | ⚠️ Same |
| 5 | Zero/negative amounts | ⚠️ Zero tested, negative untested | ⚠️ Partial — see table | ⚠️ Untested |
| 6 | Threshold edge cases | N/A | ⚠️ Open — both scenarios unguarded | N/A — no threshold concept |
| 7 | Double voting | N/A | ✅ Pass | ✅ Pass |
| 8 | Storage archival | ⚠️ Open | ⚠️ Open | ⚠️ Open |

---

## Using this checklist on a PR

1. If the PR adds or changes a state-changing entry point, add a row for it to §1 and confirm
   its auth proof follows the pattern in the callout at the top of this document — a
   `mock_all_auths()`-only test is not sufficient evidence to mark it ✅.
2. If the PR touches `initialize` in any contract, re-read §2 — the front-running exposure is
   about the *absence* of a check, which a diff-based review easily misses if it only looks at
   what changed.
3. If the PR touches `contracts/Cargo.toml`'s `[profile.release]`, treat §4 as directly
   relevant regardless of what the PR description says it's about.
4. Carry forward every ⚠️ **Open finding** above into the PR description rather than silently
   re-approving it — this document records what is *currently* true, not what has been
   accepted as permanently acceptable.
