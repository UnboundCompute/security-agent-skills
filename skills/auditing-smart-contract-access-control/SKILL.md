---
name: auditing-smart-contract-access-control
description: >-
  Audit a smart contract for a privileged action any caller can reach, so an attacker invokes a function
  that should be restricted. Covers a state-changing or fund-moving function missing an authorization
  modifier, an ownership or role check that is wrong or bypassable, an unprotected initializer that lets an
  attacker seize ownership of a proxy or an uninitialized contract, a delegatecall to an attacker-supplied
  or upgradeable target that runs foreign code in this contract's context, a self-destruct or upgrade
  reachable without the right role, and a role granted to an address that should not hold it. Use when the
  fix would add or repair an authorization check, not reorder effects before interactions (that is the
  reentrancy skill). The attacker calling a privileged function is the source, the restricted action
  executing for them is the sink, and a missing or defeated authorization check is the bug.
license: MIT
---

# Auditing smart-contract access control: who is allowed to call the privileged function

Access control on a contract is a set of gates: a modifier or a check that says only the owner, only a
role, only during initialization. The bug is a privileged action, moving funds, changing configuration,
upgrading, self-destructing, whose gate is missing, wrong, or reachable by the wrong caller. This is a
crowded audit area, so the discipline is confirming the action is genuinely privileged and the gate is
genuinely absent or defeated on a reachable path, not flagging every public function. The dividing line
from reentrancy: if the fix is to add or repair an authorization check, it belongs here; if the fix is to
reorder effects before interactions or add a mutex, it belongs in the reentrancy skill. Two initialization
and delegation shapes, the unprotected initializer and the delegatecall to an attacker-controlled target,
are access-control bugs and are audited here.

## When to use

- A function changes state, moves funds, upgrades, self-destructs, or grants a role.
- You see an authorization modifier, an ownership or role check, an initializer, or a delegatecall.
- You want to know whether an attacker can reach a privileged action the contract meant to restrict.

## Scope check

Audit only contracts you own or are authorized to assess, and exercise a privileged path only on a test
network or a local fork, calling an unguarded privileged function on a live contract can seize or destroy
real value. Prove it on a fork. If you can't name the authorization, stop.

## The loop

1. **Inventory the privileged actions and their gates first.** List every function that moves funds,
   changes configuration, upgrades, self-destructs, or grants a role, and for each find the gate that is
   meant to restrict it (an owner or role modifier, an internal check). A function is only a finding if it
   is genuinely privileged and its gate is absent, wrong, or reachable by the wrong caller; establish both
   before flagging.

2. **Check for the missing modifier.** Look for a privileged, state-changing, or fund-moving function with
   no authorization modifier and no internal caller check, callable by any address. A public function that
   changes ownership-relevant state with no gate is the core shape.

3. **Check the gate's correctness.** Look for an ownership or role check that is present but wrong: the
   wrong role, a comparison that can be satisfied by the caller, a check on a value the caller controls, or
   a modifier applied to a harmless function while the sensitive one nearby lacks it.

4. **Check the initializer.** Look for an initializer on a proxy or an upgradeable or uninitialized
   contract that is not protected against being called by anyone, letting an attacker run it first and seize
   ownership or set attacker-chosen configuration. An unprotected initializer is an access-control bug.

5. **Check delegatecall and privileged primitives.** Look for a delegatecall to an attacker-supplied or
   attacker-influenceable target, which runs foreign code in this contract's storage and permission context,
   and for a self-destruct or an upgrade reachable without the correct role, and for a role or ownership
   grant to an address that should not hold it.

6. **Confirm and record.** Confirm by showing an unauthorized caller reaches the privileged action on a
   fork. Kill the lead if a correct authorization modifier or internal check gates the function on the
   reachable path, if the initializer is protected (an initializer guard, constructor-only, or already
   initialized so it cannot be re-run), if the delegatecall target is fixed and trusted and not
   attacker-influenceable, if the function is privileged in name but changes nothing sensitive and moves no
   value, or if the caller the check admits is genuinely the intended trusted party. Record the function,
   the missing or defeated gate, and the caller that reaches it.

## Where access control leaks

- **A public privileged function is the surface, not every public function.** The finding is a genuinely
  sensitive action with no working gate; confirm the action is privileged before flagging the visibility.
- **A present check can still be the wrong check.** A modifier on the wrong role, or a comparison the caller
  satisfies, gates nothing; read what the check actually enforces, not that one exists.
- **An unprotected initializer hands over ownership.** On a proxy or uninitialized contract, an open
  initializer lets an attacker run it first and become owner; this is access control, audited here.
- **A delegatecall runs foreign code in your context.** A delegatecall to an attacker-influenceable target
  executes with this contract's storage and permissions; the target's controllability is the bug.
- **A guard on the wrong function is misdirection.** The modifier on a harmless function while the sensitive
  sibling nearby is bare is a common shape; check the gate is on the action that matters.

## Worked example (a confirm and a kill)

> **Confirm.** A proxy's initializer sets the owner and is not protected against repeated or public calls,
> and the contract was deployed without initialization; an attacker calls the initializer first, becomes
> owner, and then invokes the owner-gated upgrade to point the proxy at their implementation. **Confirmed**
> ownership seizure through an unprotected initializer, `critical`, remediation = protect the initializer so
> it runs once and only during deployment, and verify initialization at deploy.
>
> **Kill.** A function that transfers protocol fees looks unguarded at first read, but it carries an
> owner-only modifier resolved through the contract's access-control base, and only the deployer address
> satisfies it. An arbitrary caller reverts. **Killed**, `kill_reason` = "the fee-transfer function is gated
> by a correct owner-only modifier on the reachable path, so no unauthorized caller can execute it."

## Rationalizations to reject

- *"The function is public, so it is a bug."* -> Only if it is privileged and ungated. Confirm it changes
  sensitive state or moves value and has no working check before flagging visibility alone.
- *"It has an onlyOwner modifier."* -> Does the modifier enforce the right role, and is it on the sensitive
  function rather than a harmless neighbor? A present check can still be the wrong check.
- *"The initializer is fine, it sets up the contract."* -> Is it protected from being called by anyone on an
  uninitialized proxy? An open initializer is an ownership-seizure primitive.
- *"It uses delegatecall for upgrades."* -> Is the target attacker-influenceable? A delegatecall to a
  controllable target runs foreign code in this contract's context.
- *"Only the team would call it."* -> Does the code enforce that, or just assume it? Intent is not a gate; the
  check has to restrict the caller on-chain.

## Executing this in practice

You need the privileged functions and what each one does, the authorization modifiers and internal checks
and what they actually enforce, the initializer and whether it is protected, the delegatecall targets and
their controllability, and the roles and who holds them. For each privileged action, decide whether an
unauthorized caller reaches it on the live path. Reading the function tells you what it does; reading the
gate tells you whether it stops the wrong caller.

## Related

- `hunting-smart-contract-reentrancy` - the split partner: if the fix is to reorder effects before
  interactions or add a mutex rather than add or repair an authorization check, adjudicate it there.
- `hunting-defi-economic-and-oracle-flaws` - the economic sibling; a privileged parameter setter reachable
  by the wrong caller that skews pricing hands the economic impact to that skill.
- `finding-fail-open-flaws` - the general shape of a check that does not deny; an access gate that admits the
  wrong caller is its on-chain instance.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the attacker calling a privileged function, sink = the
  restricted action executing for them, evidence = the missing or defeated gate and the reachable caller.
