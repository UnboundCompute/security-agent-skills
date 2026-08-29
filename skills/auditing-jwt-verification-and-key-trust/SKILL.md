---
name: auditing-jwt-verification-and-key-trust
description: >-
  Audit how a service verifies JSON Web Tokens for the classic verification bypasses: an algorithm-confusion
  attack where a token switches the signing algorithm so a public key is used as a symmetric secret or the
  algorithm is set to none, a key selected from an attacker-controllable header (a key id, a JWKS URL, or an
  embedded key) so the token names its own signer, a signature that is decoded but not actually verified, and
  claims (expiry, issuer, audience) that are parsed but not enforced. Covers services that accept and verify
  JWTs to authenticate or authorize a caller. Use when a JWT is the credential and its verification is the
  boundary. The forged or unverified token is the source, the authenticated or authorized action it grants is
  the sink, and the algorithm confusion, attacker-chosen key, or unverified claim that accepts it is the bug.
license: MIT
---

# Auditing JWT verification and key trust: the token names its own signer unless you stop it

A JSON Web Token is a signed claim, and its security is entirely in how the verifier checks the signature and
the claims. The failures are a small, famous set, and they recur because the token itself carries the
parameters an attacker wants to control. The algorithm lives in the token header, so a verifier that trusts it
can be steered into algorithm confusion, verifying an RS256 token as HS256 using the public key as the shared
secret, or accepting an algorithm of none with no signature at all. The key can be selected from the header
too, a key id, a JWKS URL, or an embedded key, so a token can name a signer the attacker controls. A verifier
that decodes the token and reads the claims but never actually checks the signature accepts anything. And a
verifier that checks the signature but not the expiry, issuer, and audience honors expired or wrong-context
tokens. The audit pins the algorithm and key on the server side and confirms the signature and claims are
enforced. You audit this by presenting manipulated tokens and confirming each is refused.

## When to use

- A service accepts JWTs and verifies them to authenticate or authorize a caller.
- The signing algorithm or key may be chosen from token-supplied fields (alg, kid, jku, embedded key).
- Claims such as expiry, issuer, and audience may be parsed but not enforced during verification.

## Scope check

Test JWT verification only against services you own or are authorized to assess, on non-production tokens and
accounts. Forging tokens attempts real authentication, so use non-production credentials and never use a forged
token to reach real data. If you can't name the authorization, stop.

## The loop

1. **Establish the intended algorithm and key first.** Name the exact algorithm the service should accept and
   the specific key or key set it should verify against, both fixed on the server side. This is the
   false-positive killer: a verifier that pins the algorithm, selects the key from server configuration or a
   pinned trusted key set, verifies the signature, and enforces expiry, issuer, and audience is correct. Name
   the intended algorithm and key, then test manipulation.

2. **Test algorithm confusion and none.** Present a token whose header switches the algorithm: an RS256 token
   re-signed as HS256 using the public key as the secret, and a token with the algorithm set to none and no
   signature. Confirm the verifier rejects any algorithm other than the one it expects and never accepts none.
   A verifier that honors the header's algorithm is steerable into using the wrong verification path.

3. **Test attacker-chosen key selection.** Check whether the key is selected from a token-supplied field: a key
   id that indexes an attacker-influenceable store or path, a JWKS URL the token names so the server fetches the
   attacker's key, or an embedded key the token carries. Present tokens that manipulate each and confirm the
   server ignores token-supplied key hints and uses only its pinned trusted keys.

4. **Confirm the signature is actually verified.** Confirm the code verifies the signature rather than merely
   decoding the token and reading claims. Present a token with a valid structure but an invalid signature and
   confirm it is rejected. A decode-without-verify path accepts any well-formed token regardless of signature.

5. **Check claim enforcement.** Confirm the verifier enforces expiry (and not-before), issuer, and audience,
   not just parses them. Present an expired token, one from the wrong issuer, and one for a different audience,
   and confirm each is refused. A signature-valid token with an unenforced claim is honored outside its
   intended time or context.

6. **Confirm and record.** Confirm by getting a manipulated token accepted, an algorithm-confused or none
   token, one signed by an attacker-named key, one with an unverified signature, or an expired or wrong-audience
   token, on non-production accounts and without reaching real data. Kill the lead if the verifier pins the
   algorithm, uses only server-side trusted keys, verifies the signature, and enforces expiry, issuer, and
   audience. Record the forged token, the authenticated or authorized action sink, and the algorithm confusion,
   attacker-chosen key, or unverified claim that accepted it.

## Where JWT verification leaks

- **Algorithm from the header enables confusion.** Trusting the token's algorithm allows RS256-to-HS256
  confusion using the public key as the secret, or an algorithm of none with no signature.
- **Key from the token names its own signer.** A key id, a JWKS URL, or an embedded key taken from the token
  lets an attacker point verification at a key they control.
- **Decode is not verify.** Reading the claims without checking the signature accepts any well-formed token.
- **Parsed claims are not enforced claims.** Expiry, issuer, and audience read but not checked honor expired or
  wrong-context tokens.
- **A valid signature is not sufficient authorization.** A correctly signed token still must match the expected
  issuer, audience, and time to authorize this action.

## Worked example (a confirm and a kill)

> **Confirm.** A service verifies RS256 tokens but selects the verification algorithm from the token header. A
> token re-signed as HS256, using the service's known public key as the HMAC secret, is accepted because the
> verifier runs the HMAC path with the public key. The forged token authenticates as an arbitrary user.
> **Confirmed** JWT algorithm-confusion authentication bypass, `critical`, remediation = pin the accepted
> algorithm to exactly RS256 on the server, reject any other algorithm and none, and never derive the
> verification key or algorithm from token-supplied fields.
>
> **Kill.** The verifier accepts only the one configured algorithm, rejects none and any mismatch, selects the
> key solely from a pinned trusted key set ignoring the token's key id, JWKS URL, and embedded key, verifies the
> signature on every token, and enforces expiry, issuer, and audience. An algorithm-confused, wrong-key,
> unsigned, expired, or wrong-audience token is refused. **Killed**, `kill_reason` = "algorithm pinned, key
> taken only from server-side trusted set, signature verified, and claims enforced; no manipulated token
> authenticates."

## Rationalizations to reject

- *"The library verifies the token."* → Confirm it pins the algorithm and ignores token-supplied keys; many
  verify paths honor the header's algorithm and key hints unless configured not to.
- *"We use RS256."* → Confirm the server rejects a token that claims HS256 or none; accepting the header's
  algorithm is exactly the confusion attack.
- *"The key id tells us which key."* → A token-supplied key id, JWKS URL, or embedded key is attacker input;
  select the key from your trusted set, never from the token.
- *"We read the claims."* → Reading is not verifying; confirm the signature is checked and expiry, issuer, and
  audience are enforced, not just parsed.
- *"The signature is valid."* → A valid signature proves origin, not context; enforce issuer, audience, and
  expiry so the token authorizes only this action now.

## Executing this in practice

You need the algorithm the service accepts, how the verification key is selected, whether the signature is
actually verified, and which claims are enforced. For each, present an algorithm-confused or none token, a
token naming an attacker-controlled key, an invalid-signature token, and expired and wrong-audience tokens.
Reading the verification code shows the intended pinning; a manipulated token accepted shows whether it holds.

## Related

- `auditing-oauth-token-audience-and-scope-trust` - the audience and scope layer above verification; a token
  must verify here before its audience and scope can be trusted there.
- `auditing-saml-and-oidc-federation-trust` - OIDC ID tokens are JWTs, so this verification is the mechanism
  underneath that skill's issuer and audience checks.
- `auditing-webhook-authenticity-and-callback-trust` - signed webhooks share the verify-the-signature-and-the-
  claims discipline; the same decode-without-verify and key-trust mistakes appear there.
- `hunting-non-human-identity-and-secret-reachability` - the signing key is a high-value secret; that skill
  traces who can reach it, which decides whether forgery is even needed.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the forged or unverified token, sink = the
  authenticated or authorized action it grants, evidence = the algorithm confusion, attacker-chosen key, or
  unverified claim that accepted it.
