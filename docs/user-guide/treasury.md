# Group treasury and proposals

How a group's shared treasury works in Clicked today: depositing funds, proposing a
withdrawal, approving or rejecting it, what happens once enough members agree, and what
happens if a proposal is rejected or simply runs out of time.

This is written against the current **Treasury** page (`/app/treasury`) and its **Propose
Withdrawal** and proposal-voting controls. Clicked also has a separate **Governance
Proposals** page — that one covers unrelated, protocol-level community proposals (e.g.
"increase validator quorum"), not your group's own treasury. Nothing on that page affects your
group's funds; this guide only covers the Treasury page's proposals.

---

## The short version

A group treasury is a shared pool of funds that belongs to the group, not to any one person —
not even Clicked. Moving money out of it requires enough of the group's members to agree, the
same way a safe with multiple locks needs enough of the keyholders to show up. Clicked's job is
to show you the treasury's state and let you propose, vote, and (eventually) release funds —
the money itself sits in a smart contract on the Stellar network the whole time, not in a
Clicked-controlled account.

## Membership and the approval threshold, in plain terms

Every treasury has:

- A **member list** — the specific people allowed to propose withdrawals and vote on them.
  Being in the group conversation doesn't automatically make you a treasury member; membership
  is managed separately by the treasury's admin.
- A **threshold** — the number of approval votes a withdrawal needs before it's allowed to go
  through. This is a fixed number, not a percentage, and it doesn't automatically track the
  member count if membership changes later (more on that below).

### Worked example: three members, threshold of two

Say a group treasury has three members — Alex, Bola, and Chidi — and a threshold of **2**.

1. **Alex proposes a withdrawal** of 500 XLM to pay a vendor. Proposing something you believe
   in counts as your own approval automatically — Alex's proposal starts at **1 approval out
   of 2 needed**.
2. **Bola approves.** That's the second approval — **2 out of 2** — so the proposal is now
   approved and ready to be carried out. Chidi's vote was never needed for this one; once a
   proposal hits the threshold, it's approved regardless of how the remaining members would
   have voted.
3. If instead **Chidi had rejected** the proposal before Bola voted, that single rejection
   wouldn't kill it by itself — rejecting is a vote against, not a veto. A proposal is only
   blocked once enough people reject it that reaching the threshold becomes mathematically
   impossible. With 3 members and a threshold of 2, that point is reached at **2 rejections**:
   once 2 of the 3 members have said no, there's no way for 2 approvals to still happen, so the
   proposal is blocked. A single "no" vote, on its own, just lowers the odds — it doesn't end
   the vote.

**Why a threshold, not just "everyone agrees"?** Requiring unanimous agreement means one
unavailable or unresponsive member can freeze the treasury indefinitely. Requiring only one
person's approval means any single member could drain the treasury alone. A threshold in
between is the group's own choice of how much agreement is enough — set when the treasury was
created, not something you choose per withdrawal.

**A subtlety worth knowing:** the threshold is a fixed number, set once. If members later leave
the group's treasury roster, the threshold doesn't shrink to match — a treasury with a
threshold of 2 still needs 2 approvals even if it's down to 2 members total, at which point
every remaining member must agree on everything. Ask whoever administers your treasury what the
current threshold and member count are if this matters to a decision you're making.

## Funds live on-chain, not inside Clicked

The treasury's balance isn't a number in a Clicked database that Clicked could edit or lose —
it's held by a smart contract on the Stellar network. Clicked's role is to show you that
balance, let members propose and vote, and (once a proposal is approved) trigger the on-chain
transfer. The money itself moves by the contract transferring tokens from its own on-chain
balance to the recipient, authorized by the vote outcome and the admin, not by anything
Clicked's servers decide on their own.

**What this means if Clicked is ever down or unreachable:** your group's treasury balance is
not at risk — it continues to exist on the Stellar network exactly as it did before, independent
of whether Clicked's app or servers are running. What you'd lose access to is the *convenience*
of Clicked's interface for viewing the balance, proposing, and voting through chat. The funds
themselves aren't "inside" Clicked in any sense that an outage could threaten.

## Why you're asked to sign with Freighter

Freighter is the Stellar wallet extension Clicked uses to prove actions are really coming from
you, not just from whoever is logged into your Clicked account at the time.

When you **approve or reject** a proposal, Clicked asks Freighter to sign a short message that
names the specific proposal and the specific choice you're making (approve or reject). You'll
see a Freighter popup asking you to confirm — that's you personally authorizing that exact
vote, in a way that can't be reused for a different proposal or a different vote. If you
cancel or close the Freighter popup, your vote is not recorded.

This signature is separate from your Clicked login. Your login proves you're allowed to use
the app; your Freighter signature proves it was genuinely you, wallet in hand, who approved or
rejected that specific withdrawal — the same reason a bank asks for a card PIN even after
you've already shown ID.

(Proposing a withdrawal and viewing the treasury don't currently prompt a Freighter signature
the way voting does — those actions go through your logged-in Clicked session. If that changes,
this guide will note it.)

## Proposal expiry — what happens to the money

When you propose a withdrawal, you also pick how long it stays open for voting (today: 24
hours, 72 hours, or 7 days). This is strictly a voting deadline — it has nothing to do with the
funds being set aside or locked anywhere.

**No money moves when a proposal is created, and none moves while it's pending.** Proposing a
withdrawal doesn't reserve, freeze, or earmark the amount out of the treasury's balance — the
treasury's full balance stays available the entire time a proposal is being voted on. This
matters for what happens at the end of a proposal's life:

- **If it's approved** (reaches the threshold before expiring): the withdrawal can go ahead,
  and the funds move only at that point — not earlier.
- **If it's rejected** (enough members vote no — see the worked example above): nothing
  happens to the treasury's balance. The money was never moved in the first place, so there's
  nothing to "return." The proposal simply ends without ever becoming a transfer.
- **If it expires** with neither outcome reached (not enough approvals, not enough rejections):
  same as rejection, as far as the treasury's balance is concerned — nothing moves, and the
  proposed withdrawal simply does not happen. The amount stays in the treasury, fully available
  for a new proposal if the group still wants to make that payment.

In short: a withdrawal proposal is a request and a vote, not a hold on the money. The treasury
only ever loses funds at the moment an approved withdrawal is actually carried out — rejection
and expiry both just mean that moment never arrives for that particular proposal.
