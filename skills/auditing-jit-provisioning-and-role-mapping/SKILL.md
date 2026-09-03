---
name: auditing-jit-provisioning-and-role-mapping
description: >-
  Audit just-in-time account provisioning at federated (SAML or OIDC) login for trust misplaced in the assertion
  that drives it: a first login that creates an account and assigns roles from identity-provider claims (groups,
  email domain, department) the service never validates, a claim-to-role mapping that grants more privilege than
  the claim should or defaults new users into a privileged role, an email or domain claim trusted to auto-join a
  tenant so an attacker with a lookalike address lands inside it, and a JIT update that re-elevates an account on
  every login from mutable claims. Use when a federated login provisions an account and the mapping from
  assertion claims to local roles and tenancy is the boundary. The attacker-shaped login assertion is the
  source, the over-privileged or wrong-tenant provisioned account is the sink, and the unvalidated claim or
  over-granting role mapping is the bug.
license: MIT
---

# Auditing JIT provisioning and role mapping: the first login decides the role, from claims you did not issue

Just-in-time provisioning creates a local account the first time someone logs in through a federated identity
provider, and often updates that account's roles on every subsequent login, all from the claims in the login
assertion. The claims, group memberships, an email, a domain, a department, come from the identity provider,
not from you, so JIT hands account creation and role assignment to whatever the assertion says. That is fine
when the mapping is tight and the claims are validated, and dangerous when either is loose. A claim-to-role
mapping that grants more privilege than the claim should, or defaults every new user into a privileged role,
elevates at first login. An email or domain claim trusted to auto-join a tenant or organization lets an
attacker with a lookalike or attacker-controlled address land inside it. A JIT update that re-reads mutable
claims every login can re-elevate or reassign an account an admin previously corrected. And any claim provisioned
without validation, an unverified email, a group the attacker can influence, an unsigned or overtrusted
assertion field, becomes an access grant. The audit follows a federated login into the account and roles it
creates and checks that every claim is validated and every mapping is minimal. You audit this by logging in
with shaped assertions and seeing what account and privilege you are given.

## When to use

- A federated login (SAML or OIDC) provisions a local account and assigns roles or tenancy from assertion
  claims on first login, and may update them on repeat logins.
- A claim-to-role mapping may over-grant, default new users into a privileged role, or trust an email or domain
  claim to auto-join an organization.
- Claims (email, groups, department) may be provisioned without validation.

## Scope check

Test JIT provisioning only against applications and identity providers you own or are authorized to assess, on
non-production tenants. Logging in with shaped assertions creates real accounts and role grants, so use test
identity providers and tenants and never provision or elevate access in an organization that is not yours. If
you can't name the authorization, stop.

## The loop

1. **Establish the intended claim-to-role and claim-to-tenant mapping first.** Name which claims the service
   trusts, how each maps to a local role, how tenancy or organization membership is decided, and what a brand
   new user should get by default. This is the false-positive killer: a service that validates every claim it
   provisions from, maps claims to least privilege, defaults new users into no privileged role, and decides
   tenancy from a verified, non-spoofable signal is behaving correctly. Name the intended mapping, then test
   shaped logins.

2. **Test the claim-to-role mapping.** Log in with assertions carrying various group and role claims and confirm
   each maps to exactly the intended privilege. Attempt a claim that should map to an ordinary role and check it
   cannot reach an admin or elevated role, and confirm a new user with no privileged claim is not defaulted into
   one. A mapping that over-grants or defaults high is a first-login escalation.

3. **Test tenant and organization auto-join.** Log in with an email or domain claim and confirm how the service
   decides which tenant or organization the account joins. Try a lookalike domain, a subdomain, and an
   attacker-controlled address, and confirm none auto-joins an organization it should not. An email or domain
   claim trusted to auto-join lets an attacker land inside a tenant.

4. **Test claim validation.** For each provisioned claim, confirm the service validates it: the email is
   verified, the assertion is signed and the claim is within its authority, and the group is one the identity
   provider is authorized to assert. Provision from an unverified email and an attacker-influenced group and
   confirm both are rejected. An unvalidated claim is an access grant the attacker writes.

5. **Test JIT update on repeat login.** Log in again with changed claims and confirm whether the service
   re-reads them to re-elevate or reassign the existing account. Check whether an admin's manual correction
   (demotion, tenant move) survives the next federated login. A JIT update that blindly re-applies mutable
   claims undoes local administration every login.

6. **Confirm and record.** Confirm on a test tenant by provisioning an over-privileged account from a shaped
   role claim, auto-joining a tenant from a lookalike email, provisioning from an unverified or
   attacker-influenced claim, or re-elevating an account on repeat login, without touching a real organization.
   Kill the lead if the service validates every provisioned claim, maps claims to least privilege, defaults new
   users into no privileged role, decides tenancy from a verified signal, and does not let a JIT update override
   local correction. Record the attacker-shaped login assertion, the over-privileged or wrong-tenant account,
   and the unvalidated claim or over-granting mapping.

## Where JIT provisioning leaks

- **Over-granting role mapping.** A claim that maps to more privilege than it should, or a default that puts new
  users in a privileged role, escalates at first login.
- **Email or domain auto-join.** Trusting an email or domain claim to auto-join a tenant lets an attacker with a
  lookalike or controlled address land inside an organization.
- **Unvalidated claims.** Provisioning from an unverified email, an attacker-influenced group, or an
  overtrusted assertion field turns a claim into an access grant.
- **Blind JIT updates.** Re-reading mutable claims on every login re-elevates or reassigns accounts and undoes
  an admin's local corrections.
- **Trusting unsigned or out-of-authority claims.** Accepting a claim the identity provider is not authorized to
  assert, or from an unsigned assertion, lets the attacker set roles directly.

## Worked example (a confirm and a kill)

> **Confirm.** An application provisions accounts at first SAML login and joins the user to an organization based
> on the email domain in the assertion, with any member of the `staff` group mapped to an admin role. On a test
> identity provider, an assertion with a lookalike domain and a `staff` group claim provisions an admin account
> inside the target organization, because the service trusts the domain to auto-join and maps `staff` straight to
> admin. **Confirmed** wrong-tenant admin provisioning via trusted domain and over-granting mapping, `critical`,
> remediation = decide tenancy from a verified, claimed-domain-ownership signal rather than the raw email domain,
> map groups to least privilege with admin unreachable from a broad group, and verify the email before
> provisioning.
>
> **Kill.** The service provisions only from validated claims (verified email, signed assertion, groups within
> the provider's authority), maps each claim to least privilege with admin roles unreachable from ordinary
> claims, defaults new users into no privileged role, decides organization membership from a verified
> domain-ownership signal, and does not let a repeat login override an admin's manual role or tenant correction.
> A lookalike domain does not auto-join, a broad group does not reach admin, and an unverified email does not
> provision. **Killed**, `kill_reason` = "every provisioned claim is validated, claims map to least privilege
> with no high default, tenancy comes from a verified signal, and JIT updates do not override local correction;
> no shaped login provisions over-privileged or wrong-tenant access."

## Rationalizations to reject

- *"The claims come from the IdP, so they are trusted."* → Confirm each claim is within the provider's authority
  and the assertion is signed; a claim the attacker can influence or that the provider should not assert is not
  trustworthy just because it arrived over federation.
- *"We map groups to roles, that is standard."* → Confirm the mapping is least privilege and admin is unreachable
  from a broad group; a `staff`-to-admin mapping is standard and wrong.
- *"New users get a default role."* → Confirm the default is not privileged; defaulting new federated users into
  an elevated role is a first-login escalation for anyone who can log in.
- *"Email domain identifies the company."* → An email domain is spoofable in an assertion and lookalikes exist;
  decide tenancy from verified domain ownership, not the raw claim.
- *"JIT keeps roles in sync."* → Confirm it does not undo local corrections; blindly re-applying mutable claims
  every login re-elevates accounts an admin already demoted.

## Executing this in practice

You need which claims the service provisions from, how each maps to a role and to tenancy, how claims are
validated, and whether repeat logins re-apply mutable claims over local changes. On a test identity provider,
log in with shaped role, group, and email claims, with lookalike and unverified addresses, and on repeat with
changed claims. Reading the provisioning and mapping code shows the intended trust; an over-privileged or
wrong-tenant account from a shaped login shows whether it holds.

## Related

- `auditing-scim-provisioning-trust` - the push counterpart; SCIM provisions ahead of login while JIT provisions
  at login, and both turn a source attribute into a role and a tenant.
- `auditing-saml-and-oidc-federation-trust` - the assertion validation JIT depends on; an unsigned or wrapped
  assertion feeds JIT attacker-chosen claims.
- `auditing-idp-initiated-flow-trust` - an IdP-initiated login is a common JIT trigger; the two combine when an
  unsolicited assertion provisions an account.
- `hunting-iam-privilege-escalation-paths` - a role over-granted at first login is the start of an escalation
  path that skill traces onward.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the attacker-shaped login assertion, sink = the
  over-privileged or wrong-tenant provisioned account, evidence = the unvalidated claim or over-granting role
  mapping.
