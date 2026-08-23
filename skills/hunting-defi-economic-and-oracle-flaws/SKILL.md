---
name: hunting-defi-economic-and-oracle-flaws
description: >-
  Hunt a decentralized-finance protocol for a way to profit by moving a price or breaking an economic
  invariant, rather than by defeating an access check or re-entering. Covers a price read from a spot
  source an attacker can move within a transaction, a manipulable oracle or a single-source feed with no
  sanity bound, a swap or liquidation that trusts a pool ratio a flash loan can skew, rounding or
  fee-accounting that a repeated interaction drains, share or collateral math that lets a first or
  precise depositor steal value, and a slippage or deadline parameter left unchecked. Use when the fix is
  an economic or pricing safeguard (a manipulation-resistant oracle, a bound, corrected math), not an
  authorization check or a reentrancy guard. The manipulable price or invariant is the source, the value
  the attacker extracts is the sink, and a value decision that trusts a movable input is the bug.
license: MIT
---

# Hunting DeFi economic and oracle flaws: profiting by moving a price nobody bounded

Some of the most valuable protocol bugs break no access check and re-enter nothing: they move a price or
exploit accounting math to extract value through actions the protocol permits. A contract reads a spot
price a flash loan can swing inside one transaction, trusts a pool ratio an attacker sets, or rounds a fee
in a direction a repeated call drains. This hunt is about the economic layer, the values a protocol trusts
and the invariants it assumes hold. It is distinct from access control (nothing here requires a missing
gate) and from reentrancy (nothing here requires re-entry); if the fix is a manipulation-resistant oracle,
a sanity bound, or corrected math, it is an economic flaw and belongs here. The discipline in a crowded
field is proving the manipulation is profitable after costs, not just that a price can move.

## When to use

- A protocol reads a price, a pool ratio, or a share value to make a swap, loan, or liquidation decision.
- Fee, rounding, or share math runs on values an attacker can influence within a transaction.
- You want to know whether an attacker can profit by moving a price or breaking an accounting invariant.

## Scope check

Analyze only protocols you own or are authorized to assess, and simulate an economic exploit only on a
fork or test network, a manipulation here extracts real value from real liquidity providers. Prove
profitability on a fork. If you can't name the authorization, stop.

## The loop

1. **Find the value decisions and their inputs first.** List the points where the protocol reads a price, a
   ratio, or a share value to move funds (a swap, a loan, a liquidation, a mint or redeem), and for each
   trace the input it trusts and whether an attacker can move that input within a transaction. The economic
   bug lives at a value decision that trusts a movable input; establish those before anything else.

2. **Check the price source.** Look for a price read from a spot source an attacker can move in the same
   transaction (a single pool's instantaneous ratio), a single-source oracle with no sanity bound or
   staleness check, or a feed that can be pushed outside a sane range without the protocol noticing.

3. **Check flash-loan-amplified manipulation.** Look for a decision that trusts a pool ratio or a balance an
   attacker can skew with borrowed capital for one transaction, swinging a price, a collateral valuation, or
   a liquidation threshold, then unwinding, so the attacker needs no capital of their own.

4. **Check accounting, rounding, and share math.** Look for a fee or reward computation that rounds in a
   direction a repeated interaction accumulates, share or collateral math that misprices the first or a
   precisely sized deposit, and an invariant (total shares versus total assets, debt versus collateral) that
   a sequence of permitted actions can break.

5. **Check slippage and deadline protection.** Look for a swap or trade that accepts an unchecked minimum
   output or an absent deadline, letting a sandwich or a delayed execution extract value the user did not
   agree to.

6. **Confirm and record.** Confirm by simulating the manipulation on a fork and showing a net profit after
   the cost of moving the price, the flash-loan fee, and gas. Kill the lead if the price comes from a
   manipulation-resistant source (a time-averaged or multi-source oracle with sanity and staleness bounds),
   if the decision already bounds or sanity-checks the value, if the rounding favors the protocol or the dust
   is economically negligible, if a share-inflation vector is closed (a minimum initial deposit, a dead-share
   mint, virtual offsets), if slippage and deadline are enforced, or if the manipulation costs more than it
   yields at realistic liquidity. Record the value decision, the movable input, and the profit path.

## Where economic flaws leak

- **The bug is a value decision trusting a movable input.** No gate is missing and nothing re-enters; the
  finding is a price or ratio the protocol trusts that an attacker can move, so start at the decision.
- **Spot price is manipulable by construction.** An instantaneous single-pool ratio moves within a
  transaction; reading it for a value decision is the flaw, not an edge case.
- **A flash loan removes the capital barrier.** Any manipulation that needs only single-transaction capital
  is in reach; do not dismiss a price swing as too expensive without pricing the flash loan.
- **Rounding in the wrong direction compounds.** A fee or share computation that rounds toward the caller is
  drained by repetition even when each increment looks negligible.
- **Profitability is the proof, not possibility.** A price that can move is a lead; the finding is a net
  profit after manipulation cost, flash-loan fee, and gas at realistic liquidity.

## Worked example (a confirm and a kill)

> **Confirm.** A lending market values collateral using the spot ratio of a single pool; an attacker takes a
> flash loan, swings that pool to inflate their collateral's apparent value, borrows against the inflated
> valuation, and repays the loan, leaving the market with bad debt. A fork simulation shows a net profit
> after fees and gas. **Confirmed** oracle manipulation via a spot-price collateral valuation, `critical`,
> remediation = value collateral with a manipulation-resistant, time-averaged, multi-source oracle with
> sanity and staleness bounds.
>
> **Kill.** A protocol reads a price that looks spot, but it comes from a time-averaged oracle over a window
> long enough that a single-transaction swing barely moves it, with a staleness check and a sanity bound
> rejecting outliers; a fork attempt to move the effective price nets a loss after the cost of holding the
> swing. **Killed**, `kill_reason` = "the price is a bounded time-averaged multi-source feed, so
> single-transaction manipulation is not profitable after cost."

## Rationalizations to reject

- *"The price comes from a real pool."* -> A single pool's spot ratio is movable within a transaction; a
  real pool is exactly what a flash loan swings. The source's manipulation resistance is what matters.
- *"Manipulating it would cost too much."* -> Priced with a flash loan? Single-transaction capital is cheap;
  compute the cost against the yield before dismissing it.
- *"The rounding is off by a tiny amount."* -> In which direction, and can it be repeated? Rounding toward
  the caller is drained by repetition regardless of the per-call size.
- *"The first deposit is an edge case."* -> First-depositor and precise-deposit share math is a known
  value-theft vector; confirm a minimum deposit, dead shares, or virtual offsets close it.
- *"The swap has slippage protection."* -> Is a real minimum output enforced and a deadline set? An unchecked
  minimum or absent deadline is a sandwich vector.

## Executing this in practice

You need the value decisions (swaps, loans, liquidations, mints, redeems), the price, ratio, and share
inputs each trusts and whether an attacker can move them in a transaction, the oracle sources and their
bounds, the fee, rounding, and share math, and the slippage and deadline handling. For each decision,
simulate whether moving the input yields a net profit after cost on a fork. Reading the code tells you
which input is trusted; the fork simulation tells you whether manipulating it pays.

## Related

- `hunting-smart-contract-reentrancy` - the sibling for control-flow re-entry, including a read-only
  reentrancy that feeds a mid-transaction value into a pricing decision this skill then adjudicates.
- `auditing-smart-contract-access-control` - the sibling for missing or defeated gates; a privileged
  parameter setter reachable by the wrong caller that skews pricing starts there and lands here.
- `hunting-business-logic-flaws` - the general analogue: extracting value through permitted actions in an
  unintended sequence, of which economic manipulation is the on-chain form.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the manipulable price or invariant, sink = the value
  the attacker extracts, evidence = a fork simulation showing net profit after manipulation cost.
