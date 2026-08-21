---
name: auditing-device-code-and-pkce-flows
description: >-
  Audit the server side of the authorization-code-with-proof-key and device-authorization grants for
  bugs that let a stolen or guessed code become a token. Covers a token endpoint that issues without
  checking the proof-key verifier at all, that accepts the plain challenge method or a challenge-absent
  downgrade, or that binds the verifier to the client rather than to the specific code; and a device
  grant whose short user code is brute-forceable because polling is unthrottled, whose device code is
  not bound to the requesting client, or whose approval is not tied to the authenticated approver.
  Scoped to the proof-key and device-code specifics, not general federated login, which a separate
  skill covers. Use when reviewing a token endpoint or a device-authorization endpoint. The token
  request parameters are the source, token issuance is the sink, and an unenforced proof binding
  between them is the bug.
license: MIT
---

# Auditing proof-key and device-code grants: when a code becomes a token without proof

The proof-key exchange and the device-authorization grant both exist to close one gap: a
authorization code, on its own, can be intercepted, and a short user code can be guessed. Each grant
adds a proof that binds the token request to the party that actually started the flow, a verifier
that only the real client knows, or a device code that only the real device holds. The server side is
where that proof is enforced, and where it is quietly not. When the token endpoint issues without
checking the verifier, honors a downgraded challenge, or hands tokens for a device code it never
bound to a client, an intercepted or guessed code becomes a live token. You find these by reading the
issuance path and asking what proof it demands before it mints a token.

## When to use

- The code implements or wraps a token endpoint that redeems an authorization code, or a device-authorization grant.
- A public client (one that cannot hold a secret) relies on the proof key as its protection against code interception.
- You want to know whether a captured code, a downgraded challenge, or a guessed user code yields a token.

## Scope check

Exercise token and device-authorization endpoints only on systems you own or are authorized to
assess, with test clients and test accounts. A confirmed issuance bypass is an authentication bypass,
so treat it as account-takeover-grade and coordinate. If you can't name the authorization, stop.

## The loop

1. **Map the issuance endpoints and whether a conformant server owns them.** Find the token endpoint
   and, if present, the device-authorization and approval endpoints. Determine whether issuance is
   delegated to a conformant authorization server or handled in application code. If it proxies to a
   conformant server that enforces the proof, the local absence of a check is not the bug; confirm the
   delegation actually covers this path before reading further.

2. **Check that the proof-key verifier is enforced.** On the code-for-token exchange, the endpoint must
   load the challenge stored against this exact authorization code and confirm the supplied verifier
   derives to it under the strong method. The bug is a handler that reads the code, looks up the grant,
   and issues without ever reading the verifier, so an intercepted code redeems with no proof at all.

3. **Check for method and presence downgrades.** The endpoint must require the strong challenge method
   and reject the plain method, where challenge and verifier are equal and an interceptor who saw the
   challenge already holds the verifier. It must also enforce the invariant that a challenge present at
   authorization time requires a matching verifier at token time; issuing when the challenge is simply
   absent lets an attacker strip the proof entirely. Confirm the verifier is bound per authorization
   code, not stored per client, or any valid verifier for that client redeems any code.

4. **Check the device grant's user code and polling.** The user code is short and human-entered, so its
   only protection is that guessing is throttled and the code is single-use and short-lived. The bug is
   a poll or approval path with no interval, backoff, or attempt cap, letting an attacker brute-force a
   pending code, combined with a user code drawn from a weak generator or one that outlives its window
   or survives a redemption.

5. **Check the device-code and approval bindings.** The device code presented at the token endpoint must
   be bound to the client it was issued to, or one client redeems another's code. The approval step must
   tie the approved code to the authenticated user who approved it and be protected against a
   cross-site-request forcing an approval; otherwise an attacker's pending flow is completed with a
   victim's identity. Where a redirect target is involved, it must match exactly, not by prefix.

6. **Confirm and record.** Confirm by driving the live endpoint: redeem a code with no verifier or a
   plain challenge, poll a user code without pause, or redeem a device code from a different client, and
   show a token issued. Kill the lead if issuance is delegated to a conformant server, the strong method
   is required and per-code bound, the challenge-present invariant holds, and the device grant enforces
   an interval, single-use high-entropy user codes with a short expiry, and strict device-code and
   approval bindings. Record the endpoint, the missing proof, and the token obtained.

## Where grant proof leaks

- **A public client has no secret, so the verifier is its whole defense.** If the token endpoint does
  not demand it, code interception is enough; there is nothing else to stop it.
- **The plain method turns the proof into a copy of the challenge.** Anyone who saw the authorization
  request holds the verifier. Only the strong method binds anything.
- **Downgrade by omission is the quiet version.** If the endpoint enforces the proof only when the
  client sends a challenge, an attacker simply does not send one.
- **A user code is short by design.** Its safety is a throttle and a short life, not its length; an
  unthrottled poll or approval endpoint brute-forces it.
- **A device code unbound to a client is a bearer token for the wrong bearer.** Binding the code to the
  client that requested it is what stops cross-client redemption.

## Worked example (a confirm and a kill)

> **Confirm.** The token endpoint reads the authorization code, loads the grant, and issues an access
> token. The verifier parameter is never read, and the challenge stored at authorization time is never
> compared. The client is public. A captured authorization code, replayed with no verifier, yields a
> token. **Confirmed** proof-key verifier not enforced, `high`, remediation = require the strong
> challenge method, load the challenge bound to the exact code, reject any exchange whose verifier does
> not derive to it, and reject a public-client exchange that carries no proof.
>
> **Kill.** The endpoint forwards the exchange to a conformant authorization server that enforces the
> strong method, binds the verifier to the code, and rejects the plain method and challenge-absent
> exchanges; the device grant it wraps enforces a polling interval, single-use high-entropy user codes
> with a short expiry, and a device-code-to-client binding. Replaying without a verifier is rejected
> upstream. **Killed**, `kill_reason` = "issuance delegated to a conformant server that enforces a
> per-code strong-method verifier and device-code binding; downgraded and proofless exchanges rejected."

## Rationalizations to reject

- *"The code is single-use, so interception does not matter."* -> Single-use only means the attacker
  must redeem before the client does; the proof key is what makes a raced code useless.
- *"We support the plain method for older clients."* -> The plain method offers no protection against an
  interceptor. Supporting it is the downgrade, not a compatibility nicety.
- *"The user code is random, so it cannot be guessed."* -> A short human-entered code is guessable in
  bulk unless polling and approval are throttled and the code expires quickly.
- *"The device code is unguessable, so binding is redundant."* -> Binding stops a different client from
  redeeming a code it obtained, not just a guesser; unguessability is not the same property.
- *"The upstream server handles the proof."* -> Only if this path actually reaches it with the
  parameters intact. Confirm the delegation, do not assume it.

## Executing this in practice

You need the token-endpoint handler, the device-authorization and approval handlers, the store that
holds challenges and device codes, and the polling and expiry logic. For each, decide what proof is
demanded before a token is issued and whether that proof is bound to the specific code and client.
Reading the issuance path tells you which checks exist; driving a proofless, downgraded, or
cross-client exchange against a test client tells you which ones hold.

## Related

- `auditing-saml-and-oidc-flows` - the general federated-login sibling; this skill deliberately narrows
  to the proof-key and device-code enforcement those flows leave to the token endpoint.
- `auditing-webauthn-and-passkey-flows` - another authentication-flow audit where a valid-looking
  request must be bound to its true initiator before it grants access.
- `auditing-randomness-and-nonce-quality` - a weak user code or authorization code is a grant bug rooted
  in a weak generator; that skill covers the entropy this one assumes.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the token request parameters, sink = token
  issuance, evidence = the proofless or downgraded exchange that still returns a token.
