---
name: auditing-cross-chain-bridge-and-message-trust
description: >-
  Audit a cross-chain bridge or messaging protocol for misplaced trust in messages that cross chains: a
  destination contract that accepts a mint or release on a forged or unverified proof of a source-chain event, a
  message whose signer set or validator quorum can be spoofed or is too small, a message that can be replayed on
  the destination or across chains for a repeated withdrawal, and a lock-and-mint or burn-and-release accounting
  that a crafted message pushes out of balance. Covers token bridges, message-passing layers, and any protocol
  where an action on one chain is authorized by an event claimed to have happened on another. Use when a
  destination-chain action depends on trusting a source-chain event and that verification is the boundary. The
  forged or replayed cross-chain message is the source, the unauthorized mint, release, or state change is the
  sink, and the missing or spoofable source-event verification is the bug.
license: MIT
---

# Auditing cross-chain bridge and message trust: the destination must prove the source event, not assume it

A cross-chain bridge does one hard thing: it makes an action on the destination chain, minting a wrapped token,
releasing locked funds, running a message, conditional on an event that happened on the source chain. The
destination cannot see the source directly, so it trusts a proof or an attestation that the event occurred, and
every bridge hack is a failure of that trust. If the destination accepts a mint or release on a forged proof,
an unverified claim, or an attestation from a signer set an attacker can spoof or that is too small to be safe,
the attacker manufactures value from nothing. If a valid message can be replayed, on the destination twice, or
on another chain, one legitimate event authorizes many withdrawals. And the lock-and-mint or burn-and-release
accounting that is supposed to keep supply balanced can be pushed out of balance by a message that mints
without a matching lock or releases without a matching burn. The audit follows a message from source event to
destination action and checks that the destination genuinely verifies the event, binds the message so it
executes once, and keeps the accounting balanced. You audit this by crafting messages the destination should
reject and seeing whether it mints or releases.

## When to use

- A bridge or messaging layer authorizes a destination-chain action (mint, release, call) on the strength of a
  claimed source-chain event.
- The proof or attestation may be forgeable, or the signer set or validator quorum may be spoofable or too
  small.
- A cross-chain message may be replayable on the destination or across chains, or the lock/mint accounting may
  be pushable out of balance.

## Scope check

Test bridge and message trust only against protocols and infrastructure you own or are authorized to assess, on
testnets or a local fork. Forging messages and triggering mints or releases moves real value, so use test
deployments and never mint, release, or unbalance a live bridge or touch funds that are not yours. If you can't
name the authorization, stop.

## The loop

1. **Establish how the destination verifies the source event first.** Name what proof or attestation the
   destination requires (a light-client proof, a signer-set or validator quorum, a relayer signature), how many
   independent parties must agree, and how a message is bound so it executes once. This is the false-positive
   killer: a destination that verifies a sound proof of the source event, requires a quorum an attacker cannot
   reach, binds each message against replay, and keeps lock/mint accounting balanced is behaving correctly. Name
   the verification, then attack it.

2. **Test proof and attestation forgery.** Submit a destination action with a forged proof, an invalid or
   malformed attestation, or a proof of an event that did not happen, and confirm the destination rejects it. A
   destination that mints or releases on an unverified or forgeable proof is the core bridge bug.

3. **Probe the signer set or quorum.** Determine who attests to source events and how many must agree. Check
   whether the quorum is large and independent enough, whether the signer set can be spoofed or a threshold met
   by an attacker (compromised or too few keys), and whether the set can be changed by a message the attacker
   influences. A spoofable or too-small quorum means attacker-signed messages are accepted.

4. **Test replay on the destination and across chains.** Take a valid message and replay it on the destination
   a second time, and replay it on a different chain or entry point. Confirm each message is bound to its
   destination chain, contract, and a unique nonce so it executes exactly once, in one place. A replayable
   message turns one event into repeated withdrawals.

5. **Check lock/mint and burn/release accounting.** Trace whether every mint on the destination corresponds to
   a real lock on the source and every release to a real burn, and whether a crafted message can mint without a
   lock or release without a burn. Confirm the accounting cannot be pushed out of balance. Broken accounting is
   how a bridge ends up insolvent.

6. **Confirm and record.** Confirm on a test deployment by minting or releasing on a forged proof, an
   attacker-reachable quorum, or a replayed message, or by unbalancing the accounting, without touching a live
   bridge or real funds. Kill the lead if the destination verifies a sound proof, requires an unreachable
   quorum, binds every message against replay across chains and entry points, and keeps accounting balanced.
   Record the forged or replayed message, the unauthorized mint, release, or state change, and the missing or
   spoofable source-event verification.

## Where bridge trust leaks

- **Unverified or forgeable proofs.** A destination that accepts a mint or release without soundly proving the
  source event manufactures value from a forged claim.
- **A spoofable or too-small quorum.** A signer set an attacker can spoof, meet with too few keys, or change via
  an influenced message authorizes malicious withdrawals.
- **Replayable messages.** A message not bound to its destination chain, contract, and unique nonce executes
  more than once, or on another chain, for repeated effect.
- **Unbalanced lock/mint accounting.** A mint without a matching lock, or a release without a matching burn,
  pushes supply out of balance and toward insolvency.
- **Trusted relayer with no verification.** A relayer whose message is trusted on its say-so, rather than a
  verified proof, is a single point an attacker takes over.

## Worked example (a confirm and a kill)

> **Confirm.** A token bridge releases locked funds on the destination when a set of five relayer signatures
> attests to a burn on the source, but the release function checks only that five signatures are present, not
> that they come from the authorized set. On a test deployment, an attacker supplies five self-generated
> signatures and releases funds with no corresponding burn, draining the locked pool. **Confirmed** unauthorized
> release via unverified signer set, `critical`, remediation = verify each signature against the authorized
> validator set and require a genuine independent quorum, bind the release to a unique burn event, and reconcile
> release against a matching burn.
>
> **Kill.** The destination verifies a sound proof (or a genuine quorum of the authorized, independent validator
> set) of the source event, binds every message to its destination chain, contract, and a unique nonce so it
> executes once, and reconciles every mint against a real lock and every release against a real burn. A forged
> proof is rejected, a self-signed quorum fails, a replayed message reverts, and no message mints or releases
> without its counterpart. **Killed**, `kill_reason` = "destination verifies a sound source-event proof from an
> unreachable quorum, messages are replay-bound across chains and entry points, and lock/mint accounting stays
> balanced; no forged or replayed message mints or releases."

## Rationalizations to reject

- *"Relayers are trusted."* → A trusted relayer with no verifiable proof is a single point of compromise; the
  destination must verify the source event, not the relayer's word.
- *"It checks the signatures."* → Confirm it checks them against the authorized set and requires a real
  independent quorum; counting signatures without verifying the signers is the classic bridge bug.
- *"Messages have nonces."* → Verify the nonce is bound to the destination chain and contract too; a nonce alone
  does not stop replay on another chain or entry point.
- *"Supply is fixed by the contract."* → Trace lock-to-mint and burn-to-release end to end; a message that mints
  without a lock breaks the invariant regardless of the token's own cap.
- *"The bridge is audited."* → Bridge exploits repeatedly pass prior audits; verify the source-event proof,
  quorum, replay binding, and accounting yourself against the live deployment.

## Executing this in practice

You need how the destination verifies source events (proof or quorum), the authorized signer set and threshold,
how messages are bound against replay, and the lock/mint and burn/release accounting. On test deployments,
submit forged proofs, self-signed quora, and replayed messages, and attempt to mint or release without a
counterpart. Reading the verification and accounting contracts shows the intended trust; a mint or release on a
message the destination should have rejected shows whether it holds.

## Related

- `hunting-signature-replay-and-eip712-domain-trust` - the signature and domain binding that a bridge's
  attestations depend on; replayable signatures break both.
- `hunting-mev-and-transaction-ordering-exposure` - cross-chain messages have ordering and timing assumptions a
  sequencer or relayer can exploit, alongside the verification checked here.
- `auditing-account-abstraction-and-paymaster-trust` - another stack where separate parties validate and
  execute; the trust-split analysis is the same shape.
- `hunting-wallet-drainer-and-dapp-approval-abuse` - the user-facing side of the same ecosystem; bridge and
  approval abuse are two ways value leaves a wallet.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the forged or replayed cross-chain message, sink = the
  unauthorized mint, release, or state change, evidence = the missing or spoofable source-event verification.
