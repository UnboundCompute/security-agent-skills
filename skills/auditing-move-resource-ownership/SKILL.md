---
name: auditing-move-resource-ownership
description: >-
  Audit a Move smart contract (Aptos or Sui) for a public entry function or a passed object or resource
  that acts without verifying signer authority, ownership, or capability possession, after the function
  visibility and the ability set are resolved. Covers a public entry function with no signer-authority
  check, an object or resource whose ownership is not verified before it is acted on, a capability returned
  or stored so it can leak and escalate privilege, an ability (key, store, copy, drop) granted too broadly
  and enabling duplication, arithmetic that overflows without aborting, and an init or upgrade path leaving
  mutable authority. Use when reviewing module functions, their visibility, and the resource and ability
  model, not the EVM account-model or reentrancy checks their own skills own. Any signer reaching the
  function is the source, a state change acting on an unowned resource is the sink, and a missing ownership
  or authority assertion is the bug.
license: MIT
---

# Auditing Move resource ownership: acting on a resource without proving the caller owns it

Move's safety rests on linear resources and abilities, and the bug is a function that acts on a resource
or an object without proving the caller is entitled to it. A public entry function with no signer-authority
check, an object passed in and mutated without verifying its owner, a capability resource that leaks
because it can be stored or returned, an ability granted too broadly, these are the ways an arbitrary
signer reaches a privileged state change. You audit it by resolving each function's visibility (who can
call it) and the ability set of the types it touches (what can be copied, stored, or dropped), then
checking whether the function asserts ownership or authority before it acts. The account-model and
reentrancy shapes belong to the EVM smart-contract skills; this skill owns the resource-and-ability model
of Move.

## When to use

- You are reviewing a Move module's functions, their visibility, and the resources and objects they touch.
- A public or entry function acts on a passed object or a global resource, or hands out a capability.
- You want to know whether an arbitrary signer can drive a state change on a resource they do not own.

## Scope check

Audit only contracts you own or are authorized to assess, and submit a transaction only against a network
in scope (a devnet or testnet, or a mainnet only with authorization), a call that passes moves real value.
Adjudicate on the module source. If you can't name the authorization, stop.

## The loop

1. **Resolve the visibility and the ability set first.** For each function, determine who can call it
   (public, public entry, public-to-friend or package-internal, or private) and for each type it touches
   determine the abilities it carries (key, store, copy, drop). Visibility decides whether an arbitrary
   signer reaches the function, and the abilities decide whether a resource or capability can be duplicated
   or leaked, so settle both before flagging.

2. **Check access control on entry functions.** Look for a public entry function that performs a privileged
   action (moving funds, changing admin state, minting) with no assertion that the signer is the owner or
   an authorized admin, so any signer can call it.

3. **Check object and resource ownership.** Look for a function that receives an object or borrows a global
   resource and acts on it without verifying the caller owns it, so a caller passes an object they do not
   own (or targets an address they do not control) and the function mutates it.

4. **Check capability handling.** Look for a capability resource returned to the caller, stored where it can
   be read or taken, or granted without a guard, so possession of the capability, which should be tightly
   held, leaks and escalates privilege.

5. **Check abilities, arithmetic, and lifecycle.** Look for an ability granted too broadly (a value type
   that is copyable or droppable when it should be linear, enabling duplication or silent discard),
   arithmetic that overflows on a runtime or toolchain that does not abort, and an init or upgrade path
   that leaves state re-initializable or authority mutable after deployment.

6. **Confirm and record.** Confirm by constructing a transaction from an unrelated signer that reaches the
   function and drives the state change on a resource the signer does not own. Kill the lead if the function
   asserts the signer's address against the owner or an admin capability before acting, if the object
   parameter is guarded by an ownership assertion or is a shared object with correct access rules, if the
   capability is a witness consumed at initialization and has no store ability so it cannot be held, if the
   arithmetic uses checked helpers or a runtime that aborts on overflow, if the framework enforces
   once-only initialization so the init is not re-callable, or if the function is package-internal or
   friend-only and not reachable by an arbitrary caller. Record the function, its resolved visibility, and
   the ownership or authority assertion it skips.

## Where resource ownership leaks

- **Visibility decides reachability.** A public entry function is callable by any signer; a friend-only or
  package-internal function is not, so resolve visibility before calling a missing check exploitable.
- **A passed object is not a proof of ownership.** Receiving an object or borrowing a global resource does
  not mean the caller owns it; the function has to assert the signer's authority over it before acting.
- **A capability is only safe while it cannot leak.** A capability with a store ability that is returned or
  stored can be taken; one consumed at init with no store ability cannot be held, which is the safe shape.
- **Abilities are the duplication surface.** A value type that is copyable or droppable when it should be
  linear lets value be duplicated or silently discarded; the ability set is part of the authorization.
- **Init and upgrade are authority seams.** Re-initializable state or a mutable post-deployment authority
  lets a later caller seize control; once-only init and a fixed authority close it.

## Worked example (a confirm and a kill)

> **Confirm.** A public entry function `withdraw` takes a mutable reference to a shared vault object and
> transfers funds to the signer, with no check that the signer owns the vault. Any signer submits the call
> against another user's vault and drains it. **Confirmed** a missing ownership assertion on a public entry
> function acting on an object the caller does not own, `critical`, remediation = assert the signer's
> address (or an owner capability) against the vault owner before the transfer.
>
> **Kill.** The same function begins with an assertion that the vault's owner field equals the transaction
> sender, aborting otherwise, so only the owner can withdraw. A call from an unrelated signer aborts before
> the transfer. **Killed**, `kill_reason` = "the function asserts the vault owner against the transaction
> sender before the state change, so a caller cannot act on a vault they do not own."

## Rationalizations to reject

- *"It is an entry function, that is normal."* -> Entry means any signer can call it; does it assert the
  caller's authority before acting? Reachable plus unguarded is the bug.
- *"The caller passes the object in."* -> Passing an object is not owning it; the function must verify the
  signer owns the object or the target address, not trust the parameter.
- *"The capability is internal."* -> Does its type have the store ability, and is it ever returned or
  stored? A storable capability can leak; a witness consumed at init cannot.
- *"Move is safe by construction."* -> Its linearity holds only with the right ability set; a copyable or
  droppable value that should be linear breaks the guarantee.
- *"Init only runs once."* -> Does the framework enforce that here, or can the state be re-initialized? A
  re-initializable init or a mutable authority is an ownership seam.

## Executing this in practice

You need the module source, each function's visibility, the ability set of every type it touches, the
ownership or authority assertions before each state change, the capability types and whether they can be
stored or returned, the arithmetic and whether the runtime aborts on overflow, and the init and upgrade
paths. For each function, decide whether an arbitrary signer can drive a state change on a resource they do
not own. Reading the visibility and abilities tells you what is reachable and duplicable; reading the
assertions tells you what is guarded.

## Related

- `auditing-smart-contract-access-control` - the EVM sibling for adding or repairing an authorization gate;
  the same missing-authority shape, on the account model rather than Move's resource-and-ability model.
- `hunting-smart-contract-reentrancy` - the EVM sibling for effect ordering; Move's linear resources remove
  the classic reentrancy analog, so that shape lives there, not here.
- `hunting-defi-economic-and-oracle-flaws` - the economic layer that a confirmed ownership bug on a
  value-bearing Move object often feeds; profitability on a fork is the proof shared with that skill.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = any signer reaching the function, sink = a state
  change acting on an unowned resource, evidence = the resolved visibility and ability set and the missing
  ownership or authority assertion.
