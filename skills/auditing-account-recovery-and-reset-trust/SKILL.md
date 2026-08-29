---
name: auditing-account-recovery-and-reset-trust
description: >-
  Audit password reset and account recovery flows for the trust that lets an attacker take over an account: a
  reset token that is guessable, long-lived, reusable, or not bound to the account it was issued for, a
  recovery path that verifies a weaker factor than login and bypasses multi-factor, a reset link whose host
  comes from an attacker-controllable header so the token leaks, and a recovery that trusts an unverified email
  or phone change to redirect the reset. Covers the recovery surface of authentication systems, where resetting
  a credential or recovering access is the alternate door into an account. Use when an application offers
  password reset or account recovery and that flow is a path to authentication. The attacker-driven recovery
  request is the source, the account it takes over is the sink, and the weak token, bypassed factor, or leaked
  reset link that grants it is the bug.
license: MIT
---

# Auditing account recovery and reset trust: the alternate door into every account

Every login has a back door built in on purpose: the password reset and account recovery flow. It exists so a
locked-out user can get back in, which means it is, by design, a way to gain access to an account without the
password. That makes it a prime takeover target, and it is frequently weaker than the front door it protects.
The reset token may be guessable, long-lived, reusable, or not bound to the account, so an attacker can
predict or replay it. The recovery path may verify a weaker factor than login and skip the multi-factor
requirement, so recovery becomes the multi-factor bypass. The reset link's host may come from a request header
an attacker controls, so the emailed token leaks to the attacker's domain. And recovery may trust an
unverified email or phone change, redirecting the reset to the attacker. The audit treats recovery as an
authentication path with the same rigor as login. You audit this by walking the flow and testing each trust it
places.

## When to use

- An application offers password reset or account recovery as a way to regain access to an account.
- Reset tokens may be weak, long-lived, reusable, or not bound to the requesting account.
- Recovery may verify a weaker factor than login, bypass multi-factor, or leak the reset link off-domain.

## Scope check

Test recovery flows only against accounts and applications you own or are authorized to assess, on
non-production accounts. Exercising reset attempts real credential changes, so use test accounts you control
and never reset or take over an account that is not yours. If you can't name the authorization, stop.

## The loop

1. **Establish the intended recovery assurance first.** Name what proof recovery should require and that it must
   be at least as strong as login, including any multi-factor requirement. This is the false-positive killer: a
   recovery flow with a high-entropy, single-use, short-lived, account-bound token, that enforces the same
   factors as login and builds the link from a server-fixed host, is correct. Name the intended assurance, then
   test each element.

2. **Assess the reset token strength and lifecycle.** Examine the token: its entropy (guessable or
   unpredictable), lifetime (short or long-lived), reuse (single-use or replayable), and invalidation (consumed
   on use and on a new request). A token an attacker can guess, reuse, or use long after issuance is a
   takeover primitive regardless of the rest of the flow.

3. **Check token-to-account binding.** Confirm the token is bound to the exact account it was issued for and
   cannot be used to reset a different account by changing an identifier in the request. A reset step that takes
   the account from a parameter while validating a token issued for another account lets an attacker reset
   someone else's credential with their own valid token.

4. **Check factor strength and multi-factor bypass.** Compare the assurance recovery requires to login. If
   recovery verifies a weaker factor, or completes without the multi-factor step login enforces, then recovery
   is the way to bypass multi-factor entirely. Confirm recovery does not lower the bar below login and re-checks
   the second factor where login requires it.

5. **Check reset-link host and destination trust.** Determine whether the reset link's host or base URL is built
   from a request header (such as the host header) an attacker can set, which sends the emailed token to the
   attacker's domain. Also confirm recovery does not trust an unverified email or phone change to redirect where
   the reset is delivered. The link and its destination must come from server-fixed configuration, not attacker
   input.

6. **Confirm and record.** Confirm by taking over a test account you control through a real weakness, a guessed
   or replayed token, a token rebound to another test account, a multi-factor bypass via recovery, or a reset
   link leaked via a spoofed host, without touching any account that is not yours. Kill the lead if the token is
   high-entropy, single-use, short-lived, and account-bound, recovery enforces login-equivalent factors, and the
   reset link and destination come from server-fixed configuration. Record the recovery request, the
   account-takeover sink, and the weak token, bypassed factor, or leaked reset link.

## Where recovery trust leaks

- **A weak or long-lived token is guessable or replayable.** Low entropy, long lifetime, or reuse turns the
  reset token into a takeover primitive.
- **An unbound token resets the wrong account.** A token not bound to its account, plus an account identifier in
  the request, lets an attacker reset someone else's credential.
- **Recovery bypasses multi-factor.** A recovery path that verifies a weaker factor than login, or skips the
  second factor, is the way around multi-factor.
- **A spoofable link host leaks the token.** Building the reset link from an attacker-controllable header sends
  the emailed token to the attacker's domain.
- **Trusting an unverified contact change redirects the reset.** Honoring an unverified email or phone update
  sends recovery to the attacker.

## Worked example (a confirm and a kill)

> **Confirm.** A password-reset email builds its link host from the request host header. Requesting a reset for a
> test account the tester controls while supplying an attacker-chosen host header produces a reset link pointing
> at the attacker's domain, so when followed the valid token is delivered to the attacker, who completes the
> reset. The same endpoint's token is also long-lived and reusable. **Confirmed** account takeover via
> host-header reset-link poisoning with a weak token, `critical`, remediation = build the reset link from
> server-fixed configuration only, issue a high-entropy single-use token that expires quickly and is invalidated
> on use, and bind the token to the requesting account.
>
> **Kill.** The reset token is high-entropy, single-use, short-lived, invalidated on use and on any new request,
> and bound to the exact account it was issued for; recovery enforces the same factors as login including
> multi-factor; and the reset link host and delivery destination come from server-fixed configuration, ignoring
> request headers and any unverified contact change. A guessed, replayed, rebound, or off-domain reset attempt
> fails. **Killed**, `kill_reason` = "strong single-use account-bound token with short expiry,
> login-equivalent factors enforced, and server-fixed link and destination; no recovery request takes over an
> account."

## Rationalizations to reject

- *"The token is random."* → Confirm its entropy, single use, short expiry, and invalidation; a random-looking
  token that is long-lived or reusable is still a takeover primitive.
- *"Reset requires the email."* → If recovery skips the multi-factor step login enforces, it is the multi-factor
  bypass; recovery must be at least as strong as login.
- *"The link points to our site."* → Confirm the host is server-fixed, not from a request header; a spoofable
  host sends the token to the attacker.
- *"The token identifies the account."* → Confirm it is bound so it cannot reset a different account named in the
  request; an unbound token plus an identifier is a cross-account reset.
- *"They updated their email."* → Confirm the change was verified before recovery trusts it; an unverified
  contact change redirects the reset to the attacker.

## Executing this in practice

You need the reset token's entropy, lifetime, reuse, and invalidation, its binding to the account, the factors
recovery requires versus login, and how the reset link host and delivery destination are determined. Walk the
flow on test accounts you control, testing token guessing and replay, cross-account rebinding, multi-factor
bypass, and host-header link poisoning. Reading the recovery code shows the intended assurance; taking over a
test account through a weakness shows whether it holds.

## Related

- `auditing-session-lifecycle-and-fixation` - recovery grants a session; its lifecycle and fixation properties
  are the next boundary after the reset completes.
- `auditing-saml-and-oidc-federation-trust` - federated login and account recovery are two doors to the same
  account; a weak recovery undercuts strong federation.
- `hunting-broken-object-level-authorization` - the cross-account reset here is an object-authorization failure
  on the reset endpoint; the two skills meet at the account identifier.
- `auditing-webauthn-and-passkey-flows` - passkeys strengthen the front door; confirm recovery does not offer a
  weaker path around them.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the attacker-driven recovery request, sink = the
  account it takes over, evidence = the weak token, bypassed factor, or leaked reset link.
