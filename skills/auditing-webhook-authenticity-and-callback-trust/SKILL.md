---
name: auditing-webhook-authenticity-and-callback-trust
description: >-
  Audit both directions of webhook trust: an inbound handler that acts on a payload without proving it
  authentic, and an outbound fetch of a caller-supplied URL that reaches internal targets. Covers
  inbound handlers with no signature check, a signature compared in non-constant time, a signature
  computed over a re-serialized body instead of the exact raw bytes, a verification result that is
  computed but never enforced, and no timestamp or replay defense; and outbound callback or fetch URLs
  validated by substring or blocklist, or by a single pre-connect lookup that a redirect or a rebind
  defeats. Use when reviewing code that receives a signed webhook and performs a state change, or that
  fetches a URL the caller controls. The inbound request or the caller-supplied URL is the source, the
  state-changing handler or the server-side fetch is the sink, and a missing or bypassable trust check
  between them is the bug.
license: MIT
---

# Auditing webhook authenticity and callback trust: proving the caller and pinning the target

A webhook is a trust boundary in two directions and both are easy to get wrong. Inbound, an internet
caller posts a payload and the handler acts on it, provisioning access or crediting a balance, as if
it came from the real sender; the only thing separating the real sender from anyone is a signature
check that is often absent, compared unsafely, computed over the wrong bytes, or thrown away.
Outbound, the same systems fetch a URL the caller supplied, a callback target or an import link, and a
server-side request to an internal address reaches things the internet cannot. You find both by
tracing the inbound request to the action it drives and the caller-supplied URL to the request it
makes, and asking what proof or restriction stands between.

## When to use

- A handler receives a webhook or callback and performs a state change based on its payload.
- The application fetches a URL that a user registered or supplied, for a callback, import, preview, or discovery.
- You want to know whether an unauthenticated caller can forge an event or steer a server-side request inward.

## Scope check

Send forged events or callback URLs only to systems you own or are authorized to test, and prove
internal reach only against infrastructure in scope. A working webhook forgery or a server-side
request to internal services can move money or expose credentials, so coordinate. If you can't name
the authorization, stop.

## The loop

1. **Map inbound handlers and outbound fetches.** Inventory the endpoints that receive a webhook and act
   on it, and the places the server fetches a caller-influenced URL. For each inbound handler, note what
   action it drives; for each outbound fetch, note whether the URL comes from user input. These are the
   two sinks the rest of the loop reasons about.

2. **Check that inbound verification exists and is enforced.** The handler must verify a signature over
   the payload with a server-held secret before it acts. The bug is a handler that never references the
   secret, or that computes a verification result and then proceeds regardless, or that swallows a
   verification error and falls through to processing. A result that is not gated on is not a check.

3. **Check the comparison, the signed bytes, and replay.** The signature comparison must be
   constant-time, or its timing leaks the expected value byte by byte. The signature must be computed
   over the exact raw request bytes, not a re-serialized or framework-parsed body, or a benign
   reordering breaks legitimate signatures and tempts a developer to weaken the check. And a timestamp
   within a tight window plus a stored nonce or idempotency key must stop a captured valid request from
   replaying.

4. **Check outbound destination validation.** For each caller-supplied URL, read how the destination is
   restricted. A substring, prefix, or suffix test on the host is bypassable by a lookalike, a userinfo
   segment, or a trailing dot; a blocklist of literal loopback names is bypassable by an alternate IP
   encoding or a private range; an unrestricted scheme lets non-web schemes reach internal services. The
   safe shape resolves the host and confirms the resolved address is public.

5. **Check redirects and connect-time resolution.** A destination validated once and then fetched can
   still reach an internal target if the client follows a redirect to it, or if the name resolves to a
   public address at validation time and an internal one at connect time. The safe shape pins the
   connection to the address it validated and re-validates or refuses every redirect hop.

6. **Confirm and record.** Confirm inbound by posting a forged event that the handler acts on, or by
   showing the comparison is non-constant-time, the signed bytes are wrong, or replay succeeds. Confirm
   outbound by steering a fetch to an internal or link-local target, or to a redirect that lands there.
   Kill an inbound lead if a vetted verification routine checks a raw-body signature in constant time,
   enforces the result, and bounds replay; kill an outbound lead if the destination is resolved, pinned
   to a public address, scheme-restricted, and redirect-safe, or the URL is only stored and never
   fetched. Record the handler or fetch, the missing check, and what it drove.

## Where webhook trust leaks

- **An inbound handler with no signature is an open, authenticated-looking endpoint.** Anyone who knows
  the URL drives the action. The signature is the only thing making the sender the sender.
- **The comparison and the signed bytes are where a present check still fails.** A non-constant-time
  compare leaks the secret over time; a signature over a re-serialized body verifies the wrong thing.
- **A valid request with no replay defense is reusable.** Without a timestamp window and a nonce store, a
  captured event fires again whenever the attacker likes.
- **An outbound fetch of a caller URL points inward by default.** Loopback, private ranges, and the
  link-local metadata address are reachable unless the destination is resolved and pinned to a public one.
- **A redirect is a second, unvalidated URL.** Validating only the first hop lets the server be walked
  inward by a response it trusts.

## Worked example (a confirm and a kill)

> **Confirm.** A billing webhook handler computes a signature over the parsed and re-serialized body and
> compares it to the header with an ordinary equality check, then marks an invoice paid. The comparison
> is non-constant-time and the signed bytes differ from what the sender signed, so verification is
> effectively decorative; a forged event marks arbitrary invoices paid. **Confirmed** unauthenticated
> webhook forgery, `critical`, remediation = verify a constant-time signature over the exact raw request
> bytes with the server-held secret, enforce rejection on failure, and bound replay with a timestamp
> window and an idempotency key.
>
> **Kill.** The handler passes the raw body and the signature header to a vetted verification routine that
> compares in constant time and enforces a timestamp tolerance, acting only on success; a separate
> outbound fetch resolves the caller URL, refuses private and link-local addresses, pins the connection
> to the resolved public address, restricts the scheme, and rejects redirects. Forged and replayed events
> are rejected and the fetch cannot be steered inward. **Killed**, `kill_reason` = "raw-body constant-time
> signature enforced with replay bounds; outbound destination resolved, pinned, scheme-restricted, and
> redirect-safe."

## Rationalizations to reject

- *"The webhook URL is secret, so only the sender knows it."* -> A URL is not a credential; it leaks in
  logs, referrers, and history. The signature, not the obscurity of the path, authenticates the sender.
- *"We verify the signature."* -> Over the raw bytes or a re-serialized body, and in constant time, and is
  the result actually enforced? A verify that runs and is ignored protects nothing.
- *"There is no replay risk, the event is idempotent."* -> Only if the handler truly is; most state
  changes are not, and a captured valid event replays without a timestamp and nonce.
- *"The callback host is on our allowlist."* -> Matched how? A suffix or substring test passes a lookalike,
  and a validated name can still resolve or redirect inward at connect time.
- *"It only fetches over the web scheme."* -> To any address, including internal ones, unless the resolved
  address is confirmed public and the connection pinned to it.

## Executing this in practice

You need the inbound handler bodies and the action each drives, the verification routine and exactly
what bytes it signs and how it compares, the replay defenses, and for each outbound fetch the URL's
origin and the destination-validation and redirect logic. For each inbound handler decide whether a
forged or replayed event is accepted; for each fetch decide whether it can be steered to an internal
target. Reading the code tells you the shape; posting a forged event or steering a fetch on a target in
scope tells you whether the boundary holds.

## Related

- `exploiting-ssrf-to-cloud-metadata` - the outbound half taken to its worst case; both trace a
  caller-supplied URL to a server-side request and ask what internal reach it buys.
- `auditing-cors-and-cross-origin-trust` - a sibling trust-boundary audit where an attacker-set value, the
  request origin, is trusted without proof.
- `finding-fail-open-flaws` - a swallowed verification error that falls through to processing is a
  fail-open control; both hunt a check that does not stop what it should.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the inbound request or the caller-supplied URL,
  sink = the state-changing handler or the server-side fetch, evidence = the forged event or the internal
  target reached.
