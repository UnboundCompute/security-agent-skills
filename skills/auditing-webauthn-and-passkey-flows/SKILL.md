---
name: auditing-webauthn-and-passkey-flows
description: >-
  Audit the server side of passwordless authentication for ceremony-verification bugs that let an
  attacker-shaped response become an authenticated session. Covers a registration or authentication
  ceremony whose challenge is not bound to a server-issued single-use value, an origin or
  relying-party identifier that is never checked or checked by substring, a user-verification flag
  ignored when policy required it, attestation accepted when it was required, a signature counter
  regression that hides a cloned authenticator, and the highest-severity case where a cryptographically
  valid assertion seats a session for a user other than the one the credential is bound to. Use when
  reviewing code that verifies a registration or authentication ceremony and establishes identity from
  the result. The attacker-shaped ceremony response is the source, the authenticated session is the
  sink, and a missing required check between them is the bug.
license: MIT
---

# Auditing passkey and WebAuthn ceremonies: when a verified signature seats the wrong user

A passkey ceremony proves that some authenticator signed a challenge. It does not, by itself, prove
who is signing in. Every WebAuthn bug worth finding lives in the gap between "this signature is
valid" and "therefore user U is logged in," and that gap is closed only by a specific list of
server-side checks. When one of those checks is missing, homegrown, or bypassable, attacker-shaped
bytes from the ceremony response flow straight into an authenticated session. You find these by
reading the verification path and asking, for each required check, whether it runs and whether the
value it checks is the one the attacker cannot forge.

## When to use

- Code verifies a WebAuthn registration or authentication response and then creates a credential or a session.
- The application implements the ceremony itself, or calls a library but supplies the policy and expected values.
- You want to know whether a valid signature can be replayed, phished, or bound to the wrong account.

## Scope check

Test passkey flows only on systems you own or are authorized to assess, with test accounts and test
authenticators. A ceremony bypass is an authentication bypass, so treat any confirmed finding as
account-takeover-grade and coordinate disclosure. If you can't name the authorization, stop.

## The loop

1. **Map the verification path and whether a library owns the ceremony.** Find the registration and
   authentication endpoints and the function that decides success. Determine whether the ceremony is
   delegated to a conformant verification library or hand-rolled. If a library owns it, the bug moves
   from "missing check" to "wrong arguments": read what expected-challenge, expected-origin, expected
   relying-party identifier, and required-user-verification values are passed in. A library called
   with defaults verifies only what you gave it.

2. **Check challenge binding.** The challenge in the response must equal a value the server generated,
   stored against this session, has not used before, and has not let expire. The bug is a challenge
   compared against a client-sent field, against a constant, or not at all, or a challenge that is
   never invalidated after use so the same response replays. If nothing reads a server-side challenge
   store before the compare, the ceremony is replayable.

3. **Check origin and relying-party identifier.** The response carries the origin the ceremony ran on
   and a hash of the relying-party identifier; both must match an exact allowlist. A substring, suffix,
   or pattern match is the classic bug: a suffix test for one domain also accepts a lookalike that ends
   with it. Missing origin checks turn a phishing relay into a working login.

4. **Check the user-verification flag and attestation when required.** If policy asked for user
   verification, the code must read the user-verified bit, not just user-presence; ignoring it silently
   downgrades two factors to one. If policy required attestation, the attestation statement and its
   trust path must actually be verified; "required in config, accepted unchecked in code" is the bug. A
   ceremony that legitimately requires neither is a design choice, not a finding.

5. **Check credential-to-user binding and the signature counter.** This is the highest-severity step.
   The credential must be looked up by its identifier and confirmed to belong to the user being seated,
   or the seated user must be derived from the credential's bound handle, never from a separate
   client-controlled field. A valid signature over credential A that logs you in as user B is a total
   authentication bypass. Then, if the stored counter is non-zero, a new count that is not strictly
   greater signals a cloned authenticator and must be rejected; a zero counter is spec-permitted and is
   not a regression.

6. **Confirm and record.** Confirm by exercising the live verification path with a test authenticator
   and a modified response that removes or alters the value under test, showing the ceremony still
   succeeds. Kill the lead if a conformant library performs the ceremony and is called with a
   server-stored single-use challenge, an exact expected origin and relying-party identifier, the
   required user-verification policy, and the session is seated strictly from the credential-bound user.
   Record the endpoint, the missing or bypassable check, and the seated identity.

## Where passkey trust leaks

- **The credential proves possession, not identity.** The server, not the signature, decides which
  account the possession maps to. Bind the session to the credential's user or nowhere.
- **The challenge is the only thing that makes a replay fail.** If it is not server-issued, single-use,
  and bound to the session, a captured response works twice.
- **Origin is the phishing boundary.** An exact-match origin check is what stops a relayed ceremony; a
  suffix or contains check quietly reopens it.
- **The user-verification flag is the second factor.** Passwordless is only multi-factor if the code
  actually reads that bit; unread, it is single-factor with extra steps.
- **A library hides the bug in its arguments.** A conformant verifier called without the expected
  challenge, origin, or relying-party identifier checks nothing you did not hand it.

## Worked example (a confirm and a kill)

> **Confirm.** The authentication endpoint verifies the assertion signature against the stored public
> key and returns success, then seats the session from a `userId` field in the same request body. A test
> authenticator's own valid assertion, submitted with a victim's `userId`, logs in as the victim. The
> credential was never checked to belong to the seated user. **Confirmed** credential-to-user binding
> confusion, `critical`, remediation = derive the session identity from the user bound to the matched
> credential handle and never from a client-supplied field.
>
> **Kill.** The same endpoint passes the response to a conformant verification library with the
> server-stored single-use challenge, an exact expected origin, the expected relying-party identifier,
> and a user-verification-required policy, then seats the session only from the credential record's owner
> on success. Altering the origin or reusing a challenge makes the library reject the ceremony. **Killed**,
> `kill_reason` = "ceremony delegated to a conformant verifier with server-stored single-use challenge,
> exact origin and relying-party checks, required user verification, and identity seated from the bound
> credential; tampered responses are rejected."

## Rationalizations to reject

- *"The signature verified, so the user is authenticated."* -> Verified against which credential, and is
  that credential bound to the user you are about to seat? A valid signature for the wrong account is a bypass.
- *"We check the origin."* -> Exact match or substring? A suffix test for your domain also accepts a
  lookalike that ends with it. Only an exact allowlist is a check.
- *"It is passwordless, so it is strong."* -> Only if the code reads the user-verified bit. Unread, a
  passkey is a single factor no matter how it was provisioned.
- *"A counter of zero means our check is broken."* -> Zero is permitted by the spec; the regression is a
  decrease from a non-zero stored value, not the presence of a zero.
- *"The library handles all of this."* -> Only for the values you pass it. A verifier called with defaults
  and no expected origin or challenge verifies a signature and nothing else.

## Executing this in practice

You need the bodies of the registration and authentication verification functions, the policy object
that declares user-verification and attestation requirements, the code that stores and looks up
challenges, the credential-store lookup, and the exact line where the session user is set. For each
required check, decide whether it runs on the live path and whether the value it tests is
attacker-forgeable. Reading the verification function and its callees tells you which checks exist;
replaying a tampered response against a test account tells you which ones actually hold.

## Related

- `auditing-saml-and-oidc-flows` - the federation sibling; both hunt an authentication ceremony that
  establishes identity from attacker-reachable input, and both fail when a binding check is skipped.
- `auditing-device-code-and-pkce-flows` - another authentication-flow audit where the bug is a proof
  that binds a request to its initiator being unenforced.
- `auditing-randomness-and-nonce-quality` - a predictable challenge is a passkey bug rooted in weak
  randomness; that skill covers the entropy of the values this one assumes are unguessable.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the attacker-shaped ceremony response, sink =
  the authenticated session, evidence = the tampered response that still succeeds and the seated identity.
