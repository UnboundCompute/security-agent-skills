---
name: hunting-smart-contract-reentrancy
description: >-
  Hunt a smart contract for state that is mutated after an external call, so an attacker re-enters before
  the update lands and acts on stale state. Covers a withdrawal or transfer that sends value before
  zeroing the balance, a call to an attacker-controlled contract or token that re-enters the same function,
  cross-function reentrancy where the callback re-enters a different function sharing the same state, a
  callback hook (a token receive hook, a fallback) that hands control to the attacker mid-update, and
  read-only reentrancy where a view another contract trusts returns mid-transaction state. Use when the
  fix would reorder effects before interactions or add a reentrancy guard, not add an authorization check
  (that is the access-control skill). The external call before the state update is the source, the
  re-entered function acting on stale state is the sink, and interaction-before-effects is the bug.
license: MIT
---

# Hunting smart-contract reentrancy: acting on state that has not been updated yet

Reentrancy is a control-flow bug: a contract makes an external call before it finishes updating its own
state, and the callee, which the attacker controls, calls back in and finds the old state still in place.
The classic shape is sending value before zeroing a balance, but it generalizes to any effect that lands
after an interaction, across functions that share state and through callback hooks that hand the attacker
control mid-update. This is a crowded topic, so the value is not naming the pattern but killing the lead
that a guard or an already-updated state defeats. The dividing line from access control: if the fix is to
reorder effects before interactions or add a mutex, it is reentrancy and belongs here; if the fix is to
add or repair an authorization check, it belongs in the access-control skill.

## When to use

- A function makes an external call (sends value, calls a token, invokes an attacker-supplyable address).
- State that the function or a sibling depends on is updated after that external call.
- You want to know whether an attacker can re-enter and act on state that has not been updated yet.

## Scope check

Analyze only contracts you own or are authorized to assess, and exercise a reentrancy path only on a test
network or a local fork, never against live value, an exploit here moves real funds irreversibly. Prove it
against a fork. If you can't name the authorization, stop.

## The loop

1. **Map external calls against state updates first.** For each function, find every external call (a value
   transfer, a token call, a call to a supplied address) and every state update the function and its
   siblings depend on, and record which comes first. Reentrancy exists only where an effect lands after an
   interaction; establish that ordering before anything else.

2. **Check single-function reentrancy.** Look for a function that performs an external call before updating
   the state that call's safety depends on, the withdraw-before-zeroing shape, so a re-entrant call sees the
   pre-update balance and repeats the withdrawal.

3. **Check cross-function reentrancy.** Look for a callback that re-enters a different function sharing the
   same state, where the first function has not yet committed its update, so the second acts on stale
   shared state even though neither function alone reorders its own effects.

4. **Check callback hooks.** Look for a token receive hook, a fallback, or any callback the contract
   invokes that hands control to attacker code in the middle of a state change, including standard token
   hooks that call the recipient before the sender's accounting settles.

5. **Check read-only reentrancy.** Look for a view function another contract trusts (a price, a share
   value, a balance) that returns mid-transaction state during a callback, so an integrating contract reads
   an inconsistent value even though this contract never writes during the re-entrant call.

6. **Confirm and record.** Confirm by showing a re-entrant call acts on stale state and gains value or
   breaks an invariant. Kill the lead if the function follows checks-effects-interactions so the state is
   already updated before the external call, if a reentrancy guard (a mutex modifier) protects the function
   and every sibling that shares the state, if the external call target is trusted and cannot re-enter (a
   fixed, known, non-attacker contract), if the call transfers no control (a low-level send with no code at
   the target, a pull pattern with no callback), or if the state read during a view is not relied on
   cross-contract. Record the function, the call-before-effect ordering, and the re-entry path.

## Where reentrancy leaks

- **The bug is the ordering, not the external call.** An interaction is fine after the effects land; the
  finding is an effect that lands after the interaction, so map the order before judging.
- **Cross-function reentrancy hides across two functions.** Each function may look ordered alone while the
  callback re-enters a sibling that shares uncommitted state; check the shared state, not one function.
- **Callback hooks hand over control mid-update.** A token receive hook or fallback runs attacker code
  before accounting settles; a standard hook is a reentrancy vector, not a safe primitive.
- **Read-only reentrancy poisons an integrator.** A view returning mid-transaction state misleads a
  contract that trusts it, even with no write during the re-entrant call.
- **A guard on some but not all siblings is not a guard.** A mutex has to cover every function that shares
  the state; one unprotected sibling reopens the path.

## Worked example (a confirm and a kill)

> **Confirm.** A withdraw function sends the caller their balance with an external value transfer, then sets
> their balance to zero; the recipient is a contract whose fallback calls withdraw again, and the second
> call sees the original non-zero balance and withdraws once more, draining the contract. **Confirmed**
> single-function reentrancy from interaction-before-effects, `critical`, remediation = zero the balance
> before the transfer (effects before interactions) and add a reentrancy guard.
>
> **Kill.** A withdraw function updates the balance to zero first and only then transfers value, and the
> function carries a reentrancy guard shared by every sibling that touches the balance. A re-entrant call
> finds the balance already zero and the guard engaged. **Killed**, `kill_reason` = "the function follows
> checks-effects-interactions and is guarded across all siblings, so re-entry acts on already-updated state."

## Rationalizations to reject

- *"It transfers to the user, who is not malicious."* -> The recipient can be a contract with a fallback that
  re-enters; the caller controls the callee, so assume it re-enters.
- *"Each function looks correctly ordered."* -> Does a callback re-enter a sibling sharing uncommitted state?
  Cross-function reentrancy spans two functions that each look fine alone.
- *"It is a standard token."* -> Does the token call a recipient hook before your accounting settles? A
  standard receive hook hands control to the attacker mid-update.
- *"The function is view, so it cannot be exploited."* -> Read-only reentrancy returns mid-transaction state
  to a contract that trusts it; a view can still poison an integrator.
- *"There is a reentrancy guard."* -> Does it cover every sibling sharing the state? A guard on one function
  and not its siblings leaves the cross-function path open.

## Executing this in practice

You need the functions and their external calls, the state each function and its siblings depend on and
when it is updated, the callback hooks the contract invokes, any reentrancy guard and which functions it
covers, and the views other contracts trust. For each function, decide whether an effect lands after an
interaction an attacker can re-enter. Reading the call order tells you where the window is; tracing the
re-entry across siblings and hooks tells you whether it is reachable.

## Related

- `auditing-smart-contract-access-control` - the split partner: if the fix is to add or repair an
  authorization check rather than reorder effects or add a mutex, adjudicate it there, not here.
- `hunting-defi-economic-and-oracle-flaws` - the economic-invariant sibling; a read-only reentrancy that
  feeds a manipulated value into a pricing decision hands off to that skill.
- `detecting-race-conditions` - the general concurrency analogue; reentrancy is the on-chain form of acting
  on state between a read and its commit.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the external call before the state update, sink = the
  re-entered function acting on stale state, evidence = the ordering and the re-entry path on a fork.
