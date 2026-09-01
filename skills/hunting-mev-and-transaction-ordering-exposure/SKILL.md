---
name: hunting-mev-and-transaction-ordering-exposure
description: >-
  Hunt for value a validator or searcher can extract by controlling the order of transactions in a block: a
  swap or trade with no slippage bound that a sandwich attack front-runs and back-runs, an oracle update or
  liquidation whose profit depends on being sequenced first, an on-chain action that leaks its intent to the
  public mempool before it settles, and a protocol that assumes fair ordering when block producers choose it.
  Covers DeFi swaps, AMMs, lending liquidations, auctions, and any on-chain flow whose outcome depends on where
  its transaction lands in the block. Use when the profit or safety of an on-chain action depends on ordering
  that the submitter does not control. The attacker-ordered or front-run transaction is the source, the
  extracted value or failed action is the sink, and the missing slippage bound or ordering assumption is the bug.
license: MIT
---

# Hunting MEV and transaction-ordering exposure: the block producer decides the order, not you

On a public blockchain the party that builds the block chooses the order of the transactions in it, and any
value that depends on that order is value they, or a searcher paying them, can take. A user who submits a swap
does not control whether it lands before or after someone else's, and the pending transaction sits in a public
mempool where its intent is visible before it settles. That combination is the whole class. A swap with no
slippage bound can be sandwiched: an attacker buys ahead of it to push the price, lets the victim's trade
execute at the worse price, and sells behind it, pocketing the difference. A liquidation or an oracle-dependent
action whose profit goes to whoever is sequenced first is a race the submitter usually loses to a searcher who
pays for priority. And any strategy that reveals its intent in the mempool, a large trade, an arbitrage, a
governance action, invites front-running. The hunt is to find on-chain actions whose outcome depends on
ordering the submitter cannot control, and no protective bound. You hunt this by reading the transaction's
assumptions about order and price and checking whether an adversarial ordering breaks them.

## When to use

- An on-chain action's profit or safety depends on where its transaction lands in the block (a swap,
  liquidation, oracle update, auction bid).
- Transactions are submitted to a public mempool where their intent is visible before they settle.
- A trade may carry no slippage or deadline bound, or a protocol may assume ordering the producer controls.

## Scope check

Test ordering exposure only against protocols and deployments you own or are authorized to assess, on a
testnet or a local fork. Front-running, sandwiching, and priority races execute real on-chain value movement,
so use test funds on a fork and never extract from or disrupt a live protocol or another party's transaction.
If you can't name the authorization, stop.

## The loop

1. **Establish what the action assumes about ordering and price first.** Name the outcome the submitter
   expects, the price or state they assume at execution, and any bound that protects them: a slippage limit, a
   deadline, a commit-reveal step, or private submission that keeps intent out of the public mempool. This is
   the false-positive killer: an action that carries a tight slippage bound and deadline, or that does not
   depend on ordering at all, is not exposed. Name the assumption, then test an adversarial ordering.

2. **Test for missing slippage and deadline bounds.** For each swap or trade, check whether it sets a minimum
   received amount (slippage) and a deadline. A trade with no or a loose slippage bound executes at whatever
   price the surrounding transactions leave, which is exactly what a sandwich exploits. Model the sandwich: a
   front-run that moves the price, the victim trade at the worse price, a back-run that restores it.

3. **Hunt sequencing-dependent profit.** Identify actions whose reward goes to whoever is ordered first:
   liquidations, arbitrage, oracle updates, auction settlements. Check whether the protocol assumes a fair
   order it cannot enforce, and whether a searcher paying for priority captures the value ahead of the intended
   actor. Sequencing-dependent profit with no protection is MEV waiting to be taken.

4. **Check mempool intent leakage.** Determine what a pending transaction reveals before it settles: the trade
   size, the target, the strategy. Confirm whether sensitive actions use private submission or a commit-reveal
   scheme, or whether their intent is public and front-runnable. Leaked intent turns any profitable action into
   a race the submitter is likely to lose.

5. **Assess protocol-level ordering assumptions.** Look for logic that only holds under an ordering the block
   producer is free to violate: a price read and an action assumed to be atomic across transactions, a
   first-come reward, a state that another transaction can change between read and use. An assumption of fair
   ordering in an adversarial-ordering environment is the underlying bug.

6. **Confirm and record.** Confirm on a fork by executing the adversarial ordering: sandwiching an unbounded
   swap, winning a sequencing race for a liquidation, or front-running a mempool-visible action, with test
   funds and without touching a live protocol. Kill the lead if the action carries a tight slippage bound and
   deadline, uses private or commit-reveal submission where intent matters, and does not depend on ordering the
   producer controls. Record the attacker-ordered transaction, the extracted value or failed action, and the
   missing slippage bound or ordering assumption.

## Where ordering exposure leaks

- **Swaps with no slippage bound.** A trade that accepts any output price is sandwichable: front-run to move
  the price, victim executes at the worse price, back-run to profit.
- **First-sequenced-wins profit.** Liquidations, arbitrage, and oracle-dependent rewards that go to whoever is
  ordered first are races a priority-paying searcher wins.
- **Public mempool intent.** A pending transaction that reveals size, target, or strategy invites front-running
  before it settles.
- **Assumed atomicity across transactions.** Logic that reads a price and acts assuming nothing reorders
  between them breaks when the producer inserts a transaction in the gap.
- **Loose deadlines.** A trade with a distant or absent deadline can be held and executed later at a moment
  advantageous to the producer, not the submitter.

## Worked example (a confirm and a kill)

> **Confirm.** A swap function on an AMM accepts a minimum-output parameter but the front end submits it as zero,
> so the trade executes at any price. On a fork, a front-run buy moves the pool price, the victim's swap fills at
> the degraded price, and a back-run sell restores the pool, extracting the spread from the victim. **Confirmed**
> sandwich via missing slippage bound, `high`, remediation = enforce a non-zero minimum-output (slippage) bound
> and a near-term deadline on every swap, and consider private submission for large trades so intent does not
> hit the public mempool.
>
> **Kill.** Every swap enforces a tight minimum-output slippage bound and a near-term deadline, large or
> sensitive trades are submitted privately rather than to the public mempool, and no protocol reward depends on
> an ordering the producer controls without protection. A modeled sandwich fails because the victim trade
> reverts when the price moves past the slippage bound. **Killed**, `kill_reason` = "swaps carry tight slippage
> and deadline bounds, sensitive intent is not exposed in the mempool, and no profit depends on producer-chosen
> ordering; an adversarial ordering extracts nothing."

## Rationalizations to reject

- *"The user sets slippage in the UI."* → Confirm the submitted transaction actually carries a non-zero bound;
  a default of zero or a very loose value is the same as none.
- *"Transactions confirm fast."* → Speed does not change ordering; the producer still chooses order within the
  block, and a searcher bids for position regardless of block time.
- *"Our mempool is private."* → Verify it end to end; many paths still route through public relays where intent
  leaks, and a private path with a public fallback leaks on fallback.
- *"MEV only hurts big traders."* → Sandwiching scales down to ordinary trades, and sequencing races on
  liquidations and arbitrage hit anyone relying on fair order.
- *"The contract is audited."* → Ordering exposure is an environment property, not a contract bug an audit for
  reentrancy or overflow necessarily covers; check the slippage and ordering assumptions specifically.

## Executing this in practice

You need each on-chain action's slippage and deadline bounds, whether its profit depends on being sequenced
first, what its pending transaction reveals in the mempool, and any commit-reveal or private-submission
protection. On a fork, model the sandwich and the sequencing race with test funds and observe whether the
protective bounds hold. Reading the swap and settlement code shows the ordering and price assumptions; a
successful extraction on the fork shows whether they survive an adversarial order.

## Related

- `hunting-signature-replay-and-eip712-domain-trust` - a signed order or permit is another on-chain action an
  attacker can order or replay to advantage; the two meet at transaction submission.
- `hunting-wallet-drainer-and-dapp-approval-abuse` - front-running and approval abuse both exploit what a user
  signs and submits; ordering exposure is the timing half.
- `auditing-cross-chain-bridge-and-message-trust` - cross-chain messages have their own ordering and replay
  assumptions that a sequencer or relayer can violate.
- `auditing-account-abstraction-and-paymaster-trust` - bundlers order user operations, reintroducing
  ordering-extraction at the account-abstraction layer.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the attacker-ordered or front-run transaction, sink =
  the extracted value or failed action, evidence = the missing slippage bound or ordering assumption.
