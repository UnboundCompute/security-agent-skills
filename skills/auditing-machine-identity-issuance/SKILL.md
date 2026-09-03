---
name: auditing-machine-identity-issuance
description: >-
  Audit how a platform issues machine and workload identities (certificate authorities, workload-identity
  federation, service-mesh identity, attestation-based credentialing) for trust misplaced in the thing asking
  for one: a credential issued on a weak or forgeable proof (a self-asserted name, an unvalidated label, a
  reachable metadata endpoint) so forging the proof yields a real identity, an issuing authority not
  constrained to the names it may mint, a federation trust configured so broadly (a wildcard subject, unpinned
  issuer, missing audience) that an outside principal can assume it, a certificate with an over-long lifetime or
  no revocation, and an issuance path with no binding to a verified workload. Use when a platform decides what
  proof earns a machine identity and that is the boundary. The forgeable issuance proof or
  over-broad trust is the source, the illegitimately issued machine identity is the sink, and the weak
  attestation, unconstrained issuer, or over-broad federation trust is the bug.
license: MIT
---

# Auditing machine identity issuance: the identity is only as trustworthy as the proof that earned it

Every machine identity, a service mesh certificate, a federated cloud credential, a signed workload token, is a
statement the platform makes on a workload's behalf: this is who this thing is. Downstream services trust that
statement and skip re-checking, which is the point of an identity system, so the entire chain of trust rests on
one decision made at issuance: what proof did the platform require before it minted the identity. If that proof
is weak or forgeable, the identity is legitimate and the holder is not. A certificate issued on a self-asserted
name, an unvalidated label, or a reachable metadata endpoint lets an attacker who forges the proof obtain a
real, trusted identity. An issuing authority whose scope is not constrained can mint identities for names it
should never speak for, so a compromise of one issuer forges any workload. A workload-identity federation trust
configured too broadly, a wildcard subject, an unpinned issuer, a missing audience, lets an outside principal
assume a workload identity that was meant for a specific one. A certificate or token with an over-long lifetime
or no revocation lets a compromised identity persist long after it should be dead. And an issuance path with no
binding to a verified workload hands an identity to any caller who asks. The audit follows issuance from the
proof presented to the identity granted, and checks that the proof is strong, the issuer is scoped, the
federation trust is narrow, and the identity is short-lived and revocable. You audit this by presenting a weak
or forged proof and seeing whether a legitimate identity comes back.

## When to use

- A platform issues machine or workload identities: mesh certificates, workload-identity federation credentials,
  signed workload tokens, or attestation-based credentials.
- Issuance may rely on a weak or forgeable proof, or the issuing authority may not be scoped to the names it may
  speak for.
- A federation trust may be over-broad (wildcard subject, unpinned issuer, missing audience), or issued
  identities may be over-long-lived or unrevocable.

## Scope check

Test machine-identity issuance only against platforms and identity systems you own or are authorized to assess,
with test workloads. Requesting and using issued identities exercises real trust, so use test issuers and
workloads and never obtain, forge, or use a machine identity outside your authorization. If you can't name the
authorization, stop.

## The loop

1. **Establish what proof earns each identity and what each issuer may speak for first.** Name, for each issuance
   path, the proof a workload must present, how that proof is verified, what names or scopes the issuing
   authority is allowed to mint, how narrow each federation trust is, and the lifetime and revocation of what is
   issued. This is the false-positive killer: an issuance path that requires a strong, non-forgeable attestation,
   a scoped issuer that can mint only its intended names, a federation trust pinned to a specific subject,
   issuer, and audience, and short-lived revocable identities is behaving correctly. Name the intended proof,
   then test each path.

2. **Test the issuance proof.** Present the weakest proof the path will accept, a self-asserted name, an
   unvalidated label, a value read from a reachable metadata endpoint, and confirm a legitimate identity is not
   issued on it. Try to forge or replay the proof another workload would present. An identity minted on a
   forgeable proof is a real identity in an attacker's hands.

3. **Test issuer scope.** Examine what names or scopes each issuing authority (CA, signer, token issuer) is
   constrained to mint, and confirm it cannot issue an identity for a name outside its authority. Request an
   identity for a name the issuer should never speak for. An unconstrained issuer means compromising or misusing
   one issuer forges any workload's identity.

4. **Test federation trust breadth.** For workload-identity federation, inspect the trust configuration: whether
   the subject is pinned or a wildcard, whether the issuer is pinned, and whether an audience is required and
   checked. Attempt to assume the workload identity as a different or outside principal. A wildcard subject,
   unpinned issuer, or missing audience lets a principal the trust was not meant for assume the identity.

5. **Test lifetime and revocation.** Determine the lifetime of issued identities and whether they can be revoked
   promptly. Use an identity past when it should be valid and after it should be revoked. An over-long lifetime
   or absent revocation lets a compromised machine identity keep authenticating long after it should be dead, so
   confirm identities are short-lived and revocable.

6. **Confirm and record.** Confirm with test workloads by obtaining a legitimate identity on a forged or weak
   proof, minting an identity for a name outside an issuer's scope, assuming a workload identity as an outside
   principal through a broad federation trust, or using an identity past its intended life, without touching real
   workloads. Kill the lead if issuance requires a strong non-forgeable proof, issuers are scoped, federation
   trusts are pinned to subject, issuer, and audience, and identities are short-lived and revocable. Record the
   forgeable proof or over-broad trust, the illegitimately issued identity, and the weak attestation,
   unconstrained issuer, or over-broad federation trust.

## Where issuance trust leaks

- **Forgeable issuance proof.** An identity minted on a self-asserted name, an unvalidated label, or a reachable
  metadata value lets an attacker who forges the proof obtain a legitimate identity.
- **Unconstrained issuer.** An issuing authority not scoped to the names it may mint can speak for workloads it
  should never represent, so one issuer's misuse forges any identity.
- **Over-broad federation trust.** A wildcard subject, unpinned issuer, or missing audience in a workload-identity
  federation trust lets an outside principal assume a workload identity meant for a specific one.
- **Over-long lifetime or no revocation.** A certificate or token issued for too long, or with no working
  revocation, lets a compromised identity keep authenticating long after it should be dead.
- **No binding to a verified workload.** An issuance path that does not bind the identity to a verified,
  attested workload hands a real identity to any caller who asks.

## Worked example (a confirm and a kill)

> **Confirm.** A cloud platform's workload-identity federation trust accepts tokens from an external issuer with
> the subject claim configured as a wildcard and no audience check, so any token that issuer mints is accepted as
> the workload. On a test project, an outside principal who can obtain a token from that issuer assumes the
> workload identity and calls the platform's APIs as it, because the trust pins neither the subject nor an
> audience. **Confirmed** workload-identity assumption via over-broad federation trust, `high`, remediation =
> pin the trust to the exact subject that represents the intended workload, require and verify a specific
> audience, and pin the issuer so only the intended external identity can assume the workload.
>
> **Kill.** Each issuance path requires a strong non-forgeable attestation of the workload; issuers are
> constrained to the exact names they may mint; every federation trust is pinned to a specific subject, a pinned
> issuer, and a required audience; and issued identities are short-lived and revocable. A forged or weak proof
> yields no identity, an issuer cannot mint outside its scope, an outside principal cannot assume a pinned
> workload trust, and an expired or revoked identity is rejected. **Killed**, `kill_reason` = "issuance requires
> a strong non-forgeable proof, issuers are scoped, federation trusts are pinned to subject, issuer, and
> audience, and identities are short-lived and revocable; no forged proof or broad trust yields a machine
> identity it should not."

## Rationalizations to reject

- *"The workload says who it is."* → A self-asserted name is not a proof; require a strong attestation the
  attacker cannot forge before minting an identity for a claimed workload.
- *"The issuer is trusted."* → Confirm it is scoped to the names it may mint; an unconstrained issuer, once
  misused or compromised, forges any workload's identity, so trust is not enough without scope.
- *"Federation is set up and working."* → Working for the intended workload is not the same as pinned against
  others; confirm the subject, issuer, and audience are all pinned so no outside principal can assume it.
- *"Certificates last a year for convenience."* → A long-lived identity with weak revocation is a long-lived
  compromise; issue short-lived, automatically renewed identities that can be revoked.
- *"Anything on the network can request one."* → An issuance path with no binding to a verified workload hands
  identities to any caller; bind each identity to an attested workload, not to reachability.

## Executing this in practice

You need each issuance path's required proof and how it is verified, what names each issuer may mint, the
subject-issuer-audience pinning of each federation trust, and the lifetime and revocation of issued identities.
With test workloads, present weak and forged proofs, request out-of-scope names, attempt to assume a workload
trust as an outside principal, and use identities past their intended life. Reading the issuance and trust
configuration shows the intended proof; a legitimate identity obtained on a forged proof or a broad trust shows
where the proof fails.

## Related

- `hunting-mutual-tls-and-service-identity-gaps` - the consuming side of mesh identity; issuance decides who gets
  a certificate, mTLS decides whether it is checked, and the two together bound service identity.
- `auditing-cicd-oidc-trust` - a specific federation-trust case; a CI pipeline assuming a cloud identity is
  workload-identity federation with the same subject, issuer, and audience pinning questions.
- `auditing-service-account-key-lifecycle` - the credential an issued identity often replaces; short-lived
  attested identities are the alternative to long-lived downloaded keys.
- `hunting-non-human-identity-and-secret-reachability` - once an identity is issued, its reachability and reuse
  are that skill's subject; issuance is where the identity's trust begins.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the forgeable issuance proof or over-broad trust, sink =
  the illegitimately issued machine identity, evidence = the weak attestation, unconstrained issuer, or
  over-broad federation trust.
