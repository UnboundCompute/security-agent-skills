---
name: auditing-oauth-token-audience-and-scope-trust
description: >-
  Audit how a resource server trusts OAuth access tokens for confusion it should reject: a token minted for one
  audience accepted by a different service, a scope treated as coarser or finer than it is so a token reaches
  an operation it was not granted, a token-issuer or authorization-server mix-up where a token from one issuer
  is honored by a party that trusts another, and a resource server that validates the signature but not the
  audience, issuer, or scope. Covers OAuth and bearer-token architectures where an access token authorizes a
  call between a client, an authorization server, and one or more resource servers. Use when a resource server
  accepts bearer tokens and the audience, issuer, and scope checks are the boundary. The token presented to the
  wrong audience or beyond its scope is the source, the resource operation it reaches is the sink, and the
  missing audience, issuer, or scope validation is the bug.
license: MIT
---

# Auditing OAuth token audience and scope trust: a token is only good for what it was issued for

A bearer access token is a claim with a precise domain: it was issued by a particular authorization server,
for a particular audience, carrying a particular scope, and it authorizes only what those three agree on. A
resource server that checks the signature but not the audience, issuer, and scope is trusting a token far
beyond what it was minted for. The confusions are specific and common. A token issued for service A is
presented to service B, and if B does not verify the audience it accepts A's token as its own. A scope is
treated as coarser than it is, or a missing-scope check lets any valid token reach a privileged operation. A
token from a different authorization server is honored by a party that only meant to trust one issuer. Each is
a token used outside its domain. The audit checks that every token is validated against the audience, issuer,
and scope the operation requires, not merely that it is a well-formed, signed token. You audit this by testing
whether a token good for one thing is accepted for another.

## When to use

- A resource server accepts OAuth or bearer access tokens to authorize calls.
- Multiple services or audiences accept tokens, and the audience or issuer may not be verified per service.
- Scopes gate operations, and the resource server may not check the specific scope each operation requires.

## Scope check

Test token trust only against services you own or are authorized to assess, on non-production tokens and
accounts. Presenting tokens across audiences exercises real authorization, so use non-production credentials
and never use a confused token to reach real data. If you can't name the authorization, stop.

## The loop

1. **Establish each operation's required token domain first.** Name, per operation, the audience, issuer, and
   scope a token must have to be authorized: this service as the audience, the trusted authorization server as
   the issuer, the specific scope for the action. This is the false-positive killer: a resource server that
   verifies all three on every call authorizes exactly the tokens it should. Name the required domain, then
   test what the server actually accepts.

2. **Check audience validation.** Determine whether the resource server verifies that the token's audience is
   itself. Present a token minted for a different service or audience and confirm it is rejected. A server that
   accepts a token whose audience is another service is honoring credentials issued for someone else, which is
   the core audience-confusion bug.

3. **Check issuer validation and mix-up.** Confirm the resource server verifies the token issuer against the
   specific authorization server it trusts, and that in a multi-issuer or federated setup a token from one
   issuer is not honored where another was intended. A party that trusts issuer X but validates only that the
   token is signed by some known key can be given a token from issuer Y.

4. **Check scope enforcement per operation.** For each operation, confirm the server checks the exact scope the
   action requires rather than accepting any valid token or treating a broad scope as covering everything. Test
   whether a token with a read scope reaches a write operation, or a token with an unrelated scope reaches a
   privileged one. Missing or coarse scope checks let a token exceed its grant.

5. **Check token binding and reuse.** Where applicable, confirm the token cannot be replayed to a different
   audience or reused past its intended context: audience restriction, sender constraint, and expiry are
   honored. A token that is valid, signed, and unexpired but usable anywhere is a bearer secret with no domain,
   so the binding checks are part of the boundary.

6. **Confirm and record.** Confirm by presenting a token outside its domain, a cross-audience token accepted by
   the wrong service, a wrong-issuer token honored, or an insufficient-scope token reaching a privileged
   operation, on non-production accounts and without touching real data. Kill the lead if the resource server
   verifies audience, issuer, and per-operation scope on every call and honors token binding and expiry. Record
   the misused token, the resource-operation sink, and the missing audience, issuer, or scope validation.

## Where token trust leaks

- **Unvalidated audience honors another's token.** A resource server that does not check the audience accepts a
  token minted for a different service as its own.
- **Unvalidated issuer enables mix-up.** Trusting any known signer rather than the specific authorization server
  lets a token from the wrong issuer through.
- **Missing scope checks exceed the grant.** Accepting any valid token, or treating a broad scope as universal,
  lets a token reach an operation it was not granted.
- **A signed token is not an authorized token.** Signature validity proves origin, not that the token is for
  this audience, this issuer, and this scope.
- **An unbound token is a portable secret.** A token with no audience restriction, sender constraint, or honored
  expiry can be replayed wherever a bearer token is accepted.

## Worked example (a confirm and a kill)

> **Confirm.** Two internal services accept tokens from the same authorization server, and service B validates
> the token signature and expiry but not the audience. A token minted for service A, obtained legitimately by a
> client, is presented to service B and accepted, letting the client reach service B operations it was never
> granted. **Confirmed** audience confusion across resource servers, `high`, remediation = validate the audience
> on every resource server so each accepts only tokens minted for itself, verify the issuer against the specific
> authorization server, and check the exact scope each operation requires.
>
> **Kill.** Each resource server verifies that the token audience is itself, that the issuer is the specific
> authorization server it trusts, and that the token carries the exact scope the requested operation requires,
> and it honors audience restriction and expiry. A token minted for another service, from another issuer, or
> lacking the operation's scope is rejected. **Killed**, `kill_reason` = "audience, issuer, and per-operation
> scope verified on every call with honored binding and expiry; a token is accepted only for exactly what it was
> issued for."

## Rationalizations to reject

- *"The token is signed and valid."* → Signature validity is origin, not domain; confirm the audience, issuer,
  and scope match this operation, or a valid token is accepted outside its grant.
- *"Both services use the same auth server."* → Same issuer is not same audience; without an audience check
  service A's token works on service B.
- *"Any valid token means they are authenticated."* → Authentication is not authorization; check the specific
  scope each operation requires, not merely that a token is present.
- *"We trust our issuer."* → Confirm you validate the specific issuer, not just any known signer; a multi-issuer
  setup enables a mix-up otherwise.
- *"Tokens expire quickly."* → Expiry limits the window, not the domain; an unbound token is usable at any
  audience for its lifetime.

## Executing this in practice

You need, per operation, the required audience, issuer, and scope, and what the resource server actually
validates on each call. For each service, present a cross-audience token, a wrong-issuer token, and an
insufficient-scope token, and observe what is accepted. Reading the token-validation code shows the intended
domain checks; a token accepted outside its domain shows whether they hold.

## Related

- `auditing-jwt-verification-and-key-trust` - the token-verification mechanics beneath this; algorithm and key
  confusion undermine the same audience and issuer claims this skill checks.
- `auditing-saml-and-oidc-federation-trust` - the federation companion; assertion and issuer trust are the
  identity-layer version of audience and issuer validation.
- `hunting-broken-object-level-authorization` - once a token is accepted, whether it may act on the specific
  object is the next authorization question this pairs with.
- `mapping-service-account-impersonation-chains` - tokens that cross audiences are one way an identity reaches
  beyond its intended scope; the two maps meet at token exchange.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the token presented to the wrong audience or beyond its
  scope, sink = the resource operation it reaches, evidence = the missing audience, issuer, or scope validation.
