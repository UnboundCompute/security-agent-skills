---
name: hunting-kerberos-and-ad-delegation-abuse
description: >-
  Hunt for Active Directory Kerberos delegation configurations that let one identity act as another: a service
  account with unconstrained delegation that can impersonate any user who authenticates to it and reuse their
  ticket anywhere, constrained delegation configured so a service can request tickets for higher-privileged
  targets, resource-based delegation an attacker can set on an object they control to relay into it, a service
  account with a weak password exposed to Kerberoasting through its service principal name, and an account not
  requiring pre-authentication that is roastable offline. Use when a service is allowed to reuse or request a
  user's Kerberos identity and the scope of that delegation is the boundary. The delegation right or roastable credential is the source, the impersonated higher-privileged
  identity is the sink, and the overbroad delegation or crackable service account is the bug.
license: MIT
---

# Hunting Kerberos and AD delegation abuse: delegation lets a service become the user, so scope it

Kerberos delegation exists so a front-end service can act as the user against a back-end service, a web app
reaching a database as the logged-in user, for instance. That is useful and also a standing impersonation
right, and the danger is entirely in its scope. Unconstrained delegation is the worst case: a service with it
receives a forwardable ticket for every user who authenticates, caches it, and can replay that ticket against
anything, so compromising one such service turns every user who touched it, up to and including a domain admin,
into an identity the attacker can wear. Constrained delegation narrows the targets, but if the allowed targets
include a higher-privileged service, or protocol transition lets the service request tickets for users who
never authenticated to it, the constraint is a ramp. Resource-based constrained delegation moves the setting
onto the target object, so an attacker who can write to an object they control can authorize their own
delegation into it. And the credential side is just as live: a service account with a service principal name
can be Kerberoasted, its ticket requested and the encrypted portion cracked offline to recover a weak password,
and an account that does not require pre-authentication is roastable the same way without even authenticating.
The hunt inventories delegation rights and roastable accounts and traces each to the highest identity it can
reach. You hunt this by finding who may delegate for whom and who can be cracked, then following the chain up.

## When to use

- An Active Directory environment uses Kerberos with service accounts, service principal names, and one or more
  delegation models configured.
- A service may have unconstrained delegation, constrained delegation to a sensitive target, or
  attacker-settable resource-based delegation.
- A service account may be Kerberoastable through its SPN, or an account may not require Kerberos
  pre-authentication.

## Scope check

Test Kerberos and delegation only in directories you own or are explicitly authorized to assess, in a lab or a
scoped engagement. Requesting service tickets, cracking them, and exercising delegation impersonate real
identities and touch domain security, so use test forests and authorized engagements and never impersonate,
crack, or delegate for identities outside your authorization. If you can't name the authorization, stop.

## The loop

1. **Establish the intended delegation and which accounts are sensitive first.** Inventory every account with a
   delegation right, the model it uses, the exact targets it is meant to reach, and which identities are
   high-privilege and must never be impersonable. This is the false-positive killer: an environment with no
   unconstrained delegation, constrained delegation scoped to only the intended non-sensitive targets,
   resource-based delegation writable only by trusted admins, sensitive accounts marked not-delegatable, and
   strong service-account passwords is behaving correctly. Name the intended scope, then hunt the deviations.

2. **Find unconstrained delegation.** Enumerate accounts configured for unconstrained delegation. Each is a
   ticket-collector: any user who authenticates to it, including highly privileged ones, hands it a reusable
   ticket. Trace which high-privilege identities could authenticate to each such service, because those are the
   identities an attacker who owns the service can impersonate anywhere.

3. **Examine constrained delegation and protocol transition.** For each constrained-delegation account, list its
   allowed targets and check whether any is a higher-privileged service, and whether protocol transition lets it
   request tickets for users who never authenticated to it. A target that reaches sensitive services, or
   transition that fabricates a user context, turns a narrow delegation into an escalation.

4. **Check resource-based constrained delegation writability.** For sensitive target objects, check who can
   write the attribute that authorizes resource-based delegation into them. If an attacker-controllable account
   can set it, they can authorize their own service to delegate into the target. Confirm only trusted admins can
   write delegation authorization on sensitive objects.

5. **Find roastable service accounts and pre-auth-disabled accounts.** Enumerate accounts with a service
   principal name (Kerberoastable) and accounts that do not require pre-authentication (roastable offline).
   Assess password strength for the roastable set: a weak or reused service-account password recovered offline
   yields that account's privileges directly. Confirm service accounts have strong, managed passwords and
   pre-authentication is required.

6. **Confirm and record.** Confirm in a lab or authorized engagement by impersonating a higher-privileged
   identity through a delegation right, or recovering a service-account password by cracking a requested ticket,
   and using it to reach access the account should not grant, without touching identities outside scope. Kill
   the lead if there is no unconstrained delegation, constrained targets are all non-sensitive, resource-based
   delegation is admin-only writable, service accounts have strong passwords, and pre-authentication is required.
   Record the delegation right or roastable credential, the impersonated higher-privileged identity, and the
   overbroad delegation or crackable service account.

## Where delegation trust leaks

- **Unconstrained delegation.** A service with it caches a reusable ticket for every user who authenticates,
  including privileged ones, and can replay it anywhere.
- **Constrained delegation to a sensitive target.** Allowed targets that include a higher-privileged service, or
  protocol transition that fabricates a user context, turn a narrow delegation into escalation.
- **Attacker-settable resource-based delegation.** A delegation-authorization attribute writable by an
  attacker-controlled account lets them authorize their own delegation into the target.
- **Kerberoastable weak service accounts.** A service principal name plus a weak or reused password lets an
  attacker request the ticket and crack the account's password offline.
- **No pre-authentication required.** An account that does not require Kerberos pre-authentication can be roasted
  offline without authenticating at all.

## Worked example (a confirm and a kill)

> **Confirm.** A web application's service account is configured for unconstrained delegation, and domain
> administrators occasionally browse the application. In a lab, after compromising the application host, a
> forwardable ticket for a domain admin who authenticated is captured from the service and replayed against a
> domain controller, giving domain-admin access, because unconstrained delegation cached a reusable privileged
> ticket. **Confirmed** privilege escalation via unconstrained delegation, `critical`, remediation = remove
> unconstrained delegation, use resource-based constrained delegation scoped to the specific back-end target,
> and mark privileged accounts as sensitive and not-delegatable.
>
> **Kill.** No account has unconstrained delegation; constrained delegation is scoped to specific non-sensitive
> back-end targets with no protocol transition into privileged services; resource-based delegation attributes on
> sensitive objects are writable only by trusted admins; service accounts use strong managed passwords so a
> requested ticket does not crack; and pre-authentication is required everywhere. A captured ticket reaches only
> its intended target and a roasted ticket does not yield a password. **Killed**, `kill_reason` = "no
> unconstrained delegation, constrained targets are non-sensitive, resource-based delegation is admin-only, and
> service accounts are strong with pre-auth required; no delegation right or roast reaches a higher-privileged
> identity."

## Rationalizations to reject

- *"Delegation is required for the app to work."* → Required is not unconstrained; use resource-based or
  constrained delegation scoped to the exact back-end target, never a right that reuses every user's ticket
  anywhere.
- *"The service account is internal."* → An internal service account with a service principal name is
  Kerberoastable by any authenticated user; give it a strong managed password or its ticket cracks offline.
- *"Only admins can configure delegation."* → Confirm who can write the resource-based delegation attribute on
  each sensitive object; if an attacker-controllable account can, they configure their own delegation.
- *"Privileged accounts know not to log in there."* → Unconstrained delegation captures a ticket from anyone who
  authenticates; mark sensitive accounts not-delegatable rather than relying on where they log in.
- *"Pre-authentication is on by default."* → Confirm it per account; a single account with pre-auth disabled is
  roastable offline without authenticating, so enumerate the exceptions.

## Executing this in practice

You need the inventory of delegation rights (which model, which targets), who can write resource-based
delegation attributes on sensitive objects, the set of accounts with service principal names, and which
accounts do not require pre-authentication, plus service-account password strength. In a lab or authorized
engagement, trace each delegation right to the highest identity it can reach and test whether roastable accounts
crack. Reading the directory configuration shows the intended scope; a captured privileged ticket or a cracked
service password shows what the scope fails to contain.

## Related

- `hunting-iam-privilege-escalation-paths` - the cloud-and-directory escalation-path view; a delegation right or
  roasted account is one edge in the broader path to a high-privilege identity.
- `mapping-service-account-impersonation-chains` - delegation is one impersonation mechanism; that skill maps the
  chains service accounts form across a system.
- `hunting-non-human-identity-and-secret-reachability` - a Kerberoastable service account is a non-human identity
  whose secret is reachable and crackable; the two hunts meet on service-account exposure.
- `hunting-mutual-tls-and-service-identity-gaps` - the certificate-based analogue of service identity; both ask
  whether a service can act as an identity it should not.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the delegation right or roastable credential, sink =
  the impersonated higher-privileged identity, evidence = the overbroad delegation or crackable service account.
