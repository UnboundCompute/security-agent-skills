---
name: auditing-break-glass-account-trust
description: >-
  Audit emergency break-glass and privileged-access accounts for the ways their standing power outlives the
  emergency they exist for: a break-glass account with a static, shared, or never-rotated credential that is a
  permanent superuser login rather than a sealed last resort, an emergency account excluded from the MFA,
  conditional-access, or logging controls covering other admins so its use is easy and invisible, a
  just-in-time elevation that grants more than the task needs or is not revoked when the task ends, an
  emergency-access path with no approval, time bound, or alerting so it is used routinely, and a break-glass use
  that triggers no review afterward. Use when a standing emergency identity or an elevation path holds power
  beyond ordinary admins and the controls around when and how it is used are the boundary. The break-glass credential or elevation request is the source, the unbounded or unaudited privileged
  access is the sink, and the missing rotation, control coverage, or use-time alerting is the bug.
license: MIT
---

# Auditing break-glass account trust: the last-resort key must be sealed, watched, and rare

A break-glass account exists for the emergency where normal admin access is broken, the identity provider is
down, everyone is locked out, so it holds the highest privilege and, by design, bypasses some of the controls
that gate everyone else. That combination, maximum power plus fewer gates, is precisely why it is dangerous when
its own controls are loose. The account is supposed to be a sealed last resort: strong unique credential,
rotated after each use, covered by logging and alerting, used rarely and reviewed every time. The failures are
where it stops being sealed. A static, shared, or never-rotated credential turns the emergency key into a
permanent superuser login anyone who once saw it can use. An emergency account excluded from the multi-factor,
conditional-access, and logging controls that cover other admins is both easy to use and invisible when used. A
just-in-time or privileged-access-management elevation that grants more than the task needs, or is not revoked
when the task ends, leaves standing privilege behind a temporary label. An emergency path with no approval, no
time bound, and no alerting gets used routinely instead of exceptionally. And a break-glass use that triggers no
review means the one time it mattered, nobody looked. The audit checks that the emergency identity is sealed,
watched, minimally scoped, and reviewed, so its power is available in a real emergency and inert otherwise. You
audit this by asking who could use it right now, whether anyone would notice, and whether it would ever be
un-elevated.

## When to use

- A system has a break-glass or emergency-access account, or a just-in-time privileged-elevation path, that
  holds power beyond ordinary admins.
- The emergency credential may be static, shared, or never rotated, or the account may be excluded from MFA,
  conditional-access, or logging controls.
- An elevation may over-grant, may not be revoked when the task ends, or a break-glass use may trigger no
  alerting or review.

## Scope check

Test break-glass and privileged-elevation controls only against accounts and platforms you own or are
authorized to assess, with test accounts. Exercising an emergency account or elevation uses real high privilege,
so use test identities and never log in as, elevate through, or alter a break-glass account that is not yours.
If you can't name the authorization, stop.

## The loop

1. **Establish how the emergency identity is meant to be sealed and watched first.** Name the break-glass and
   elevation paths, what credential each uses and how it rotates, which controls (MFA, conditional access,
   logging, alerting) cover it, what scope an elevation grants, and how each use is approved and reviewed. This
   is the false-positive killer: an emergency account with a strong unique credential rotated after use, covered
   by logging and alerting, elevations scoped to the task and revoked when it ends, and every use reviewed is
   behaving correctly. Name the intended seal, then test each control.

2. **Test the credential seal.** Examine the break-glass credential: whether it is strong and unique or static,
   shared, or reused, and whether it is rotated after each use and stored under strict control (split knowledge,
   sealed vault). A credential anyone who once saw it can still use, or one never rotated, is a permanent
   superuser login rather than a sealed last resort.

3. **Test control coverage.** Confirm the emergency account is subject to the same logging and alerting as every
   other admin, and understand exactly which gates it bypasses and why. An emergency account excluded from
   logging is invisible when used; one excluded from MFA and conditional access with no compensating alerting is
   both easy to use and unnoticed, which is the opposite of a watched last resort.

4. **Test elevation scope and revocation.** For a just-in-time or privileged-access elevation, confirm it grants
   only what the task needs and is revoked when the task ends or the time window closes. Elevate, complete a
   task, and check the privilege is actually removed. An elevation that over-grants, or that leaves privilege
   standing after the task, is permanent power behind a temporary label.

5. **Test approval, time bound, and alerting on use.** Confirm using the emergency path requires approval where
   feasible, is time-bounded, and fires an immediate alert and a recorded justification. Use the path and check
   that an alert reaches someone and a review is prompted. An emergency path with no approval, no expiry, and no
   alert gets used routinely and silently instead of exceptionally and visibly.

6. **Confirm and record.** Confirm with test accounts by using a static or unrotated break-glass credential,
   acting through an emergency account that logs or alerts nothing, keeping privilege after an elevation should
   have ended, or exercising an emergency path that prompts no review, without touching real emergency accounts.
   Kill the lead if the credential is strong, unique, and rotated after use, the account is logged and alerted,
   elevations are scoped and revoked, and every use is reviewed. Record the break-glass credential or elevation
   request, the unbounded or unaudited privileged access, and the missing rotation, control coverage, or
   use-time alerting.

## Where break-glass trust leaks

- **Static or shared credential.** A break-glass password that is shared, reused, or never rotated is a permanent
  superuser login for anyone who ever saw it, not a sealed last resort.
- **Excluded from controls.** An emergency account left out of logging is invisible when used, and one left out
  of MFA and conditional access with no compensating alerting is easy and silent to abuse.
- **Over-granting or non-revoked elevation.** A just-in-time elevation that grants more than the task needs, or
  is not revoked when the task ends, leaves standing privilege behind a temporary label.
- **No approval, time bound, or alert.** An emergency path with no approval, no expiry, and no alerting gets used
  routinely and unnoticed instead of exceptionally and visibly.
- **No post-use review.** A break-glass use that triggers no review means the highest-privilege action in the
  system happens with nobody looking afterward.

## Worked example (a confirm and a kill)

> **Confirm.** A cloud tenant keeps a break-glass global-admin account with a static password stored in a shared
> document, excluded from MFA and conditional access "so it always works in an emergency," and its sign-ins raise
> no alert. On a test tenant, the account logs in with the shared static credential, performs privileged actions,
> and nothing alerts or is reviewed, so it functions as an always-available, unwatched superuser rather than a
> sealed last resort. **Confirmed** unbounded and unaudited break-glass access via static shared credential and
> excluded controls, `high`, remediation = store the credential under split knowledge in a sealed vault and
> rotate it after each use, keep the account covered by logging with an immediate alert on any sign-in, and
> require a recorded justification and post-use review for every use.
>
> **Kill.** The break-glass credential is strong and unique, held under split knowledge in a sealed vault, and
> rotated after every use; the account is fully logged and any sign-in fires an immediate alert with a required
> justification; just-in-time elevations are scoped to the task and revoked when it ends; and every use triggers
> a review. Using the account is possible in a real emergency but never quiet or routine, and elevation never
> outlives its task. **Killed**, `kill_reason` = "the emergency credential is strong, sealed, and rotated after
> use, the account is logged and alerted, elevations are scoped and revoked, and every use is reviewed; the
> last-resort power is available but bounded and watched."

## Rationalizations to reject

- *"It has to work when everything else is down."* → Available is not the same as unwatched or static; a sealed,
  rotated credential with alerting still works in an emergency and is not a standing superuser the rest of the
  time.
- *"We exclude it from MFA so we never get locked out."* → An emergency account excluded from controls with no
  compensating alerting is the easiest and quietest account to abuse; add compensating monitoring for every gate
  it bypasses.
- *"Only a few people know the password."* → A shared static password known to a few and never rotated is known
  to everyone who ever left; make it unique, sealed, and rotated after each use.
- *"The elevation is temporary."* → Confirm the privilege is actually revoked when the task or window ends; an
  elevation that is not un-done is permanent power with a temporary name.
- *"We would notice if it were misused."* → Without logging, alerting, and post-use review you would not; verify
  a use actually reaches someone and prompts a review.

## Executing this in practice

You need the break-glass and elevation paths, each credential's strength and rotation, which controls cover or
exclude the account, the scope and revocation of elevations, and the approval, time bound, alerting, and review
around each use. With test accounts, use the credential, act through the account, elevate and complete a task,
and check what is logged, alerted, revoked, and reviewed. Reading the emergency-access procedure and platform
configuration shows the intended seal; an account that is static, silent, over-granted, or un-reviewed shows
where the seal is broken.

## Related

- `hunting-iam-privilege-escalation-paths` - a loosely controlled break-glass account or elevation is a
  high-value escalation target; that skill traces the paths that reach it.
- `auditing-mfa-enrollment-and-reset-abuse` - the controls a break-glass account often bypasses; whether the
  bypass is compensated is shared ground.
- `mapping-service-account-impersonation-chains` - emergency and highly privileged identities are prime nodes in
  an impersonation chain; that skill maps where their authority leads.
- `reviewing-secrets-manager-access-policy-trust` - where the sealed break-glass credential should live and who
  may retrieve it is that skill's subject.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the break-glass credential or elevation request, sink =
  the unbounded or unaudited privileged access, evidence = the missing rotation, control coverage, or use-time
  alerting.
