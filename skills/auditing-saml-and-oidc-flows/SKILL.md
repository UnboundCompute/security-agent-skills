---
name: auditing-saml-and-oidc-flows
description: >-
  Audit federated single sign-on for the flaws that let an attacker forge or replay
  an identity: signature wrapping and signature stripping on signed assertions,
  unsigned or unverified tokens accepted, redirect_uri and audience manipulation,
  missing state and nonce allowing replay and cross-site request forgery, and
  identity confusion where one provider's assertion is honored for another account.
  Use when reviewing a SAML or OIDC integration, an identity-provider connection, or
  any login that trusts an external assertion. The verification step is the target.
license: MIT
---

# Auditing SAML and OIDC flows: the whole trust rests on one verification

Federated login moves the trust boundary to a signed assertion from an identity
provider, and the entire security rests on the relying party verifying that assertion
exactly. The classic flaws are all failures of that verification: a signature checked
over the wrong element, a token accepted without a signature, an audience or redirect
target not pinned, a flow with no anti-replay value. Each lets an attacker present an
identity that is not theirs.

## When to use

- You are reviewing a SAML or OIDC integration or an identity-provider connection.
- A login trusts an externally-issued, signed assertion or token.
- You are assessing account-linking and how an external identity maps to a local one.

## Scope check

Audit login flows you own or are authorized to test, with test accounts and a test
identity provider. Do not forge assertions against systems you do not control. If you
can't name the authorization, stop.

## The loop

1. **Identify the trusted assertion and where it is verified.** Find the token or
   assertion the relying party trusts (a signed SAML assertion, an OIDC id token) and
   the exact code that validates it. Everything downstream trusts whatever that check
   accepts, so the check is the audit target.

2. **Test signature presence and coverage.** Is every accepted assertion actually
   signed, and is the signature verified over the exact element whose contents are
   used? Try stripping the signature (does an unsigned assertion get accepted), and
   try signature wrapping (add a second, attacker-controlled assertion the app reads
   while the signature still validates over the original). A signature that does not
   cover the data the app consumes is no signature.

3. **Verify audience, issuer, and expiry.** Does the relying party confirm the
   assertion was issued for it (audience or client id), by the expected issuer, and
   still valid (not expired, not before)? An assertion minted for another service, or
   by an unexpected issuer, must be rejected. Missing audience pinning lets a token
   from elsewhere be replayed here.

4. **Test redirect_uri and response routing.** For the redirect-based flows, is
   redirect_uri matched exactly against a strict allowlist, or can it be widened
   (extra path, open redirect, wildcard) to divert the code or token to the attacker?
   Loose redirect matching is how the credential leaves the intended path.

5. **Test anti-replay and binding.** Is there a state value bound to the user's
   session (cross-site request forgery and mix-up defense) and a nonce bound into the
   id token (replay defense), both verified on return? Absence lets an assertion or
   code be replayed or injected into another user's session. Confirm an assertion
   cannot be replayed twice.

6. **Test identity mapping.** How is the asserted subject mapped to a local account:
   by a stable, provider-scoped identifier, or by a mutable, cross-provider field like
   email that another identity source could also assert? Mapping on an unverified or
   non-unique claim is account takeover by identity confusion. Confirm or kill each
   and record.

## Where federation breaks

- **The signature must cover what you read.** Wrapping and stripping both exploit a
  gap between what is signed and what is consumed.
- **An assertion is a bearer credential.** If audience and replay are not enforced,
  whoever holds it is whoever it names.
- **redirect_uri is a capability.** Loose matching hands the code or token to the
  attacker's endpoint.
- **Email is not an identity.** Map on the provider's stable subject, not a claim
  other providers can also set.

## Worked example (a confirm and a kill)

> **Confirm.** A relying party verifies the SAML signature but reads the subject from
> the first matching element, while the signature covers a second. An attacker wraps a
> forged subject element the app reads and leaves the signed one intact for
> verification. The forged identity is accepted. **Confirmed** signature wrapping,
> `critical`, remediation = verify the signature over the exact consumed element,
> reject multiple assertions, and use a hardened parser.
>
> **Kill.** An OIDC relying party accepts only signed id tokens, verifies signature,
> issuer, audience, and expiry, matches redirect_uri exactly against an allowlist,
> binds and checks state and nonce, and maps identity on the provider-scoped subject.
> A stripped, wrapped, or replayed token is rejected at verification. **Killed**,
> `kill_reason` = "signature, issuer, audience, expiry, exact redirect_uri, and
> state/nonce all verified; identity mapped on stable subject."

## Rationalizations to reject

- *"The assertion is signed."* → Signed over what? Confirm coverage of the element you
  actually read.
- *"We check the signature, that's enough."* → Not without audience, expiry, and
  anti-replay. A valid signature on a token minted elsewhere still verifies.
- *"redirect_uri starts with our domain."* → Prefix and substring matching are
  bypassable. Match the full URI against an allowlist.
- *"We link accounts by email."* → Another provider can assert that email. Map on the
  issuer-scoped subject id.

## Executing this in practice

You need the exact assertion the relying party trusts, the verification code, and the
ability to replay a login while modifying the assertion, the redirect_uri, and the
state and nonce. An intercepting proxy plus a test identity provider works; the
verification-coverage analysis and the anti-replay checks are the method.

## Related

- `auditing-guard-gaps` - the verification step is the guard; this finds where it is
  missing or partial.
- `finding-fail-open-flaws` - a verification that passes on error is a federation
  bypass.
- `writing-vuln-reports` - an identity-forgery finding to a reproducible writeup.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the attacker-shaped assertion
  or redirect, sink = the login that trusts it.
