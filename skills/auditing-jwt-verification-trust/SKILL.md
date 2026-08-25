---
name: auditing-jwt-verification-trust
description: >-
  Audit code that verifies a JSON Web Token for a signature or claims check that trusts token-supplied
  parameters, so an attacker can forge a token the server accepts, after the algorithm pinning and the key
  source are resolved. Covers an algorithm taken from the token header rather than pinned server-side, an
  RS256-to-HS256 key confusion where a public key is used as an HMAC secret, an accepted none algorithm or
  a verification call with signature checking off, a kid, jku, or x5u parameter sourcing a key from an
  untrusted location, audience, issuer, and expiry claims left unchecked, and an HMAC secret that is weak,
  guessable, or committed. Use when reviewing the verification call and its options in source, not the
  token-generation entropy the randomness skill owns or the OAuth flow the OIDC skill owns. A token with
  an attacker-chosen header or bytes is the source, a verification call that gates identity is the sink,
  and an unpinned algorithm or a token-sourced key is the bug.
license: MIT
---

# Auditing JWT verification trust: when a signed token is verified on the token's own terms

A JSON Web Token is only as trustworthy as the verification code, and the bug is verification that trusts
what the token itself supplies: the algorithm from the header, the key from a header-named location. If the
server accepts the token's `alg`, an attacker switches a signature-verified token to an HMAC one signed
with the public key, or to `none`; if it fetches the key from the token's `kid`, `jku`, or `x5u`, the
attacker points it at a key they control. You audit it by resolving whether the algorithm is pinned
server-side and where the verification key comes from, then checking the claims. This is a source-code
audit: the whole value is deciding whether a reported token flaw is actually reachable in this codebase,
so the false-positive killers carry the weight. Token-generation entropy belongs to the randomness skill;
the OAuth grant dance belongs to the OIDC skill.

## When to use

- You are reviewing the code path that verifies a signed token and uses its claims for an identity decision.
- You see a decode-or-verify call, an algorithms option, a key lookup by header parameter, or a claims read.
- You want to know whether an attacker can forge a token this verification call accepts.

## Scope check

Audit only systems you own or are authorized to assess, and present a crafted token only against an
endpoint in scope, a forged token that verifies grants real access. Adjudicate on the verification call
and its options. If you can't name the authorization, stop.

## The loop

1. **Resolve the algorithm pinning and the key source first.** Read the verification call and its options:
   is the accepted algorithm pinned to a fixed value server-side, or taken from the token header, and where
   does the verification key come from (a static preconfigured key or keyset, or a location named by the
   token's `kid`, `jku`, or `x5u`). These two facts decide whether the header-driven attacks have a sink,
   so settle them before flagging anything.

2. **Check the algorithm.** Look for the accepted algorithm read from the token header rather than pinned,
   enabling an RS256-to-HS256 key confusion, where the verifier uses the public key as an HMAC secret and
   the attacker signs with that public key, and an accepted `none` algorithm or a call with signature
   verification disabled.

3. **Check the key source.** Look for the verification key resolved from a token-supplied `kid` (a path
   traversal or an injection into the key lookup), or fetched from a token-supplied `jku` or `x5u` URL (a
   server-side request to an attacker location), so the token names the key that verifies it.

4. **Check the claims.** Look for audience, issuer, expiry, and not-before claims not enforced, so a token
   minted for another audience or an expired token is accepted, and for a subject or role claim trusted
   without binding to the verified context.

5. **Check the secret.** Look for an HMAC secret that is weak, guessable, a default, or committed to the
   repository, so the attacker signs a valid token directly without any of the above.

6. **Confirm and record.** Confirm by presenting a forged token (algorithm-switched, `none`, signed with an
   attacker-named key, or signed with a recovered secret) and showing the verification accepts it and the
   identity decision follows. Kill the lead if the library pins the accepted algorithms so the header `alg`
   is ignored, defeating the confusion and none attacks, if verification happens at an API gateway or
   middleware before the handler so a handler reading claims unverified is downstream of a real check, if
   the key is resolved from a static preconfigured keyset rather than the token header so `kid`, `jku`, and
   `x5u` reach no sink, if the library version rejects `none` by default when a key or required algorithm is
   supplied, if the claims are enforced by the verify wrapper's options rather than a manual check (absence
   of the manual check is not a miss), or if the secret comes from a secrets manager with real entropy.
   Record the verification call, the token-supplied parameter it trusts, and the identity decision it gates.

## Where verification trust leaks

- **The algorithm must be pinned server-side.** Reading `alg` from the header lets the attacker choose the
  scheme (HMAC-with-the-public-key, or none); a fixed accepted-algorithms list ignores the header and kills
  both.
- **The key must not come from the token.** Sourcing the verification key from `kid`, `jku`, or `x5u` lets
  the token name the key that verifies it; a static preconfigured keyset gives those parameters no sink.
- **Verification may live upstream.** A handler that reads claims without verifying them is fine when a
  gateway or middleware verified first; find where the real check is before calling the handler the bug.
- **Claims enforced by options are still enforced.** Audience, issuer, and expiry checked by the verify
  wrapper count; the absence of a separate manual check is not a missing check.
- **A weak or committed secret is the shortcut.** If the HMAC secret is guessable or in the repository, the
  attacker signs a valid token directly and none of the header tricks are needed.

## Worked example (a confirm and a kill)

> **Confirm.** A handler decodes the token with the algorithm taken from the header and the key looked up
> from the token's `kid`, then trusts the subject claim for authorization. An attacker crafts a token with
> `alg` set to HMAC and a `kid` selecting a known public key, signs it with that public key, and the
> verification accepts it. **Confirmed** token forgery through a header-chosen algorithm and a token-sourced
> key gating identity, `critical`, remediation = pin the accepted algorithm, resolve the key from a static
> keyset only, and enforce the audience, issuer, and expiry.
>
> **Kill.** The same system verifies at the gateway with the accepted algorithms pinned to RS256, the
> public key loaded from a static configured keyset (the token `kid` only selects among preconfigured keys,
> never a location), and the audience, issuer, and expiry enforced by the verify options. The handler reads
> the subject downstream of that check. A crafted `alg`, `none`, or `jku` is rejected. **Killed**,
> `kill_reason` = "the algorithm is pinned and the key is from a static keyset, so alg-confusion, none, and
> kid/jku injection have no sink; verification happens upstream in the gateway."

## Rationalizations to reject

- *"The library verifies the signature."* -> Against which algorithm and key? If it reads `alg` from the
  header or the key from `kid`/`jku`, the token chooses its own verification; pin the algorithm and the key.
- *"The handler does not check claims."* -> Does a gateway or middleware verify first? Find the real check
  before flagging the handler; if none exists, the missing claims check is the bug.
- *"We support key rotation via kid."* -> Selecting among a preconfigured keyset by `kid` is fine; resolving
  a key from a token-named location is not. Confirm the key is never sourced from the token.
- *"none is only for internal tokens."* -> An accepted `none` on any path an attacker reaches is a forgery;
  require a signed algorithm and reject none.
- *"The secret is strong."* -> Is it from a secrets manager, or committed and guessable? A weak or committed
  secret lets the attacker sign a valid token directly.

## Executing this in practice

You need the verification call and its options (the accepted algorithms, the key source), where
verification happens (handler, middleware, or gateway), how the key is resolved and whether the token can
influence it, the claims that are enforced and where, and the provenance and entropy of any HMAC secret.
For each verified token, decide whether an attacker can forge one the call accepts. Reading the
verification options tells you what is pinned; presenting a crafted token tells you what is accepted.

## Related

- `auditing-saml-and-oidc-flows` - owns the OAuth and OIDC grant flow (the `state`, `nonce`, and redirect
  handling); this skill owns the signature, algorithm, key-source, and claims mechanics of verifying a
  presented token.
- `auditing-randomness-and-nonce-quality` - owns the entropy of a generated token or nonce; this skill owns
  whether a presented signed token is verified correctly, not how it was minted.
- `auditing-grpc-service-authorization` - a sibling where a token or metadata gates a call; a forged token
  accepted here is the credential that defeats the authorization checked there.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = a token with an attacker-chosen header or bytes,
  sink = a verification call that gates identity, evidence = the unpinned algorithm or token-sourced key and
  the accepted forged token.
