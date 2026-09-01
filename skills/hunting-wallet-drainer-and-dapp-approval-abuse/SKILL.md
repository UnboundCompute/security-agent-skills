---
name: hunting-wallet-drainer-and-dapp-approval-abuse
description: >-
  Hunt for dApp flows that trick a user's wallet into signing away its assets: an unlimited or unnecessary token
  approval a user grants to a contract that can then move all of their tokens, a signed permit or
  approve-for-all that authorizes spending far beyond the intended action, a blind-signing prompt that hides
  what is really being authorized, a malicious or spoofed spender or contract address the user is led to
  approve, and a transaction whose displayed intent differs from what it actually executes. Covers dApp
  front ends, wallet-connection flows, and token-approval interactions where a user's signature or approval
  authorizes a contract to move their assets. Use when a user signs an approval or transaction and the gap
  between what they think they authorized and what they did is the boundary. The deceptive or overbroad approval
  request is the source, the drained or movable assets are the sink, and the unlimited scope or hidden intent is
  the bug.
license: MIT
---

# Hunting wallet drainer and dApp approval abuse: users sign what they are shown, not what they mean

A wallet drainer does not break cryptography; it gets the user to authorize the theft themselves. Every token a
dApp moves on a user's behalf requires an approval or a signed permit, and the danger is the gap between what
the user believes they are authorizing and what the approval actually grants. A user who wants to swap one
token often gets asked for an unlimited approval, granting the contract the right to move all of that token,
forever, not just the amount for this trade. A signed permit or an approve-for-all can authorize spending far
beyond the intended action. A blind-signing prompt shows an opaque blob instead of a human-readable action, so
the user cannot see what they are agreeing to. And a spoofed spender address, a malicious contract, or a
transaction whose displayed intent differs from its real effect turns a routine-looking click into a drain. The
hunt is to look at every approval and signature a dApp asks for and compare the scope and the real effect
against what the user is shown and actually needs. You hunt this by walking the approval flow as a user and
checking what each prompt truly authorizes.

## When to use

- A dApp front end asks users to grant token approvals, sign permits, or approve-for-all to interact with a
  contract.
- An approval may be unlimited or broader than the action needs, or a permit may authorize spending beyond the
  intended trade.
- A signing prompt may be a blind signature, or the spender or contract address may be spoofed or malicious.

## Scope check

Test approval flows only against dApps and contracts you own or are authorized to assess, on testnets or a
local fork with test wallets. Granting and exercising approvals moves real assets, so use test funds and never
approve, drain, or move assets from a wallet or contract that is not yours. If you can't name the
authorization, stop.

## The loop

1. **Establish what the action actually needs first.** Name the minimal authorization the intended interaction
   requires: which token, which spender, what amount, for how long. This is the false-positive killer: a flow
   that requests an exact-amount approval for the specific action, to a verified spender, with a human-readable
   prompt and a bounded lifetime, is asking for only what it needs. Name the minimal grant, then compare each
   prompt against it.

2. **Check approval scope against need.** For each approval, compare the amount and duration granted against
   what the action requires. An unlimited approval, or one far larger or longer-lived than the trade, hands the
   contract standing authority to move all of that token later. Confirm the flow requests exact-amount,
   short-lived approvals and offers revocation.

3. **Inspect permits and approve-for-all.** Examine any signed permit or approve-for-all: what it authorizes,
   to whom, and for how much. A permit signed for an unlimited amount, or an approve-for-all over an entire NFT
   collection, authorizes far beyond a single action and is a favorite drainer primitive. Confirm the signed
   scope matches the intended interaction.

4. **Test for blind signing and intent mismatch.** Look at what the signing prompt shows the user versus what
   the transaction or signature actually does. A prompt that shows an opaque blob (blind signing) or a
   displayed intent that differs from the real effect (a swap that is actually an approval, a mint that is
   actually a transfer) is deception. Confirm prompts are human-readable and match the real effect.

5. **Verify spender and contract identity.** Check the spender and contract addresses the user is asked to
   approve against the legitimate, verified addresses. A spoofed spender, a look-alike contract, or an address
   swapped in by a compromised front end routes the approval to an attacker. Confirm addresses are verified and
   displayed, not hidden or trusted blindly.

6. **Confirm and record.** Confirm on a test wallet by showing that an approval or permit grants more than the
   action needs and that the granted scope lets a contract move assets the user did not intend, on a fork with
   test funds and without draining a real wallet. Kill the lead if approvals are exact-amount and short-lived to
   verified spenders, permits match the intended action, prompts are human-readable and match the real effect,
   and revocation is offered. Record the deceptive or overbroad approval request, the drained or movable assets,
   and the unlimited scope or hidden intent.

## Where approval trust leaks

- **Unlimited approvals.** Granting an infinite allowance for a single trade leaves the contract able to move
  all of that token indefinitely.
- **Overbroad permits and approve-for-all.** A permit or collection-wide approval authorizes spending far beyond
  the intended action, a standing drain primitive.
- **Blind signing.** An opaque prompt the user cannot read hides what is being authorized, so they consent to
  something they never saw.
- **Intent mismatch.** A transaction whose displayed action differs from its real effect tricks the user into
  authorizing the effect, not the display.
- **Spoofed spender or contract.** A look-alike or swapped-in address routes a legitimate-looking approval to an
  attacker's contract.

## Worked example (a confirm and a kill)

> **Confirm.** A dApp swap flow requests an unlimited (max-uint) approval for the input token before the trade,
> and the swap contract is upgradeable by an admin. On a fork, after the user grants the unlimited approval, the
> contract (or a later upgrade) transfers the user's entire balance of that token, not just the swapped amount,
> because the standing unlimited allowance authorizes it. **Confirmed** drain via unlimited approval, `high`,
> remediation = request an exact-amount approval scoped to the trade, prompt the user with the real amount and
> spender, and surface and encourage revocation of standing allowances.
>
> **Kill.** The flow requests an exact-amount approval for the specific trade to the verified swap contract, any
> permit it asks the user to sign matches that amount and spender, the signing prompt is human-readable and its
> displayed intent equals the real effect, and the UI shows current allowances with a one-click revoke. There is
> no standing authority for the contract to move more than the trade. **Killed**, `kill_reason` = "approvals are
> exact-amount and short-lived to verified spenders, permits match the action, prompts are readable and match
> the real effect, and revocation is offered; no signature authorizes more than the user intended."

## Rationalizations to reject

- *"Unlimited approval is more convenient."* → Convenience is a standing drain; request exact-amount approvals
  per action and let the user re-approve, or the contract keeps the right to take everything.
- *"The user confirmed it."* → They confirmed what they were shown; a blind or mismatched prompt means their
  confirmation does not cover the real effect.
- *"It is a well-known dApp."* → A compromised or spoofed front end serves a malicious spender under a trusted
  brand; verify the spender and contract address, not the logo.
- *"Permits are gasless and safe."* → A permit is a signature that grants spending; an overbroad or unlimited
  permit is exactly as dangerous as an unlimited approval, and off-chain so easier to solicit.
- *"They can revoke later."* → Most users never do, and the drain happens before they think to; scope the grant
  at signing time, do not rely on later revocation.

## Executing this in practice

You need every approval, permit, and signature the flow requests, the amount and duration each grants, what the
signing prompt shows versus the real effect, and the spender and contract addresses involved. On a fork with
test wallets, walk the flow, inspect each prompt's real scope, and show whether the granted authority exceeds
the action. Reading the front-end and contract interaction shows the intended grant; a standing allowance that
lets the contract move assets the user did not mean to authorize shows the abuse.

## Related

- `hunting-signature-replay-and-eip712-domain-trust` - a permit is a signed message; its scope and binding are
  that skill's subject, and an overbroad permit is often also replayable.
- `hunting-mev-and-transaction-ordering-exposure` - what a user signs and submits is also exposed to ordering;
  approval abuse and front-running both exploit the signing-and-submission surface.
- `auditing-cross-chain-bridge-and-message-trust` - bridges and drainers are two ways value leaves a wallet;
  both hinge on what an action is really authorized to do.
- `auditing-clickjacking-and-ui-redressing` - a redressed dApp UI can trick a user into clicking a signing
  prompt they cannot see; UI deception and approval deception combine.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the deceptive or overbroad approval request, sink = the
  drained or movable assets, evidence = the unlimited scope or hidden intent.
