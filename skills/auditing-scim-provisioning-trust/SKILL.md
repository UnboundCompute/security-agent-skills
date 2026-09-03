---
name: auditing-scim-provisioning-trust
description: >-
  Audit a SCIM 2.0 provisioning endpoint for trust misplaced in the identity provider that drives it: a
  provisioning API authenticated by a weak, shared, or long-lived bearer token anyone holding it can use to
  create users and grant groups, a handler that lets one tenant's client create or modify users in another
  tenant, a group or role pushed over SCIM that maps to more privilege than the source attribute should grant, a
  deprovisioning path that fails to disable a departed user so access lingers, and attribute updates (email,
  external id, admin flag) trusted without validation so an attacker reassigns an account. Use when an external
  identity source creates, updates, and deletes accounts over SCIM and the service provider's trust in that
  source is the boundary. The crafted or replayed SCIM request is the source, the unauthorized account, group,
  or lingering access is the sink, and the weak provisioning auth or unvalidated attribute mapping is the bug.
license: MIT
---

# Auditing SCIM provisioning trust: the provisioning token creates admins, so guard it like one

SCIM lets an identity provider create, update, and delete accounts in a service provider automatically, and
that means the SCIM endpoint is an account-and-group factory reachable over the network. Whoever can call it
with a valid credential can mint users, add them to privileged groups, and change who an existing account
belongs to, so the provisioning credential is effectively an admin credential and the trust in the calling
identity provider is the whole boundary. The failures follow. A provisioning endpoint authenticated by a weak,
shared, or never-expiring bearer token hands account creation to anyone who obtains it. A handler that does not
scope requests to the calling tenant lets one customer's provisioning client touch another customer's users. A
group or role pushed over SCIM that maps to more privilege than the source attribute warrants is a quiet
escalation. A deprovisioning event that does not actually disable the account leaves a departed user with
standing access. And attribute updates, email, external id, an admin flag, trusted without validation let an
attacker reassign or elevate an account. The audit treats the SCIM endpoint as privileged, checks its
authentication and tenant scoping, and follows each lifecycle event to what it actually grants or revokes. You
audit this by calling the SCIM API as the identity provider would and as an attacker who stole its token would.

## When to use

- An identity provider provisions accounts and groups into a service provider over SCIM 2.0 (create, update,
  deactivate, delete).
- The SCIM endpoint may be authenticated by a shared, weak, or long-lived bearer token.
- Provisioning may not be scoped per tenant, deprovisioning may not revoke access, or attribute mappings may
  over-grant.

## Scope check

Test SCIM provisioning only against services and directories you own or are authorized to assess, on
non-production tenants. Creating, modifying, and deleting accounts over SCIM changes real identity state, so use
test tenants and never provision, elevate, or delete accounts that are not yours. If you can't name the
authorization, stop.

## The loop

1. **Establish who may provision and what each event should grant first.** Name the trusted identity provider,
   how the SCIM endpoint authenticates it, which tenant its requests are confined to, and what privilege each
   group or role mapping is supposed to confer. This is the false-positive killer: an endpoint that
   authenticates the provider with a strong, scoped, rotatable credential, confines every request to the
   calling tenant, maps groups to exactly their intended privilege, and revokes access on deprovision is behaving
   correctly. Name the intended trust, then test each edge.

2. **Test the provisioning authentication.** Examine the credential the SCIM endpoint accepts: whether it is a
   strong per-provider secret or a shared, guessable, or long-lived bearer token, and whether it can be rotated
   and revoked. Try calling the endpoint with a missing, malformed, or another tenant's token. An endpoint that
   accepts a weak or shared token lets anyone holding it create accounts and grant groups.

3. **Test tenant scoping.** As one tenant's provisioning client, attempt to create, read, update, or delete
   users and groups belonging to another tenant, by external id, by user id, and by filter. Confirm every SCIM
   operation is confined to the calling tenant. A handler that acts cross-tenant lets one customer provision
   into another.

4. **Test group and role mapping.** Push group memberships and role attributes over SCIM and confirm each maps
   to exactly the privilege it should, not more. Attempt to add a provisioned user to an admin or elevated group
   the source attribute should not reach. A mapping that grants more than the attribute warrants is a
   provisioning-time privilege escalation.

5. **Test deprovisioning and attribute reassignment.** Deactivate and delete a provisioned user and confirm
   access is actually revoked (sessions, tokens, and group membership), not just a flag flipped. Separately,
   update sensitive attributes (email, external id, username, admin flag) and confirm the service validates them
   so an attacker cannot reassign or elevate an account. Lingering access after deprovision and unvalidated
   attribute updates are both live bugs.

6. **Confirm and record.** Confirm on a test tenant by provisioning or elevating an account with a weak or
   cross-tenant token, adding a user to a group beyond its mapping, or keeping access after a deprovision event,
   without touching real accounts. Kill the lead if the endpoint authenticates the provider with a strong scoped
   rotatable credential, confines requests to the calling tenant, maps groups to their intended privilege only,
   revokes access on deprovision, and validates attribute updates. Record the crafted or replayed SCIM request,
   the unauthorized account, group, or lingering access, and the weak provisioning auth or unvalidated attribute
   mapping.

## Where SCIM trust leaks

- **Weak or shared provisioning token.** A guessable, shared, or never-expiring bearer token on the SCIM
  endpoint hands account creation and group grants to anyone who holds it.
- **No tenant scoping.** A handler that does not confine operations to the calling tenant lets one provisioning
  client create or modify another tenant's users and groups.
- **Over-granting group or role mapping.** A group or role pushed over SCIM that maps to more privilege than the
  source attribute warrants escalates at provisioning time.
- **Deprovisioning that does not revoke.** A deactivate or delete that flips a flag but leaves sessions, tokens,
  or group membership intact leaves a departed user with access.
- **Unvalidated attribute updates.** Trusting email, external id, username, or an admin flag from a SCIM update
  lets an attacker reassign or elevate an account.

## Worked example (a confirm and a kill)

> **Confirm.** A service exposes a SCIM endpoint authenticated by a single shared bearer token that is the same
> across all customers and never rotated. On a test tenant, using that token, a request creates a user in a
> different tenant and adds them to that tenant's administrators group, because the handler does not scope the
> operation to the token's tenant. **Confirmed** cross-tenant account creation and privilege grant via shared
> provisioning token and missing tenant scoping, `critical`, remediation = issue a strong per-provider,
> per-tenant credential that can be rotated and revoked, and confine every SCIM operation to the calling
> tenant's users and groups.
>
> **Kill.** The SCIM endpoint authenticates each provider with a strong per-tenant credential that is rotatable
> and revocable, confines every create, read, update, and delete to the calling tenant, maps each group to
> exactly its intended privilege with admin groups unreachable from ordinary source attributes, revokes
> sessions and tokens on deprovision, and validates attribute updates. A cross-tenant call is rejected, an
> over-privilege group add fails, and a deprovisioned user loses access. **Killed**, `kill_reason` = "SCIM
> endpoint uses a strong scoped rotatable credential, confines every operation to the calling tenant, maps
> groups to their intended privilege only, revokes access on deprovision, and validates attributes; no crafted
> request provisions, elevates, or retains access it should not."

## Rationalizations to reject

- *"The SCIM token is secret."* → A shared or long-lived token leaks and cannot be revoked without breaking
  everyone; issue a strong per-provider credential that rotates, and treat it as an admin credential.
- *"Only our identity provider calls it."* → Anyone with the token calls it; confirm the endpoint scopes to the
  authenticated tenant and rejects cross-tenant ids and filters, not just that the provider is honest.
- *"Groups come from the IdP, so they are trusted."* → Confirm each group and role maps to exactly its intended
  privilege; an attribute the IdP sends should not be able to reach an admin group it was never meant to.
- *"Deactivating the user is enough."* → Verify sessions, tokens, and group membership are actually revoked; a
  flipped flag that leaves a live session is not deprovisioning.
- *"Attribute updates are routine."* → An unvalidated email or external-id change reassigns an account; validate
  sensitive attributes on every SCIM update.

## Executing this in practice

You need the SCIM endpoint's authentication (credential strength, scope, rotation), whether operations are
confined to the calling tenant, how groups and roles map to privilege, what deprovisioning actually revokes,
and how attribute updates are validated. Call the endpoint as the provider and as a token-holder attacking
another tenant, exercising create, update, group-add, delete, and attribute-change on test tenants. Reading the
provisioning handler shows the intended trust; an account created, elevated, or retained against policy shows
whether it holds.

## Related

- `auditing-jit-provisioning-and-role-mapping` - the just-in-time counterpart; SCIM pushes accounts ahead of
  login while JIT creates them at login, and both hinge on how a source attribute maps to privilege.
- `auditing-directory-sync-trust` - the bulk directory synchronization that often sits beside SCIM; both trust
  an external source to shape internal identity state.
- `auditing-multi-tenant-isolation` - the per-tenant scoping SCIM must enforce is that skill's whole subject,
  applied to the provisioning surface.
- `hunting-iam-privilege-escalation-paths` - a group or role over-granted at provisioning time is the entry to
  an escalation path that skill traces onward.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the crafted or replayed SCIM request, sink = the
  unauthorized account, group, or lingering access, evidence = the weak provisioning auth or unvalidated
  attribute mapping.
