---
name: hunting-signature-replay-and-eip712-domain-trust
description: >-
  Hunt for signed messages a contract or backend accepts more than once or in a context they were never meant
  for: an EIP-712 signature with no nonce so it replays, a signature missing chain id or verifying-contract in
  its domain separator so it replays across chains or deployments, a permit or meta-transaction reused after it
  was already consumed, a signature whose signed fields omit something the action depends on so a different
  action reuses it, and a signer recovery that accepts a malleable or zero-address signature. Covers on-chain
  signature verification, permits, meta-transactions, and off-chain-signed orders where a signature authorizes
  an action. Use when a signed message authorizes an action and the binding of that signature to one action,
  one chain, and one use is the boundary. The reused or cross-context signature is the source, the repeated or
  unintended authorized action is the sink, and the missing nonce or domain binding is the bug.
license: MIT
---

# Hunting signature replay and EIP-712 domain trust: a signature authorizes one action, once, here

A signature is an authorization, and an authorization has to be bound to exactly what it authorizes: one
action, on one chain, at one contract, used once. When any part of that binding is missing, the same signature
authorizes more than the signer intended. EIP-712 exists to make the binding explicit, the domain separator
carries the chain id and the verifying contract, and the typed data carries the action's fields, but a
contract only gets the protection it actually checks. A signature with no nonce can be submitted again, so a
permit or meta-transaction replays. A domain separator missing the chain id or the verifying contract lets a
signature valid on one chain or one deployment be replayed on another. Signed fields that omit something the
action depends on, an amount, a recipient, a deadline, let a different action reuse the same signature. And
signer recovery that accepts a malleable signature or an unchecked zero address lets a forged or duplicated
signature through. The hunt is to find where a signed message is verified and check whether its binding is
complete. You hunt this by taking a valid signature and trying to use it twice, elsewhere, or for something
else.

## When to use

- A signed message (an EIP-712 permit, a meta-transaction, an off-chain order) authorizes an on-chain or
  backend action.
- The signature may lack a nonce, or its domain separator may omit the chain id or verifying contract.
- Signer recovery may accept a malleable signature or an unchecked zero address, or the signed fields may omit
  part of the action.

## Scope check

Test signature replay only against contracts and services you own or are authorized to assess, on testnets or a
local fork. Replaying a signature executes a real authorized action, so use test keys and deployments and never
replay a signature that moves real value or acts for someone else. If you can't name the authorization, stop.

## The loop

1. **Establish what the signature is bound to first.** Name the one action it authorizes, the chain and
   contract it is meant for, and how it is marked used: a nonce, a deadline, a consumed flag. This is the
   false-positive killer: a signature bound by a per-signer nonce, a domain separator carrying the chain id and
   verifying contract, signed fields covering every part of the action, and non-malleable recovery is used once,
   here, for what it says. Name the binding, then test each missing piece.

2. **Test nonce and single-use.** Take a valid signature and submit it a second time. Confirm a per-signer nonce
   or a consumed marker makes the replay revert. A permit or meta-transaction with no nonce, or one that is not
   marked used, executes again every time it is submitted.

3. **Test cross-chain and cross-contract replay.** Replay the signature on a different chain and against a
   different deployment of the contract. Confirm the domain separator binds the chain id and the verifying
   contract so the signature is valid in exactly one place. A domain missing either lets one signature act on
   every chain or every deployment.

4. **Test field completeness.** Compare the signed typed-data fields against everything the action depends on:
   amount, recipient, token, deadline, function selector. Submit the signature for a variant action that
   differs in an unsigned field. If the action changes but the signature still verifies, the omitted field is
   attacker-controlled under a valid signature.

5. **Test recovery soundness.** Probe signer recovery: submit a malleable variant of a signature (the
   complementary s-value), a zero or malformed signature, and confirm recovery rejects them and never treats a
   zero address as a valid signer. Malleable or zero-address recovery lets a signature be duplicated or forged
   past the check.

6. **Confirm and record.** Confirm on a test deployment by replaying a signature a second time, on another
   chain or contract, or reusing it for a different action, with test keys and without moving real value. Kill
   the lead if the signature is bound by a nonce, a chain-and-contract domain separator, complete signed fields,
   and non-malleable recovery. Record the reused or cross-context signature, the repeated or unintended
   authorized action, and the missing nonce or domain binding.

## Where signature trust leaks

- **No nonce.** A signature with no per-signer nonce or consumed marker replays every time it is submitted.
- **Domain separator missing chain id or contract.** A signature valid on one chain or deployment replays on
  another when the domain does not bind where it applies.
- **Incomplete signed fields.** An action that depends on a field the signature does not cover can be varied in
  that field under the same valid signature.
- **Malleable or zero-address recovery.** Accepting a malleable signature variant or treating a zero address as
  a valid signer lets a signature be duplicated or forged past verification.
- **No deadline.** A signature with no expiry stays valid forever, widening the window for replay and for use
  under conditions the signer no longer intends.

## Worked example (a confirm and a kill)

> **Confirm.** A meta-transaction relayer verifies an EIP-712 signature over the call data and forwards it, but
> the signed struct includes no nonce and the contract does not mark signatures used. On a fork, the same signed
> meta-transaction is submitted twice and executes both times, repeating a token transfer the signer authorized
> once. **Confirmed** signature replay via missing nonce, `high`, remediation = include a per-signer nonce in
> the signed struct, increment and check it on use so each signature executes once, and add a deadline.
>
> **Kill.** The signed struct includes a per-signer nonce that is checked and incremented on use, the domain
> separator binds the chain id and verifying contract, the typed data covers every field the action depends on
> (amount, recipient, deadline), and recovery rejects malleable and zero-address signatures. A second
> submission reverts on the nonce, a cross-chain replay fails the domain, a varied action fails verification,
> and a malleable variant is rejected. **Killed**, `kill_reason` = "signature bound by nonce, chain-and-contract
> domain, complete signed fields, and non-malleable recovery; it authorizes one action, once, here."

## Rationalizations to reject

- *"The signature is cryptographically valid."* → Validity is not authorization scope; a valid signature with no
  nonce or domain binding still replays and still acts in the wrong place.
- *"It uses EIP-712."* → EIP-712 only protects what the domain and typed data actually include; confirm the
  domain carries chain id and contract and the struct carries a nonce and every action field.
- *"Nonces are for transactions, not signatures."* → The signed message needs its own per-signer nonce or
  consumed marker; the transaction nonce does not stop a signature being submitted again.
- *"We check the deadline."* → A deadline bounds the window but does not prevent replay within it or across
  chains; you still need a nonce and a chain-and-contract domain.
- *"Recovery returns the signer."* → Confirm it rejects malleable variants and never returns or accepts the zero
  address; unchecked recovery is a forgery and duplication path.

## Executing this in practice

You need the signed struct's fields, the domain separator's contents (chain id and verifying contract), how the
signature is marked used (nonce or consumed flag), and how signer recovery handles malleability and the zero
address. On a fork or test service, replay a signature twice, on another chain and contract, and for a varied
action, and submit malleable and zero signatures. Reading the verification code shows the intended binding; a
signature that executes twice, elsewhere, or for a different action shows what the binding omits.

## Related

- `auditing-account-abstraction-and-paymaster-trust` - account validation verifies signatures over user
  operations; the nonce and domain binding here is exactly what that validation must enforce.
- `auditing-cross-chain-bridge-and-message-trust` - bridge attestations are signatures over cross-chain events;
  replay and domain binding decide whether one event authorizes one action.
- `hunting-wallet-drainer-and-dapp-approval-abuse` - a permit is a signature that grants spending; replayable or
  overbroad permits are how drainers move funds.
- `auditing-oauth-token-audience-and-scope-trust` - the same binding idea off-chain: a token, like a signature,
  must be bound to one audience and scope or it is replayable out of context.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the reused or cross-context signature, sink = the
  repeated or unintended authorized action, evidence = the missing nonce or domain binding.
