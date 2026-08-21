---
name: auditing-cors-and-cross-origin-trust
description: >-
  Audit the code and configuration that decide cross-origin access, for trust a browser turns into a
  read of authenticated data. Covers a response that reflects an arbitrary request origin into the
  allow-origin header alongside allow-credentials, an allowlist that accepts the null origin, allowlist
  matching by prefix, suffix, substring, or an unanchored pattern that a lookalike origin satisfies, the
  origin header trusted as an authorization or request-forgery defense, and a cross-window message
  handler that acts on data without an exact origin and source check. Scoped to the code and config that
  build the decision, not a live-header scan. Use when reviewing cross-origin response headers,
  origin-based access logic, or cross-window message handlers. The request origin or the posted message
  is the source, the credentialed cross-origin read or the message sink is the sink, and trusting an
  attacker-set origin is the bug.
license: MIT
---

# Auditing CORS and cross-origin trust: when an attacker's origin is trusted

The request origin is fully attacker-controlled: any site the victim visits sets it. Cross-origin
sharing is safe only because the server decides, per origin, whether to hand back an authenticated
response, and a browser enforces that decision. The bugs are all the same shape: the server trusts the
origin it was told. It reflects whatever origin arrived and pairs it with credentials, so any site
reads the victim's authenticated data; it matches an allowlist by substring, so a lookalike passes; it
treats the origin as proof of who is calling. The client-side mirror is a message handler that acts on
a posted message without checking where it came from. You find these by reading the code that builds
the allow-origin decision, or handles a cross-window message, and asking whether an attacker-set origin
is trusted.

## When to use

- The code sets cross-origin response headers, statically or by computing them from the request origin.
- An access, authorization, or request-forgery decision is made by looking at the origin or referer.
- Client code receives cross-window messages and acts on their data.

## Scope check

Test cross-origin behavior only against applications you own or are authorized to assess, from a test
origin and test accounts. A confirmed credentialed cross-origin read exposes real user data, so
coordinate. If you can't name the authorization, stop.

## The loop

1. **Map the decision points.** Find where the allow-origin and allow-credentials headers are set and
   whether the origin value is static or reflected from the request, where any access or request-forgery
   logic reads the origin or referer, and where client code registers a cross-window message handler.
   These are the sources and sinks the rest of the loop examines.

2. **Check reflected origin with credentials.** The dangerous combination is an allow-origin header set to
   the arbitrary incoming origin together with allow-credentials enabled, on an endpoint that returns
   cookie-authenticated data. That lets any site read the victim's authenticated response. A wildcard
   without credentials on genuinely public data is not this bug.

3. **Check the allowlist match and the null origin.** If the origin is validated against an allowlist,
   read how. A prefix, suffix, substring, or unanchored pattern match accepts a lookalike origin that
   contains or ends with the allowed string, and an unescaped separator in a pattern matches more than it
   appears to. An allowlist that contains the literal null origin trusts a value an attacker can force
   from a sandboxed context. Only an exact match against a fixed set is a real check.

4. **Check the origin as an authorization or request-forgery defense.** The origin and referer are
   spoofable from non-browser clients and sometimes absent, so any access or state-change decision that
   relies on them as the sole control is bypassable. Flag origin-based authorization and origin-only
   request-forgery protection.

5. **Check cross-window message handlers.** A message handler that reads the message data into a sink, the
   document, navigation, or evaluation, without checking that the origin exactly matches an expected value
   lets any origin drive that sink. Flag a missing origin check, a weak one, and a handler that also fails
   to confirm the source window; flag sending a secret to a wildcard target as well.

6. **Confirm and record.** Confirm a header finding by issuing a cross-origin request from a test origin
   and showing the response is readable with credentials attached; confirm a message finding by posting
   from an unexpected origin and reaching the sink. Kill the lead if the wildcard carries no credentials on
   non-sensitive data, the endpoint is authenticated by a token the browser will not attach cross-site
   rather than a cookie, allow-credentials is absent so the body is unreadable, the allowlist is an exact
   match, or the message handler only reads inert data. Record the endpoint or handler, the trusted value,
   and what it exposed.

## Where cross-origin trust leaks

- **The origin is the attacker's to set.** Every check that trusts it without an exact allowlist is
  trusting the attacker. Reflection is the extreme case: trusting all of them.
- **Credentials are what make a cross-origin read matter.** Reflected origin plus credentials plus
  authenticated data is the exploitable triple; drop any one and the severity collapses.
- **Substring matching is the classic allowlist bug.** A suffix test for a domain also accepts a lookalike
  that ends with it; only exact equality against a fixed set holds.
- **The origin is not an identity.** Using it for authorization or as the only request-forgery defense
  trusts a header a non-browser client sets freely.
- **A message handler is a cross-origin entry point in the page.** Without an exact origin and source
  check, any site that can open the window drives whatever the handler does with the data.

## Worked example (a confirm and a kill)

> **Confirm.** An account endpoint sets the allow-origin header to the incoming request origin and enables
> allow-credentials, and returns the signed-in user's profile from a session cookie. A page on an attacker
> origin issues a credentialed cross-origin request and reads the victim's profile. **Confirmed** reflected
> origin with credentials, `high`, remediation = set allow-origin only from an exact-match allowlist,
> never reflect an arbitrary origin with credentials, and add a vary-on-origin response so a shared cache
> cannot cross responses.
>
> **Kill.** The same endpoint validates the origin against a fixed set by exact equality and only then
> echoes it, rejects the null origin, sets a vary-on-origin response, and elsewhere authorizes by a token
> the browser does not attach cross-site rather than by the origin; the one message handler checks the
> event origin by exact equality and the source window before using the data. A request from a test origin
> is not reflected and cannot read the body. **Killed**, `kill_reason` = "origin echoed only on exact-match
> allowlist without the null origin, credentials not exposed to unlisted origins, and message handler
> validates origin and source before its sink."

## Rationalizations to reject

- *"We only reflect origins we trust."* -> Reflecting the incoming origin trusts all of them unless an
  exact-match allowlist gates it first. Read the gate, not the intent.
- *"The wildcard is fine, it is public data."* -> Then it is fine, and this is not the finding. The bug is
  a reflected or listed origin paired with credentials on authenticated data.
- *"We check the origin ends with our domain."* -> A suffix check accepts a lookalike that ends with your
  domain. Only exact equality is a check.
- *"We block cross-site requests by checking the origin."* -> A non-browser client sets the origin freely
  and sometimes omits it; origin alone is not a request-forgery defense.
- *"The message handler is internal."* -> Any origin that can open or frame the window can post to it. If
  the handler reaches a sink, it needs an exact origin and source check.

## Executing this in practice

You need the code that sets the allow-origin and allow-credentials headers and whether the origin is
reflected or listed, the allowlist match logic, any origin-based authorization or request-forgery check,
and every cross-window message handler with the sink it feeds. For each, decide whether an attacker-set
origin is trusted and whether credentials or a real sink make it matter. Reading the code tells you the
match shape; a credentialed request from a test origin, or a message from an unexpected one, tells you
whether it holds.

## Related

- `testing-client-side-dom-vulnerabilities` - the cross-window message handler is a client-side sink both
  skills care about; that one covers the injection it can cause in depth.
- `auditing-webhook-authenticity-and-callback-trust` - a sibling trust-boundary audit where an
  attacker-controlled value is trusted without proof.
- `reviewing-content-security-policy` - the other browser-enforced control on the same pages; a weak policy
  and a permissive cross-origin decision often compound.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the request origin or posted message, sink = the
  credentialed cross-origin read or the message sink, evidence = the readable authenticated response or the
  sink reached from an unexpected origin.
