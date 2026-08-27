---
name: auditing-cross-account-role-trust-boundaries
description: >-
  Audit cross-account IAM role assumption for trust policies that let the wrong principal assume a role: a
  trust policy with a wildcard or overbroad principal, a missing or unverifiable external ID on a
  third-party role, a confused-deputy path where a vendor assumes your role on any customer's behalf, and a
  role chain that reaches privileges the origin principal should never hold. Covers AWS assume-role trust
  policies, condition keys that should scope who may assume, and the transitive reach of one assumption
  into the next. Use when roles in one account can be assumed from another account, a partner, or a service,
  and the trust policy is the boundary. The external principal permitted by the trust policy is the source,
  the assume-role grant is the sink, and the trust scope wider than the intended relationship is the bug.
license: MIT
---

# Auditing cross-account role trust boundaries: when the trust policy lets the wrong caller in

A role's trust policy decides who may assume it, and in a multi-account or partner setup that policy is the
whole boundary between organizations. It fails in quiet ways. A principal element written as a wildcard, or
as an entire account rather than a specific role, lets any identity in that account assume the role. A
third-party integration role that omits or does not verify an external ID is open to the confused-deputy
attack, where the vendor is tricked into assuming your role on an attacker's behalf. And assumption chains,
where one role assumes another, can carry a low-privilege origin into high-privilege territory the trust
policies never meant to connect. You audit these by reading each trust policy for who it actually admits and
by following where an admitted principal can go next.

## When to use

- Roles in one account can be assumed from another account, a partner organization, or a third-party vendor.
- A trust policy uses a broad principal, or a third-party role relies on an external ID for scoping.
- One assumed role can assume another, forming a chain that may reach privileges the origin should not.

## Scope check

Test role assumption only in accounts and organizations you own or are authorized to assess. A confirming
assumption obtains real credentials in the target account, so stay inside the authorized accounts and never
assume a role belonging to an organization outside your engagement. If you can't name the authorization,
stop.

## The loop

1. **Establish the intended trust relationship first.** For each cross-account role, name who is supposed to
   assume it and why: which specific partner role, which service, which internal account. This is the
   false-positive killer: a trust policy that admits exactly the intended principal under the intended
   conditions is correct even in a sprawling multi-account setup. Name the intended relationship, then read
   the policy against it.

2. **Read the principal element for who it actually admits.** A trust policy naming an entire account as the
   principal admits every identity in that account, not just the intended role. A wildcard principal admits
   anyone, gated only by whatever conditions follow. Determine the true set of principals the policy allows,
   not the one the author had in mind.

3. **Check third-party roles for external-ID enforcement.** A role a vendor assumes on your behalf must
   require an external ID that the vendor sets per customer and that an attacker cannot guess or supply,
   otherwise the vendor can be induced to assume your role for someone else. Confirm the trust policy
   requires the external ID as a condition and that the value is unique and secret to your relationship, not
   a shared or predictable string.

4. **Confirm the conditions actually scope the assumption.** Condition keys on the trust policy (source
   account, source identity, external ID, network conditions) are the difference between a broad principal
   and a safe one. Read whether the conditions are present, whether they use the correct keys, and whether
   they can be satisfied by an attacker. A condition that references a spoofable or caller-controlled value
   does not scope anything.

5. **Follow the assumption chain.** Where an assumed role can assume a further role, trace the chain to its
   end and ask whether the origin principal should be able to reach the final role's privileges. A chain that
   lets a partner or a low-privilege account step through intermediate roles into administrative access is a
   trust-boundary break even when each individual policy looks reasonable.

6. **Confirm and record.** Confirm by assuming the role from a principal that should not be admitted (within
   owned accounts), or by demonstrating a chain that carries an origin to privileges beyond its intended
   reach, without touching accounts outside the engagement. Kill the lead if every trust policy admits only
   the specific intended principal, third-party roles require a unique secret external ID, conditions scope
   the assumption to unspoofable values, and no chain reaches unintended privilege. Record the admitted
   principal, the assume-role sink, and the trust scope beyond intent.

## Where cross-account trust boundaries leak

- **An account-wide principal admits everyone in that account.** Naming the account instead of the specific
  role grants assumption to every identity it contains.
- **A missing external ID is a confused-deputy opening.** Without it, a vendor can be induced to assume your
  role for an attacker's benefit.
- **A wildcard principal relies entirely on the conditions.** If the conditions are absent or spoofable, the
  role is effectively open.
- **Assumption chains connect trust domains the policies never meant to link.** A safe-looking pair of
  policies can compose into an origin-to-admin path.
- **A predictable external ID is no boundary.** If the value is shared, guessable, or attacker-suppliable, it
  does not scope the third-party assumption.

## Worked example (a confirm and a kill)

> **Confirm.** A data-integration role trusts a vendor account but omits the external-ID condition. Because
> the vendor assumes the role for many customers, an attacker who is also a vendor customer induces the
> vendor to assume this role on their behalf, obtaining credentials in the target account. **Confirmed**
> confused-deputy cross-account assumption, `high`, remediation = add a required external-ID condition to the
> trust policy with a value unique and secret to this relationship, and confirm the vendor sends it on every
> assume-role call.
>
> **Kill.** Every cross-account role names the specific partner role as principal, third-party roles require a
> unique secret external ID as a condition, and no assumed role can chain into privileges beyond its intended
> reach. An assumption attempt from any other principal, or without the external ID, is denied. **Killed**,
> `kill_reason` = "trust policies admit only the specific intended role under a unique-external-ID condition
> and no chain reaches unintended privilege; no unadmitted principal can assume the role."

## Rationalizations to reject

- *"Only our partner's account is trusted."* → Naming the account admits every identity in it; scope the
  principal to the specific role, not the whole account.
- *"The external ID is set, so it is safe."* → Confirm it is unique and secret to your relationship; a shared
  or guessable value provides no scoping.
- *"Each policy is fine on its own."* → Trace the chain; safe individual policies can compose into an
  origin-to-admin path.
- *"The condition restricts it."* → Confirm the condition uses the correct key and an unspoofable value; a
  condition on a caller-controlled field is not a boundary.
- *"It is inside our organization."* → Cross-account within one organization still crosses a trust boundary;
  the trust policy is the control regardless.

## Executing this in practice

You need each cross-account role's trust policy, the intended principal for each, what the principal element
actually admits, the external-ID and condition scoping on third-party roles, and the assumption chains that
lead onward. For each role, compare the admitted set to the intended relationship and follow where an
admitted principal can chain. Reading the policy shows who is meant to assume; assuming from an unintended
principal in an owned account shows whether the boundary holds.

## Related

- `hunting-iam-privilege-escalation-paths` - the in-account escalation companion; a cross-account entry often
  lands on a role that then escalates within the target account.
- `mapping-service-account-impersonation-chains` - the same assumption-chain reasoning in cloud identity,
  where impersonation rather than assume-role carries the principal onward.
- `auditing-cicd-oidc-trust` - a federated CI principal assuming a cloud role is a cross-account trust with
  the same subject-scoping requirements.
- `adjudicating-taint-paths` - use it to trace an admitted principal through an assumption chain to a
  privileged final role.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the external principal the trust policy permits,
  sink = the assume-role grant, evidence = the trust scope beyond the intended relationship.
