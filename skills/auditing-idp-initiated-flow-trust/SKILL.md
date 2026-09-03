---
name: auditing-idp-initiated-flow-trust
description: >-
  Audit identity-provider-initiated single sign-on for trust placed in an unsolicited assertion the application
  never asked for: an IdP-initiated SAML response the service provider accepts with no matching request so there
  is no request state to bind it to, an unsolicited assertion an attacker captures and replays or delivers to a
  victim to log them into an attacker-chosen account, a RelayState value trusted as a redirect target so it
  becomes an open redirect, an assertion with no or a too-wide audience so it is accepted by a service provider
  it was not meant for, and a missing replay defense (no one-time-use, weak expiry) that lets one assertion be
  used more than once. Use when an application accepts a login assertion it did not request and the validation of
  that unsolicited assertion is the boundary. The unsolicited or replayed assertion is the source, the unintended
  authenticated session is the sink, and the missing request binding, audience, or replay defense is the bug.
license: MIT
---

# Auditing IdP-initiated flow trust: an unsolicited login has no request to anchor it, so validate everything else

In service-provider-initiated single sign-on the application starts the flow, so it has a request on file to
bind the returning assertion to: a request id it can require the response to reference, and request state it can
check. IdP-initiated single sign-on removes that anchor. Login begins at the identity provider and the
application receives an assertion it never asked for, with no matching request, no InResponseTo to check, and no
request state to compare against. Everything the request would have pinned down, that this response answers a
login this browser started, must now be validated some other way, and often is not. An unsolicited assertion an
attacker captures can be replayed, or delivered to a victim, to log a browser into an attacker-chosen account. A
RelayState value trusted as a redirect target becomes an open redirect or drops the user into an
attacker-controlled context after login. An assertion with no audience restriction, or one too wide, is accepted
by a service provider it was never meant for. And without a replay defense, one-time-use, a tight expiry, a
consumed-assertion cache, the same assertion logs in again and again. The audit treats an unsolicited assertion
as untrusted, checks its signature, audience, timing, and single-use, and refuses to trust RelayState as a
destination. You audit this by capturing an IdP-initiated assertion and trying to replay it, redirect with it,
or use it at the wrong service provider.

## When to use

- An application accepts IdP-initiated single sign-on: an unsolicited SAML assertion (or similar) with no
  matching service-provider request.
- The assertion may be accepted with no request binding, no or a too-wide audience, or no replay defense.
- RelayState may be trusted as a redirect target.

## Scope check

Test IdP-initiated flows only against applications and identity providers you own or are authorized to assess,
with test accounts. Capturing and replaying assertions logs into real sessions, so use test identities and
never replay an assertion or log into an account that is not yours. If you can't name the authorization, stop.

## The loop

1. **Establish how an unsolicited assertion is validated first.** Name what the service provider checks on an
   IdP-initiated assertion with no request to anchor it: signature and trusted issuer, audience restriction,
   timing (NotBefore, NotOnOrAfter), single-use enforcement, and how RelayState is handled. This is the
   false-positive killer: a service provider that verifies the signature and issuer, enforces a narrow audience,
   checks tight timing, consumes each assertion once, and treats RelayState only as a validated internal target
   is behaving correctly. Name the intended validation, then test each check.

2. **Test replay and single-use.** Capture a valid IdP-initiated assertion and submit it a second time. Confirm
   the service provider enforces single-use (a consumed-assertion cache plus a tight expiry window) so a
   captured assertion cannot be replayed. An unsolicited assertion with no replay defense logs in again every
   time it is submitted, and can be delivered to a victim's browser.

3. **Test audience restriction.** Examine the assertion's audience and try to use it at a different service
   provider or endpoint. Confirm the audience is present and narrow so the assertion is accepted only by its
   intended service provider. A missing or too-wide audience lets an assertion minted for one application log in
   to another.

4. **Test RelayState handling.** Set RelayState to an external URL and to an attacker-controlled path and confirm
   the service provider does not redirect to it. RelayState must be treated as an opaque token mapping to a
   validated internal destination, not as a redirect target. A trusted RelayState is an open redirect that lands
   the post-login user in an attacker-controlled context.

5. **Test signature, issuer, and timing.** Confirm the assertion's signature is verified against the trusted
   identity provider's key, the issuer is on the allowlist, and NotBefore and NotOnOrAfter are enforced with
   little clock skew. Try an assertion from an untrusted issuer, an unsigned or altered one, and an expired one,
   and confirm each is rejected. A weak signature, issuer, or timing check accepts an assertion it should not.

6. **Confirm and record.** Confirm with test accounts by replaying a captured assertion, using one at the wrong
   service provider, redirecting through RelayState, or logging in with an assertion from an untrusted issuer or
   past its window, without touching real accounts. Kill the lead if the service provider verifies signature and
   issuer, enforces a narrow audience and tight timing, consumes each assertion once, and treats RelayState only
   as a validated internal target. Record the unsolicited or replayed assertion, the unintended authenticated
   session, and the missing request binding, audience, or replay defense.

## Where IdP-initiated trust leaks

- **No replay defense.** An unsolicited assertion with no single-use enforcement or a loose expiry replays every
  time it is submitted and can be delivered to a victim.
- **Missing or too-wide audience.** An assertion with no audience restriction, or one covering more than its
  intended service provider, is accepted where it was never meant to be used.
- **Trusted RelayState.** RelayState treated as a redirect target is an open redirect that lands the post-login
  user in an attacker-controlled destination.
- **Weak signature or issuer check.** Accepting an unsigned, altered, or untrusted-issuer assertion lets an
  attacker mint the login directly.
- **Loose timing.** A wide validity window or generous clock skew widens the replay window and accepts stale
  assertions.

## Worked example (a confirm and a kill)

> **Confirm.** A service provider accepts IdP-initiated SAML responses, does not track consumed assertions, uses
> a generous validity window, and redirects the user to the RelayState value after login. On a test account, a
> captured assertion is replayed twice and logs in both times, and setting RelayState to an external URL
> redirects the authenticated user off-site, because there is no single-use enforcement and RelayState is trusted
> as a destination. **Confirmed** assertion replay and open redirect in IdP-initiated flow, `high`, remediation =
> enforce single-use with a consumed-assertion cache and a tight expiry, and treat RelayState as an opaque token
> mapping to a validated internal target, never as a redirect URL.
>
> **Kill.** The service provider verifies the signature against the trusted issuer, enforces a narrow audience so
> the assertion works only here, checks NotBefore and NotOnOrAfter with minimal skew, consumes each assertion
> once so a replay is rejected, and maps RelayState only to a validated internal destination. A replayed
> assertion fails, one for another service provider is refused, and an external RelayState does not redirect.
> **Killed**, `kill_reason` = "unsolicited assertions are signature- and issuer-verified, audience- and
> time-restricted, single-use, and RelayState is a validated internal target; no assertion replays, crosses
> service providers, or redirects off-site."

## Rationalizations to reject

- *"There is no request because the IdP starts it."* → That is exactly why the other checks must be strict; with
  no request to anchor the assertion, audience, timing, and single-use are the only binding left.
- *"The assertion is signed."* → A signed assertion still replays if it is not single-use and still works
  elsewhere if its audience is wide; verify signature and audience and replay defense, not signature alone.
- *"RelayState is just where to send them."* → A trusted RelayState is an open redirect; map it to a validated
  internal destination and never redirect to its raw value.
- *"The assertion expires quickly."* → A short expiry narrows but does not close the replay window; enforce
  single-use with a consumed-assertion cache on top of the expiry.
- *"Our IdP is trusted."* → Confirm the issuer allowlist and signature verification actually enforce that; a weak
  check accepts an assertion from an issuer you never trusted.

## Executing this in practice

You need how the service provider validates an unsolicited assertion (signature, issuer, audience, timing,
single-use) and how it handles RelayState. With test accounts, capture an IdP-initiated assertion and replay it,
use it at another service provider, drive RelayState to an external target, and submit untrusted-issuer and
expired assertions. Reading the assertion-consumption code shows the intended validation; a replayed,
cross-provider, or redirecting assertion shows what the validation omits.

## Related

- `auditing-saml-and-oidc-federation-trust` - the general assertion validation; IdP-initiated flow is the case
  where the request anchor is absent and the other checks must carry the whole binding.
- `auditing-jit-provisioning-and-role-mapping` - an unsolicited assertion often triggers JIT provisioning; the
  two combine when an IdP-initiated login creates an account.
- `auditing-sso-logout-and-session-revocation` - the teardown side of the session an unsolicited assertion
  creates; both concern sessions the application did not fully originate.
- `auditing-cors-and-cross-origin-trust` - a redirected post-login user landing in an attacker-controlled
  cross-origin context is that skill's concern; RelayState-as-redirect is where the two meet.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the unsolicited or replayed assertion, sink = the
  unintended authenticated session, evidence = the missing request binding, audience, or replay defense.
