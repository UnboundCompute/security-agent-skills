---
name: hunting-ntlm-coercion-and-relay
description: >-
  Hunt authentication coercion and relay on a Windows network, where an attacker induces a privileged
  machine or account to authenticate to a host it controls and forwards that authentication to a service that
  does not bind or verify the channel, authenticating as the coerced identity because signing is not enforced
  and channel binding is absent. Use when services accept network authentication without mandatory signing or
  channel binding and a privileged host can be induced to authenticate outbound. Covers coercion triggers,
  relay to directory, certificate, and file services, missing signing, and absent channel binding. The
  coerced outbound authentication is the source, the relaying service accepting it is the sink, and acting as
  the coerced privileged identity is the bug.
license: MIT
---

# Hunting NTLM coercion and relay: when a forced login lands on the wrong service

Windows authentication was designed to be forwarded: a client proves who it is to a server, and nothing in a
challenge-response exchange inherently ties that proof to the service the client meant to reach. An attacker
exploits both halves. First coercion: a feature that makes a machine or account connect outbound, an automatic
connection to a supplied path, a print or backup trigger, or any function that authenticates to an
attacker-named host. Second relay: the captured authentication is forwarded to a different service, and if
that service does not require signing or verify channel binding, it accepts the forwarded proof and treats the
attacker as the coerced identity. A relayed domain controller or privileged host becomes control of that
service. The bug is a service that accepts network authentication without binding it to the channel, reachable
by an identity an attacker can coerce. You hunt these by pairing coercion triggers with services that fail to
enforce signing or channel binding.

## When to use

- Services accept network authentication without mandatory signing or channel binding on a network you assess.
- A privileged machine or account can be induced to authenticate outbound to an attacker-named host.
- A directory, certificate, or file service is reachable by a coercible identity without channel protection.

## Scope check

Test coercion and relay only on networks you own or are authorized to assess, on non-production or a lab
domain, coercing only hosts you are permitted to and relaying only to services in scope, never acting as a
real privileged identity beyond a benign proof. A confirmed relay authenticates as a privileged account, so
keep every probe within scope. If you can't name the authorization, stop.

## The loop

1. **Establish whether the target service enforces signing and channel binding first.** For each service that
   accepts network authentication, determine whether it requires signing and verifies channel binding, so a
   relayed authentication is rejected, or whether it accepts forwarded proof unbound to the channel. This is
   the false-positive killer: a service that mandates signing or binds the channel cannot be relayed to, so a
   coercion trigger alone is not a finding. Name the service's enforcement before pairing it with a trigger.

2. **Enumerate coercion triggers.** Identify the functions that make a privileged machine or account
   authenticate outbound to a host the tester names: automatic connections to a supplied path, print,
   backup, or maintenance triggers, and any feature that resolves and authenticates to an operator-supplied
   location. Note which privileged identities each trigger causes to authenticate.

3. **Enumerate relay targets and their enforcement.** List the services reachable on the network that accept
   network authentication, and for each read whether signing is required and channel binding is verified.
   Directory, certificate enrollment, and file services are common targets; a target that does not bind the
   channel is a candidate sink.

4. **Pair a trigger with an unbound target.** A finding is a coercion trigger that authenticates a privileged
   identity paired with a reachable service that accepts the relayed authentication unbound. Confirm the
   coerced identity has meaningful privilege at the target, since relaying an unprivileged account gains
   nothing there.

5. **Determine the impact at the target.** Relaying to a directory service can grant object writes as the
   coerced account; relaying to a certificate enrollment interface can obtain a certificate for the coerced
   identity; relaying to a file service can read or write shares as that identity. Name what the coerced
   identity is authorized to do at the target.

6. **Confirm and record.** Confirm by coercing a permitted privileged host to authenticate and relaying it to
   an in-scope service that accepts it, observing a benign privileged action on an isolated lab domain. Kill
   the lead if the target enforces signing or verifies channel binding, if no trigger coerces a privileged
   identity, or if the coerced identity has no privilege at the target. Record the trigger, the coerced
   identity, the target service, its enforcement gap, and the benign proof, or set a `kill_reason`.

## Where coercion and relay leaks

- **The target's enforcement is the finding.** A service that mandates signing and verifies channel binding
  rejects a relay; the bug is a reachable service that accepts forwarded authentication unbound to the channel.
- **Coercion supplies the identity.** A trigger that makes a domain controller or a privileged host
  authenticate outbound hands the attacker that identity's authentication to forward.
- **Directory and certificate services are high-value targets.** Relaying a machine or privileged account to
  a directory write or a certificate enrollment yields object control or a durable credential for that
  identity.
- **File services leak as the coerced user.** A relay to a file service reads and writes shares as the coerced
  identity, reaching data the attacker's own account cannot.
- **Signing off by default is the recurring gap.** Services left without mandatory signing or channel binding,
  often for compatibility, are exactly the relay targets; the default, not a special weakness, is the hole.

## Worked example (a confirm and a kill)

> **Confirm.** A coercion trigger causes a privileged host to authenticate outbound to a tester-controlled
> host, and a reachable certificate enrollment interface accepts the relayed authentication without channel
> binding. Relaying it obtains a certificate for the coerced privileged identity on an isolated lab domain,
> demonstrated by a benign authentication with that certificate. **Confirmed** privilege escalation through
> coercion and relay to certificate enrollment, `high`, remediation = enforce signing and channel binding on
> the enrollment and directory services, disable the coercion trigger or restrict outbound authentication, and
> require channel-bound authentication network-wide.
>
> **Kill.** Every reachable service requires signing and verifies channel binding, so a relayed authentication
> is rejected, and outbound authentication from privileged hosts is restricted to known destinations. A
> coerced authentication cannot be forwarded to any service that will accept it. **Killed**, `kill_reason` =
> "all relay targets enforce signing and channel binding and reject forwarded authentication; coercion yields
> a proof that no reachable service will accept unbound."

## Rationalizations to reject

- *"We use a modern authentication protocol."* -> The relay question is whether the target binds the
  authentication to the channel and enforces signing, not which protocol; an unbound modern exchange relays
  too.
- *"Coercion requires a special trigger."* -> Many ordinary features authenticate to an operator-supplied
  path; enumerate them rather than assuming coercion needs an exotic primitive.
- *"Only unprivileged users would be relayed."* -> Coercion targets machines and privileged accounts
  specifically; confirm which identity the trigger authenticates, since that is what gets relayed.
- *"Signing is on somewhere."* -> Enforcement must hold at the specific target service; signing on one service
  does not protect another that accepts unbound authentication.
- *"The certificate service is internal."* -> Internal reachability is exactly the relay path; if a coercible
  identity can reach the enrollment interface unbound, internal placement does not protect it.

## Executing this in practice

You need every service that accepts network authentication with whether it enforces signing and channel
binding, the coercion triggers that make privileged identities authenticate outbound, and the privilege each
coerced identity holds at each reachable target. Pair a trigger that coerces a privileged identity with a
reachable target that accepts unbound authentication. Reading each target's enforcement settles most leads;
coercing a permitted host and relaying it to an in-scope service for a benign privileged action on an isolated
lab domain settles the rest.

## Related

- `hunting-adcs-certificate-template-abuse` - relaying a coerced authentication to a certificate enrollment
  interface obtains a certificate for the coerced identity, joining relay to that template escalation.
- `hunting-kerberos-and-ad-delegation-abuse` - a relayed or coerced authentication feeds the ticket-based
  escalation that skill pursues once the identity is captured.
- `auditing-windows-named-pipe-and-rpc-exposure` - the interfaces a coercion trigger abuses to induce outbound
  authentication are the local endpoints that skill audits.
- `auditing-tls-and-certificate-validation` - channel binding ties authentication to the transport, the
  validation control that skill audits from the channel side.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the coerced outbound authentication, sink = the
  relaying service accepting it unbound, evidence = a benign privileged action as the coerced identity on an
  isolated lab domain.
