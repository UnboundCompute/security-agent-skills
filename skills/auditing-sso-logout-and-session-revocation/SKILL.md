---
name: auditing-sso-logout-and-session-revocation
description: >-
  Audit single sign-on logout and session revocation for sessions that outlive the event meant to end them: a
  logout that clears the local application session but never ends the identity-provider session so re-login is
  silent, a single-logout flow the application ignores so signing out at the identity provider leaves downstream
  application sessions alive, an access or refresh token that keeps working after logout or after an admin
  disables the account, a session that survives a password reset or deprovisioning event, and a back-channel
  logout notification the application never processes. Covers SAML and OIDC single sign-on, single logout,
  back-channel logout, and token revocation across relying applications. Use when a user, admin, or identity
  provider ends a session and whether every downstream session and token actually terminates is the boundary.
  The logout or revocation event is the source, the session or token that keeps working is the sink, and the
  unpropagated logout or unrevoked token is the bug.
license: MIT
---

# Auditing SSO logout and session revocation: ending a session at one place must end it everywhere

Single sign-on spreads one login across many applications, and the hard part is not the login, it is the
logout. When a user signs out, an admin disables an account, a password is reset, or the identity provider ends
a session, every downstream session and token that login created is supposed to stop working, and often several
of them do not. A logout that clears only the local application cookie leaves the identity-provider session
alive, so the next visit signs the user back in silently and the logout was cosmetic. A single-logout flow the
application never implements means signing out at the identity provider leaves every relying application still
logged in. An access or refresh token minted during the session keeps working after logout or after the account
is disabled, because nothing revokes it. A session that survives a password reset lets a stolen session outlive
the credential change meant to shut it out. And a back-channel logout notification the application does not
process is an ended session the application never hears about. The audit ends a session by every available
means, user logout, identity-provider logout, admin disable, password reset, token revocation, and then checks
that every downstream session and token is actually dead. You audit this by logging out one way and trying to
keep using the session another.

## When to use

- Users sign in through SSO (SAML or OIDC) across one or more relying applications, and log out through the
  application, the identity provider, or an admin action.
- Logout may clear only the local session, single logout may be unimplemented, or tokens may outlive the event
  meant to end them.
- A session may survive a password reset, an account disable, or a deprovisioning event.

## Scope check

Test logout and revocation only against applications and identity providers you own or are authorized to
assess, with test accounts. Exercising logout, account disable, and password reset changes real session and
account state, so use test identities and never disable, reset, or revoke access for accounts that are not
yours. If you can't name the authorization, stop.

## The loop

1. **Establish what each ending event must terminate first.** Name every way a session can end (user logout,
   identity-provider logout, admin disable, password reset, explicit token revocation) and, for each, exactly
   which sessions and tokens it is supposed to kill across which applications. This is the false-positive killer:
   a system where each ending event propagates to every downstream session, revokes the associated access and
   refresh tokens, and processes back-channel logout is behaving correctly. Name the intended termination, then
   test each event.

2. **Test local logout completeness.** Log out of an application and check whether the identity-provider session
   also ends. Revisit the application and confirm you are not silently signed back in by a still-live
   identity-provider session. A logout that clears only the local cookie is cosmetic if the next request
   re-authenticates without a prompt.

3. **Test single logout propagation.** With sessions open in several relying applications, sign out at the
   identity provider (or one application that triggers single logout) and confirm every other application's
   session also ends. An application that ignores single-logout messages stays logged in after the user believes
   they signed out everywhere.

4. **Test token revocation after logout and disable.** Capture an access and refresh token from the session,
   then log out and, separately, have an admin disable the account. Confirm both tokens stop working promptly:
   the access token is rejected and the refresh token cannot mint a new one. A token that keeps working after
   logout or after the account is disabled is standing access nothing revoked.

5. **Test survival across password reset and deprovisioning.** With an active session and its tokens, reset the
   password and, separately, deprovision the user, then confirm the pre-existing session and tokens are
   invalidated. A session that outlives a password reset defeats the reset as a response to compromise, and one
   that outlives deprovisioning leaves a departed user with access.

6. **Confirm and record.** Confirm with test accounts by keeping a session or token working after a logout,
   single logout, account disable, password reset, or deprovisioning event that should have ended it, without
   touching real accounts. Kill the lead if every ending event propagates to all downstream sessions, revokes
   access and refresh tokens promptly, invalidates sessions on reset and deprovision, and processes back-channel
   logout. Record the logout or revocation event, the session or token that kept working, and the unpropagated
   logout or unrevoked token.

## Where session termination leaks

- **Local-only logout.** Clearing the application cookie while the identity-provider session stays alive lets
  the next visit sign the user back in silently.
- **Ignored single logout.** An application that does not process single-logout messages stays logged in after
  the user signed out at the identity provider.
- **Tokens that outlive logout.** An access or refresh token that keeps working after logout is standing access
  the logout did not revoke.
- **Survival across disable and reset.** A session or token that outlives an account disable, password reset, or
  deprovisioning leaves access in place after the event meant to remove it.
- **Unprocessed back-channel logout.** A back-channel logout notification the application never handles is an
  ended session it never hears about.

## Worked example (a confirm and a kill)

> **Confirm.** A user logs out of a relying application; the application deletes its own session cookie but sends
> nothing to the identity provider, and the OIDC access and refresh tokens issued during the session are not
> revoked. On a test account, after logout the refresh token still mints new access tokens and revisiting the
> application signs the user straight back in from the live identity-provider session. **Confirmed** session and
> token survival after logout via local-only logout and no token revocation, `high`, remediation = on logout end
> the identity-provider session (RP-initiated logout), revoke the access and refresh tokens, and honor
> back-channel logout so every relying application terminates.
>
> **Kill.** Logout triggers RP-initiated logout that ends the identity-provider session, revokes the access and
> refresh tokens so neither works afterward, and back-channel logout notifies every relying application to end
> its session. An admin disable and a password reset both invalidate existing sessions and tokens promptly. After
> any ending event, the captured token is rejected and no application silently re-authenticates. **Killed**,
> `kill_reason` = "every ending event ends the identity-provider session, revokes access and refresh tokens, and
> propagates to all relying applications; no session or token outlives the event meant to end it."

## Rationalizations to reject

- *"Logout deletes the session cookie."* → Confirm the identity-provider session also ends and tokens are
  revoked; a deleted cookie with a live IdP session re-authenticates on the next request.
- *"The token expires soon anyway."* → A short expiry is not revocation; a refresh token that still mints access
  tokens, or an access token valid for its remaining lifetime, is live access after logout.
- *"Single logout is optional."* → If the user believes signing out ends every session, an application that
  ignores single logout leaves a session the user thinks is closed.
- *"Password reset forces re-login."* → Confirm existing sessions and tokens are invalidated; a reset that leaves
  the old session alive does not lock out a session-stealing attacker.
- *"Disabling the account is immediate."* → Verify it revokes live sessions and tokens, not just blocks new
  logins; a disabled account with a working refresh token still has access.

## Executing this in practice

You need every way a session can end, what each is supposed to terminate, and how tokens are revoked and logout
is propagated (RP-initiated, single, and back-channel logout). With test accounts, capture a session and its
tokens, trigger each ending event, and try to keep using the session and tokens afterward. Reading the logout
and revocation handling shows the intended termination; a session or token that survives an ending event shows
what it fails to end.

## Related

- `auditing-session-lifecycle-and-fixation` - the single-application session lifecycle; this skill extends it to
  ending a session across every relying application at once.
- `auditing-saml-and-oidc-federation-trust` - the federation that created the sessions; logout and single logout
  are the teardown side of that trust.
- `auditing-account-recovery-and-reset-trust` - a password reset is one of the ending events here; whether it
  invalidates live sessions is shared ground.
- `auditing-oauth-token-audience-and-scope-trust` - the tokens that must be revoked on logout are that skill's
  subject; an unrevoked token is standing access.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the logout or revocation event, sink = the session or
  token that keeps working, evidence = the unpropagated logout or unrevoked token.
