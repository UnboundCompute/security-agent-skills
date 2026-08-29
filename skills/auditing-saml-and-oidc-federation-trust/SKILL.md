---
name: auditing-saml-and-oidc-federation-trust
description: >-
  Audit federated single sign-on for assertions a relying party should not trust: a SAML response whose
  signature is not verified over the right element so a wrapped or altered assertion passes, an OIDC ID token
  whose issuer, audience, or nonce is unchecked, a relying party that accepts an assertion for any user
  because the subject or email is trusted without binding, and a federation that honors an identity provider or
  signing key it should not. Covers SAML and OpenID Connect where a relying party consumes assertions or ID
  tokens minted by an identity provider to authenticate users. Use when login trust crosses from an identity
  provider to a relying party and assertion validation is the boundary. The forged or misbound assertion is
  the source, the authenticated session it grants is the sink, and the missing signature, issuer, audience, or
  binding check that accepts it is the bug.
license: MIT
---

# Auditing SAML and OIDC federation trust: the assertion is only as good as its verification

Federated login moves the authentication decision to an identity provider and hands the relying party a signed
assertion, a SAML response or an OIDC ID token, that says who the user is. The relying party's entire security
then rests on verifying that assertion correctly, and the ways to verify it incorrectly are well worn. SAML
signature wrapping slips an unsigned, attacker-authored assertion alongside a signed one so the application
reads the wrong element; a signature checked over the wrong scope, or not at all, accepts a tampered response.
An OIDC ID token whose issuer, audience, or nonce goes unchecked can come from the wrong provider, be minted
for a different application, or be replayed. And a relying party that trusts a subject or email in the
assertion without binding it to the verified issuer will authenticate any user an attacker names. The audit is
not whether federation is configured but whether a forged or misbound assertion is refused. You audit this by
testing each validation the relying party must perform on the assertion.

## When to use

- A relying party consumes SAML assertions or OIDC ID tokens from an identity provider to authenticate users.
- Signature, issuer, audience, or nonce validation on the assertion may be missing or scoped wrong.
- The relying party may trust a subject, email, or attribute without binding it to the verified issuer.

## Scope check

Test federation trust only against relying parties and identity providers you own or are authorized to assess,
on non-production accounts. Forging or replaying assertions attempts real authentication, so use non-production
identities and never authenticate as a real user. If you can't name the authorization, stop.

## The loop

1. **Establish the intended trust first.** Name which identity provider and signing key the relying party
   should trust, the expected issuer and audience, and how the assertion's subject maps to a local user. This
   is the false-positive killer: a relying party that verifies the signature over the correct element against
   the expected key, checks issuer, audience, and nonce, and binds the subject to the verified issuer is
   correct. Name the intended trust, then test each check.

2. **Check SAML signature verification and wrapping.** For SAML, confirm the signature is verified over the
   exact element whose contents are consumed, and that the application reads the signed assertion, not an
   unsigned one an attacker added. Test signature wrapping: a response containing a valid signed assertion plus
   an injected unsigned assertion the application might read instead. Confirm a tampered or wrapped response is
   rejected.

3. **Check OIDC issuer, audience, and nonce.** For OIDC, confirm the ID token's issuer matches the expected
   provider, its audience matches this relying party, its signature verifies against the provider's current
   key, and the nonce ties it to this login request. A token from the wrong issuer, minted for a different
   audience, or replayed without a nonce check authenticates something the relying party did not request.

4. **Check subject binding and attribute trust.** Confirm the relying party maps the user from the assertion's
   subject as asserted by the verified issuer, not from an email or attribute it trusts without binding. Test
   whether an assertion naming a different user's email or identifier authenticates as that user. An identity
   taken from an unbound attribute lets an attacker assert any user.

5. **Check the trusted providers and keys.** Confirm the relying party honors only the specific identity
   provider and signing keys it should, rejects assertions from unknown issuers or keys, and rotates and pins
   keys correctly. A relying party that trusts any provider in a shared federation, or accepts a key it should
   not, can be given a validly signed assertion from the wrong source.

6. **Confirm and record.** Confirm by getting a forged, wrapped, wrong-issuer, wrong-audience, replayed, or
   misbound assertion accepted as a login, on non-production accounts and without authenticating as a real
   user. Kill the lead if the relying party verifies the signature over the right element against the expected
   key, checks issuer, audience, and nonce, binds the subject to the verified issuer, and trusts only the
   intended providers and keys. Record the forged or misbound assertion, the authenticated-session sink, and
   the missing signature, issuer, audience, or binding check.

## Where federation trust leaks

- **Signature wrapping reads the wrong element.** An unsigned assertion injected beside a signed one is
  consumed if the application does not verify the signature over exactly what it reads.
- **An unchecked issuer or audience accepts the wrong token.** An OIDC ID token from the wrong provider or
  minted for a different application passes when issuer and audience are not verified.
- **A missing nonce enables replay.** An ID token not bound to the login request by a nonce can be replayed
  into a session.
- **Unbound subject or email asserts any user.** Taking the identity from an attribute the relying party trusts
  without binding to the verified issuer lets an attacker name any user.
- **An over-broad federation trusts the wrong source.** Honoring any provider or key in a shared federation
  accepts a validly signed assertion from a source it did not intend.

## Worked example (a confirm and a kill)

> **Confirm.** A SAML relying party verifies that the response contains a valid signature but reads the subject
> from an assertion element without confirming the signature covers that element. A crafted response carries a
> legitimately signed assertion plus an injected unsigned assertion naming a different user, and the application
> reads the injected one, authenticating as the attacker-named user. **Confirmed** SAML signature-wrapping
> authentication bypass, `critical`, remediation = verify the signature over exactly the element whose contents
> are consumed, reject responses with multiple or unsigned assertions, and bind the authenticated user to the
> signed subject only.
>
> **Kill.** The relying party verifies the signature over the exact consumed element against the expected
> provider key, rejects responses containing unsigned or extra assertions, checks issuer and audience (and nonce
> for OIDC), maps the user from the signed subject bound to the verified issuer, and trusts only the one
> configured provider and its pinned keys. A wrapped, wrong-issuer, or misbound assertion is refused. **Killed**,
> `kill_reason` = "signature verified over the consumed element, issuer/audience/nonce checked, subject bound to
> the verified issuer, and only the intended provider trusted; no forged or misbound assertion authenticates."

## Rationalizations to reject

- *"The response is signed."* → Confirm the signature covers exactly the element you read and there is no extra
  unsigned assertion; a signed response can still carry a wrapped forgery.
- *"The token came from our provider."* → Verify the issuer and audience on the token itself; without them a
  token from the wrong provider or for a different app passes.
- *"We match the user by email."* → An email or attribute not bound to the verified issuer lets an attacker
  assert any user; map the user from the signed subject.
- *"Nonce is optional."* → Without a nonce an ID token can be replayed into a session; bind the token to the
  login request.
- *"We trust the federation."* → Confirm you trust only the specific provider and keys intended; a broad
  federation accepts a valid assertion from a source you did not mean to honor.

## Executing this in practice

You need the expected provider, issuer, audience, and signing keys, exactly what the relying party verifies on
the assertion and over which element, how the subject maps to a local user, and the trusted-provider set. For
each, test a wrapped or tampered SAML response, a wrong-issuer or wrong-audience or replayed OIDC token, and a
misbound subject. Reading the assertion-validation code shows the intended trust; a forged assertion accepted
as login shows whether it holds.

## Related

- `auditing-oauth-token-audience-and-scope-trust` - the access-token companion; audience and issuer confusion
  are the same failure at the authorization-token layer.
- `auditing-jwt-verification-and-key-trust` - OIDC ID tokens are JWTs, so the algorithm and key-trust checks
  there are prerequisites for the issuer and audience checks here.
- `auditing-session-lifecycle-and-fixation` - once federation grants a session, its lifecycle and fixation
  properties are the next boundary this pairs with.
- `auditing-account-recovery-and-reset-trust` - federated identity and account recovery are two routes to the
  same account; audit both so one does not undercut the other.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the forged or misbound assertion, sink = the
  authenticated session it grants, evidence = the missing signature, issuer, audience, or binding check.
