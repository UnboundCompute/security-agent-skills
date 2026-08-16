---
name: hunting-non-human-identity-and-secret-reachability
description: >-
  Hunt machine credentials that are live, over-privileged, and actually reachable,
  not just present. Covers non-human identities and secrets across code, configuration,
  and infrastructure definitions: API keys, service-account credentials, and long-lived
  tokens. Separates a secret that merely exists from one an attacker can reach and use,
  and adjudicates each by whether it is still valid, how much it grants, and whether an
  untrusted path leads to it. Use when reviewing secret exposure, machine identities, or
  the blast radius of a leaked credential, and when a scanner reports many secrets and
  you need to know which ones matter. A reachable, live, over-privileged credential is
  the finding.
license: MIT
---

# Hunting non-human identity and secret reachability: existence is not the bug, reach is

A secret scanner tells you a credential is present. That is a lead, not a finding.
The questions that decide severity are different: is this credential still valid, how
much does it grant, and can an attacker actually reach it? A dead key in old history
is noise. A live, broadly scoped token that an untrusted path leads to is an incident.
Machine identities now vastly outnumber human ones, so the volume is overwhelming;
the method that matters is separating reachable and powerful from merely present.

## When to use

- You are reviewing secret exposure or the machine identities in a system.
- A scanner reports many secrets and you need to know which are exploitable.
- You want the blast radius of a leaked credential, not just its location.

## Scope check

Inventory and validate credentials in systems you own or are authorized to test, and
test validity only against your own accounts. If you can't name the authorization, stop.

## The loop

1. **Inventory the machine identities and secrets.** Enumerate credentials across
   source, configuration, infrastructure definitions, build output, and runtime
   surfaces: keys, service-account credentials, and long-lived tokens. Record where
   each lives and which identity it belongs to. This is the candidate set, not the
   finding set.

2. **Determine reachability.** For each credential, ask who can reach where it sits.
   A secret in a public location, in client-delivered code, in a log, or on a surface
   an untrusted request touches is reachable. One held only in a restricted store that
   no untrusted path leads to is not. Reachability is the first filter that turns a
   lead into a candidate worth validating.

3. **Establish whether it is live.** A credential matters only if it still works.
   Confirm validity against your own account with a benign call, or from provenance
   (rotation records, expiry) when a live check is out of scope. A revoked or expired
   secret is a documented kill, however exposed it was.

4. **Measure the privilege and blast radius.** For a live, reachable credential,
   determine exactly what it grants: which resources, which actions, and whether it is
   scoped or broad. A read-only token to one bucket and an administrative
   service-account credential are not the same finding even if both leaked.

5. **Trace the exploit path.** Connect an untrusted starting point to the credential
   and then to what the credential unlocks. A key in front-end code reached by any
   visitor, a token in a log an attacker can read, a service-account credential a
   compromised workload can load: name the actor, the reach, and the resulting access.

6. **Confirm and record.** A finding is a credential that is reachable from an
   untrusted actor, live, and grants more than it should; confirm the access it yields
   in an authorized account. Kill the lead if the credential is dead, unreachable, or
   scoped so tightly that reaching it grants nothing. Record the location, the identity,
   the validity, the privilege, and the reach.

## Where machine-identity risk hides

- **Existence is not exposure.** The scanner's hit count is a lead list; reach and
  validity decide which are real.
- **Blast radius is the severity.** The same leak is trivial for a scoped read token
  and critical for a broad administrative identity. Measure the grant.
- **Long-lived is the aggravator.** A credential that never rotates keeps a stale leak
  exploitable indefinitely; short lifetimes shrink the window.
- **Machine identities hide in the plumbing.** Build artifacts, container layers, logs,
  and infrastructure definitions carry credentials that human-focused reviews skip.

## Worked example (a confirm and a kill)

> **Confirm.** A service-account credential is committed in an infrastructure
> definition in a repository that a build publishes into a client-delivered bundle.
> Any visitor can read it, it is still valid, and it grants write access across the
> project's storage. Loading it from an authorized test reaches and writes that
> storage. **Confirmed** reachable live over-privileged credential, `critical`,
> remediation = revoke and rotate now, remove from source and history, scope the
> identity to least privilege, and keep credentials out of client-delivered output.
>
> **Kill.** A scanner flags an access key in old commit history. Validating it against
> the owning account shows it was revoked long ago, no current build ships it, and the
> identity it belonged to no longer exists. Exposed, but inert. **Killed**,
> `kill_reason` = "credential revoked and identity removed; not live, nothing to reach."

## Rationalizations to reject

- *"The scanner found forty secrets."* → It found forty leads. How many are live and
  reachable? That is the finding count.
- *"It is only in a log."* → Who can read the log? A reachable log is an exposure path,
  not a safe place.
- *"That key is low-privilege."* → Then say what it grants and confirm it. Blast radius
  is the claim; measure it rather than assume it.
- *"It was rotated at some point."* → Prove it is dead now. A credential you cannot
  confirm as revoked is treated as live.

## Executing this in practice

You need to enumerate credentials across code, configuration, build output, and
runtime surfaces, a way to decide which sit where an untrusted actor can reach them,
authorized validation to tell live from dead, and the identity's effective permissions
to size the blast radius. A view that traces an untrusted surface to a credential and
the credential to what it grants turns a scanner's flat list into ranked, adjudicated
findings; the authorized validation is the confirmation.

## Related

- `hunting-iam-privilege-escalation-paths` - what a reachable machine credential lets
  an attacker do once it lands them on a principal.
- `auditing-cicd-oidc-trust` - pipelines are a common place credentials and tokens are
  exposed to untrusted input.
- `finding-crypto-misuse` - key handling failures in code, complementary to whether a
  key is reachable and live.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the untrusted surface that
  reaches the credential, sink = the resources and actions the live credential grants.
