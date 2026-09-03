---
name: hunting-tenant-onboarding-and-discovery-abuse
description: >-
  Hunt for multi-tenant SaaS onboarding and organization-discovery flows that let an attacker join, claim, or
  enumerate a tenant they should not: a self-service signup that auto-joins a new user to an existing
  organization by email domain so anyone with a matching address lands inside it, a domain-claim step that can be
  satisfied without proving ownership so an attacker claims a domain and its users, an invitation whose token is
  guessable, reusable, or not bound to the invited address, a tenant-discovery endpoint that reveals which
  organizations or domains exist, and a first-admin race where claiming an unclaimed org grants admin over users
  who later join. Use when joining or claiming a tenant, or learning that one exists, is the boundary between an
  outsider and an organization's data. The onboarding or discovery request is the source, the unauthorized tenant
  membership, claim, or enumeration is the sink, and the unverified domain claim or unbound invitation is the
  bug.
license: MIT
---

# Hunting tenant onboarding and discovery abuse: how you get into an organization decides who is inside it

In multi-tenant SaaS the organization is the trust boundary, and onboarding is how someone gets put inside it,
so the onboarding flow decides membership as surely as any access-control check. The trouble is that onboarding
is built for growth, self-service signup, easy domain-based joining, frictionless invitations, and growth-shaped
defaults are exactly where an attacker walks in. A signup that auto-joins a new user to an existing organization
because their email domain matches lets anyone who can get an address at that domain, or spoof one, land inside
the tenant with whatever a member sees. A domain-claim step that can be satisfied without truly proving
ownership lets an attacker claim a domain and, with it, the users under it. An invitation whose token is
guessable, reusable, or not bound to the address it was sent to lets an attacker redeem someone else's invite. A
tenant-discovery endpoint that answers which organizations exist or which domains are registered hands an
attacker the map. And a first-admin or org-creation race, where claiming an as-yet-unclaimed organization makes
you its admin, grants authority over everyone who later joins. The hunt walks every way in, and every way to
learn a tenant exists, and checks that each proves what it claims. You hunt this by trying to join, claim, or
enumerate a tenant you have no right to.

## When to use

- A multi-tenant application lets users sign up, create or claim organizations, verify domains, or accept
  invitations to join a tenant.
- Signup may auto-join by email domain, domain verification may not prove ownership, or invitations may be
  guessable, reusable, or unbound to the invited address.
- A tenant-discovery endpoint may reveal which organizations or domains exist, or claiming an unclaimed org may
  grant admin.

## Scope check

Test tenant onboarding and discovery only against applications and organizations you own or are authorized to
assess, on test tenants. Joining, claiming, and enumerating tenants touches real organization membership and
data, so use test organizations and never join, claim, or enumerate a tenant that is not yours. If you can't
name the authorization, stop.

## The loop

1. **Establish how membership and claims are meant to be proven first.** Name what each onboarding path is
   supposed to require: what lets a user join an existing organization, what proves domain ownership, how an
   invitation is bound to its recipient, and how organization creation assigns the first admin. This is the
   false-positive killer: a flow that joins users only on proven membership, verifies domain ownership before
   trusting a domain, binds each invitation to one address and one use, and assigns org admin deliberately is
   behaving correctly. Name the intended proof, then test each path.

2. **Test email-domain auto-join.** Sign up with an address at a target organization's domain, a lookalike
   domain, a subdomain, and a plus-addressed or otherwise crafted variant, and check whether the signup
   auto-joins you to the existing organization. An organization that admits anyone with a matching email domain
   trusts an address as membership, so anyone who obtains or spoofs one lands inside.

3. **Test domain verification.** Exercise the domain-claim flow and confirm it requires a real ownership proof (a
   DNS record, a served file, an email to a controlled administrative address) that an outsider cannot satisfy.
   Try to complete a claim without controlling the domain. A domain-claim step satisfied without proving
   ownership lets an attacker claim a domain and the users tied to it.

4. **Test invitation binding.** Examine invitation tokens and links: whether the token is guessable or
   sequential, whether it can be redeemed more than once, and whether it is bound to the invited address so only
   that recipient can accept. Try to redeem an invite from a different account or a guessed token. A guessable,
   reusable, or unbound invitation lets an attacker join as someone else's invitee.

5. **Test discovery and org-claim races.** Probe any endpoint that reveals whether an organization or domain
   exists (signup checks, SSO-discovery lookups, error-message differences) and confirm it does not enumerate
   tenants. Separately, check whether claiming or creating an organization for an unclaimed domain or name grants
   admin over users who later join. An enumeration endpoint hands over the map, and a first-claim-wins org
   creation grants authority over future members.

6. **Confirm and record.** Confirm on test tenants by auto-joining an organization through a matching or spoofed
   domain, completing a domain claim without ownership, redeeming an invitation bound to someone else, or
   claiming an unclaimed org to gain admin, without touching real organizations. Kill the lead if joining
   requires proven membership, domain claims require ownership proof, invitations are bound and single-use,
   discovery does not enumerate, and org admin is assigned deliberately. Record the onboarding or discovery
   request, the unauthorized membership, claim, or enumeration, and the unverified domain claim or unbound
   invitation.

## Where onboarding and discovery leak

- **Email-domain auto-join.** Admitting anyone whose email domain matches trusts an address as membership, so a
  matching or spoofed address lands an outsider inside the organization.
- **Unproven domain claim.** A domain-verification step satisfied without proving ownership lets an attacker
  claim a domain and the users associated with it.
- **Guessable or unbound invitations.** An invitation token that is guessable, reusable, or not bound to the
  invited address lets an attacker redeem someone else's invite and join as them.
- **Tenant enumeration.** A discovery endpoint or error difference that reveals which organizations or domains
  exist hands an attacker the map of whom to target.
- **First-claim-wins org creation.** Claiming an unclaimed organization or domain to become its admin grants
  authority over every user who later joins it.

## Worked example (a confirm and a kill)

> **Confirm.** A SaaS application auto-joins any new signup to an existing organization when the signup email's
> domain matches the organization's domain, and its domain-claim flow marks a domain verified after an email is
> sent to an address at that domain with no ownership proof. On a test tenant, signing up with an address at the
> target domain lands the attacker inside the organization with member access, and the domain can be claimed
> without controlling it. **Confirmed** unauthorized tenant join and domain claim via email-domain trust and
> unproven verification, `high`, remediation = require an explicit invitation or admin approval to join an
> existing organization rather than trusting the email domain, and verify domain ownership with a control the
> outsider cannot satisfy (a DNS record or served token) before trusting a domain.
>
> **Kill.** Joining an existing organization requires an explicit, address-bound, single-use invitation or admin
> approval; domain claims require a DNS or served-file ownership proof; invitation tokens are unguessable, bound
> to one address, and consumed on first use; discovery endpoints and errors do not reveal whether an organization
> or domain exists; and organization admin is assigned deliberately, not by first claim. A matching-domain signup
> does not auto-join, an unproven domain claim fails, and a foreign invite cannot be redeemed. **Killed**,
> `kill_reason` = "joining needs proven membership, domain claims need ownership proof, invitations are bound and
> single-use, discovery does not enumerate, and admin is assigned deliberately; no onboarding or discovery
> request joins, claims, or enumerates a tenant it should not."

## Rationalizations to reject

- *"Same email domain means same company."* → An email domain is obtainable and spoofable and shared by
  unrelated people (contractors, subsidiaries, webmail); require an invitation or approval, not a domain match,
  to grant membership.
- *"We send a verification email to claim the domain."* → An email to the domain proves you can receive mail
  there, not that you own the domain; require a DNS record or served token an outsider cannot place.
- *"The invite link is secret."* → A guessable, reusable, or unbound invite is not secret; make tokens
  unguessable, single-use, and bound to the invited address.
- *"Discovery is just a convenience."* → An endpoint that confirms an organization or domain exists is
  reconnaissance; return uniform responses so a tenant cannot be enumerated.
- *"Whoever creates the org runs it."* → If anyone can claim an unclaimed organization or domain, an attacker
  claims one ahead of the real owner and holds admin over everyone who joins; assign admin through a verified
  path.

## Executing this in practice

You need every way to join or claim a tenant (domain auto-join, invitation, org creation), what each requires as
proof, how domain ownership is verified, how invitations are bound, and whether any endpoint reveals that a
tenant exists. On test tenants, try to join through a matching or spoofed domain, claim a domain you do not own,
redeem a foreign or guessed invite, and enumerate organizations. Reading the onboarding and discovery handlers
shows the intended proof; an unauthorized membership, claim, or enumeration shows where the proof is missing.

## Related

- `auditing-multi-tenant-isolation` - the runtime tenant boundary; onboarding decides who is inside a tenant,
  isolation decides what a member can reach, and the two together bound tenant data.
- `auditing-scim-provisioning-trust` - the provisioned way into a tenant; SCIM and self-service onboarding are
  two doors into the same organization membership.
- `auditing-jit-provisioning-and-role-mapping` - a federated first login is another onboarding path; domain and
  claim trust recur there in the assertion.
- `auditing-account-recovery-and-reset-trust` - a claim-without-proof in onboarding is the same failure as a
  recovery-without-proof; both grant access on an unverified assertion of identity.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the onboarding or discovery request, sink = the
  unauthorized tenant membership, claim, or enumeration, evidence = the unverified domain claim or unbound
  invitation.
