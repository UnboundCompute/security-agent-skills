---
name: auditing-account-abstraction-and-paymaster-trust
description: >-
  Audit an ERC-4337 account-abstraction deployment for trust misplaced in the user-operation lifecycle: a smart
  account whose validation accepts a signature or nonce it should reject, a paymaster that agrees to sponsor gas
  for operations it should not so an attacker drains its deposit, a bundler or entry-point assumption that lets
  a user operation be replayed or reordered for gain, and validation logic that reads mutable state or reaches
  outside its allowed scope. Covers smart-contract wallets, paymasters, bundlers, and the entry point in an
  account-abstraction stack where a user operation is validated and sponsored before it executes. Use when a
  user operation is validated, paid for, and executed by separate parties and that trust split is the boundary.
  The crafted user operation is the source, the drained paymaster or unauthorized execution is the sink, and the
  over-permissive validation or sponsorship rule is the bug.
license: MIT
---

# Auditing account abstraction and paymaster trust: validation, payment, and execution are three trusts

In an ERC-4337 stack a user operation is validated by the smart account, paid for by a paymaster, bundled by a
bundler, and executed through the entry point, and those are separate trust decisions made by separate parties.
Each is a place trust can be misplaced. The account's validation decides whether a signature and nonce are
acceptable; if it accepts what it should reject, an attacker executes operations as the account. The paymaster
decides whether to sponsor an operation's gas; if it sponsors what it should not, an attacker drains its
deposit by submitting operations that cost the paymaster and benefit no legitimate user, a denial-of-wallet on
the sponsor. The entry point and bundler order and submit operations; if an operation can be replayed or
reordered for gain, the lifecycle itself is the exploit. And validation runs under strict rules about what
state it may read, mutable or external state in validation is both a denial vector and a soundness hole. The
audit walks the user-operation lifecycle and checks each trust: what validation accepts, what the paymaster
agrees to pay for, and what the ordering allows. You audit this by crafting user operations that probe each
decision.

## When to use

- An ERC-4337 account-abstraction stack validates, sponsors, and executes user operations across a smart
  account, a paymaster, a bundler, and the entry point.
- A paymaster sponsors gas and could be made to pay for operations it should not.
- Account validation may accept a bad signature or nonce, or read mutable or out-of-scope state.

## Scope check

Test account-abstraction trust only against contracts and infrastructure you own or are authorized to assess,
on a testnet or a local fork. Submitting user operations and probing a paymaster spends real gas and can drain
a real deposit, so use test funds on a fork and never drain a live paymaster or execute against an account that
is not yours. If you can't name the authorization, stop.

## The loop

1. **Establish each trust decision first.** Name what the account's validation must accept and reject (which
   signer, which nonce), what the paymaster agrees to sponsor and under what limit, and what ordering the entry
   point and bundler guarantee. This is the false-positive killer: an account that validates a correct
   signature and monotonic nonce only, a paymaster that sponsors within a scoped, limited policy, and a
   lifecycle with no replay or reorder gain are behaving correctly. Name each trust, then probe it.

2. **Test account validation.** Craft user operations with a wrong or malformed signature, a reused or
   out-of-order nonce, and an unexpected signer, and confirm validation rejects each. An account that accepts a
   signature it should not, or reuses a nonce, lets an attacker execute operations as the account.

3. **Test paymaster sponsorship scope.** Submit operations the paymaster should not pay for: outside its
   intended users, actions, or limits, and confirm it declines. Then submit sponsored operations at volume and
   confirm a per-user and total limit bounds the deposit spend. A paymaster that sponsors broadly or without a
   spend cap is drainable, its deposit is the loss.

4. **Check validation-phase state rules.** Determine what the account's and paymaster's validation reads:
   whether it touches mutable storage or external contracts it should not during validation. Validation that
   depends on state a bundler cannot trust is both a griefing vector (operations that pass simulation but fail
   on-chain) and a soundness gap the attacker steers.

5. **Assess replay and ordering in the lifecycle.** Check whether a user operation can be replayed (across
   chains, entry points, or after a nonce reset) or reordered by the bundler for gain. Confirm the operation is
   bound to its chain, entry point, and nonce so it executes once, in one place. A replayable or
   reorder-sensitive operation is exploited by the party that sequences it.

6. **Confirm and record.** Confirm on a fork by executing an operation the account should have rejected,
   draining a scoped amount from a test paymaster, or replaying an operation, with test funds and without
   touching a live account or paymaster. Kill the lead if validation accepts only a correct signature and
   nonce, the paymaster sponsors within a scoped and capped policy, validation reads no untrusted state, and
   operations are bound against replay and reorder gain. Record the crafted user operation, the drained
   paymaster or unauthorized execution, and the over-permissive validation or sponsorship rule.

## Where account-abstraction trust leaks

- **Over-permissive account validation.** Accepting a bad or replayed signature, or a reused nonce, lets an
  attacker execute operations as the account.
- **Unscoped or uncapped paymaster sponsorship.** A paymaster that pays for operations outside its policy or
  without a spend limit is drainable; its deposit is the loss.
- **Untrusted state read in validation.** Validation that touches mutable or external state a bundler cannot
  trust enables griefing and steers soundness gaps.
- **Replayable user operations.** An operation not bound to its chain, entry point, and nonce can be replayed
  for repeated effect.
- **Reorder-sensitive lifecycle.** An operation whose outcome depends on bundler ordering hands value to
  whoever sequences it.

## Worked example (a confirm and a kill)

> **Confirm.** A paymaster sponsors any user operation that calls a particular application contract, with no
> per-user or total spend cap. On a fork, an attacker submits a flood of operations that each call that contract
> in a way that costs gas but does nothing useful, and the paymaster pays for all of them until its deposit is
> exhausted, denying sponsorship to real users. **Confirmed** paymaster deposit drain via unscoped sponsorship,
> `high`, remediation = scope sponsorship to authenticated users and specific actions, and enforce per-user and
> total spend caps with rate limiting so a flood cannot exhaust the deposit.
>
> **Kill.** The account validates only a correct signature from the expected signer with a monotonic nonce, the
> paymaster sponsors only scoped users and actions within per-user and total spend caps, validation reads no
> mutable or external state a bundler cannot trust, and every operation is bound to its chain, entry point, and
> nonce. A bad-signature operation is rejected, a sponsorship flood is capped, and a replay reverts. **Killed**,
> `kill_reason` = "validation accepts only a correct signature and nonce, paymaster sponsorship is scoped and
> capped, validation reads no untrusted state, and operations are replay- and reorder-bound; no operation
> executes or drains against policy."

## Rationalizations to reject

- *"The signature check is standard."* → Confirm it rejects malformed and replayed signatures and enforces the
  nonce; standard code with a nonce or domain mistake still lets operations through.
- *"The paymaster only sponsors our app."* → Scoping to a contract is not scoping to legitimate use; without
  per-user and total caps an attacker loops that contract to drain the deposit.
- *"Validation is simple, it just checks the signer."* → Confirm it reads no mutable or external state during
  validation; that is both a griefing and a soundness rule, not a style preference.
- *"Operations have nonces, so no replay."* → Verify the operation is also bound to the chain and entry point; a
  nonce alone does not stop cross-chain or cross-entry-point replay.
- *"The bundler is trusted."* → The bundler chooses order and can be anyone; design so no operation's outcome
  depends on it sequencing fairly.

## Executing this in practice

You need the account's validation logic (signature and nonce rules and what state it reads), the paymaster's
sponsorship policy and spend limits, and how operations are bound against replay and reorder. On a fork, submit
operations with bad signatures and reused nonces, probe the paymaster outside its policy and at volume, and
attempt replays. Reading the validation and paymaster contracts shows the intended trust; an operation that
executes or a deposit that drains against policy shows whether it holds.

## Related

- `hunting-signature-replay-and-eip712-domain-trust` - the signature and domain binding this skill checks in
  account validation is that skill's whole subject; audit them together for the account.
- `hunting-mev-and-transaction-ordering-exposure` - the bundler orders user operations, so ordering extraction
  reappears at the account-abstraction layer.
- `hunting-wallet-drainer-and-dapp-approval-abuse` - draining a paymaster and draining a user via approvals are
  two economic attacks on the same wallet stack.
- `reviewing-rate-limiting-and-abuse-controls` - the per-user and total spend caps that stop a paymaster drain
  are the rate-limiting discipline applied to gas sponsorship.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the crafted user operation, sink = the drained
  paymaster or unauthorized execution, evidence = the over-permissive validation or sponsorship rule.
