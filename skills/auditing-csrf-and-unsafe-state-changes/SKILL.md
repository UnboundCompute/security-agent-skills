---
name: auditing-csrf-and-unsafe-state-changes
description: >-
  Audit state-changing endpoints for cross-site request forgery, where a request that rides the victim's
  ambient cookies is authorized on that session alone with no unpredictable, session-bound proof the request
  came from the app, letting an attacker page trigger the change as the victim. Use when reviewing forms and
  actions that modify data, change settings, move funds, or alter access, and how each is protected. Covers
  missing or unvalidated tokens, tokens not bound to the session, cookie-only same-site reliance and its
  gaps, state-changing GET requests, and content-type or method-override assumptions. The cross-site request
  on the victim's session is the source, the state-changing endpoint is the sink, and acting without
  unpredictable session-bound proof of origin is the bug.
license: MIT
---

# Auditing CSRF and unsafe state changes: when the victim's cookies do the attacker's work

A browser attaches a site's cookies to every request to that site, whoever caused the request. So when a
state-changing endpoint authorizes an action on the session cookie alone, an attacker's page can submit a
form or fire a request to that endpoint, the victim's browser attaches the session, and the change happens
as the victim without them ever intending it. The defense is proof that the request originated from the
app's own pages and not a foreign one: an unpredictable token bound to the session that an attacker cannot
guess or read cross-origin, or a cookie policy that withholds the cookie on cross-site requests. The bug is
a state change that trusts the cookie and asks for nothing else. You find it by listing every action that
mutates state and checking what, beyond the cookie, each one requires.

## When to use

- An endpoint changes data, settings, access, or funds and is authorized by a session cookie.
- Forms or actions rely on a token, and you need to confirm it is present, validated, and session-bound.
- The app relies on same-site cookie behavior, custom headers, or content-type as its only cross-site guard.

## Scope check

Test cross-site request forgery only against applications you own or are authorized to assess, using test
accounts and a benign state change you can observe and reverse, never triggering a real irreversible action
on another user. A confirmed case performs a real mutation as the victim, so stay within the authorized
scope. If you can't name the authorization, stop.

## The loop

1. **Establish what each state change requires beyond the cookie first.** For every mutating endpoint,
   determine whether it demands an unpredictable, session-bound token that it validates, or relies on a
   fully-enforced same-site cookie, or nothing at all. This is the false-positive killer: an endpoint that
   validates a session-bound token on every mutation, or is reached only with a cookie the browser withholds
   cross-site, is defended, while one authorized on the ambient cookie alone is not. Name the requirement
   before building a cross-site request.

2. **Enumerate every state-changing action.** List the endpoints that modify data, change settings or access,
   move funds, or alter the account, including ones reachable by GET. Each is a candidate sink; read-only
   endpoints are not the target here.

3. **Judge the token if there is one.** A token defends only if it is present on the request, unpredictable,
   bound to the session so one user's token does not validate another's, validated server-side on every
   mutation, and not readable cross-origin or reflected back to an attacker. Decide whether the token is
   missing, static, shared, unvalidated, or improperly bound.

4. **Judge the same-site and header reliance.** Same-site cookie behavior mitigates cross-site sends but has
   gaps: the default still permits top-level navigations for some methods, subdomains and same-site siblings
   are not cross-site, and a permissive setting or an older client removes the protection. A custom-header
   requirement defends only if the endpoint rejects the request without it and the header cannot be set
   cross-origin. Decide whether the reliance actually withholds authorization cross-site.

5. **Check the method and content-type assumptions.** A state change reachable by GET is forgeable with a
   simple link or image; an endpoint that assumes a non-simple content-type but also accepts a simple one, or
   honors a method-override parameter, can be reached by a cross-site form. Determine whether the endpoint
   enforces a safe method and rejects the simple, cross-site-submittable request shapes.

6. **Confirm and record.** Confirm by issuing the state change from a foreign origin with the victim's session
   present and no valid token, on test accounts on an isolated instance, and observing the mutation. Kill the
   lead if a session-bound token is validated on every mutation, if the cookie is genuinely withheld
   cross-site for this request shape, if the endpoint rejects the cross-site-submittable form, or if the
   endpoint changes no state. Record the endpoint, the missing proof, the request shape, and the change
   effected, or set a `kill_reason`.

## Where CSRF leaks

- **The cookie is ambient.** The browser sends it on any request to the site, so a state change authorized on
  the cookie alone is forgeable; the finding is the absence of a second, unpredictable, session-bound proof.
- **Tokens must be bound and validated.** A token that is static, shared across users, optional, or unchecked
  is decoration; only a per-session unpredictable token validated on every mutation is a defense.
- **Same-site has gaps.** The default policy still allows some cross-site top-level sends, and siblings and
  subdomains are not cross-site, so same-site reliance alone leaves specific shapes forgeable.
- **GET that mutates is the easy case.** A state change reachable by GET is triggered by an image or a link
  with no form at all, so safe methods for mutations matter.
- **Content-type and override assumptions break.** An endpoint that trusts a non-simple content-type but also
  accepts a simple one, or honors a method override, is reachable by a plain cross-site form.

## Worked example (a confirm and a kill)

> **Confirm.** An email-change endpoint accepts a form post and authorizes it on the session cookie alone,
> with no token and a cookie policy that permits the cross-site send. A page on a foreign origin auto-submits
> the form while the victim is logged in, and the account email changes to the attacker's on test accounts on
> an isolated instance. **Confirmed** cross-site request forgery on an account-takeover action, `high`,
> remediation = require an unpredictable session-bound token validated on every mutation, set the session
> cookie to withhold on cross-site requests, and reject state changes issued by GET or simple cross-site
> forms.
>
> **Kill.** The same endpoint requires a per-session unpredictable token that it validates server-side on
> every submission, rejects the request when the token is absent or belongs to another session, is reachable
> only by a safe method with a cookie the browser withholds cross-site, and refuses the simple-content-type
> form. A cross-origin submission without the token is rejected and no change occurs. **Killed**,
> `kill_reason` = "every mutation requires a validated session-bound token and the cookie is withheld
> cross-site for this shape; a foreign origin cannot forge the request with the victim's ambient cookie."

## Rationalizations to reject

- *"The user is authenticated, so the request is theirs."* -> Authentication rides on the ambient cookie the
  browser sends for any origin; the missing piece is proof the request came from your pages, not a foreign
  one.
- *"We set the cookie same-site."* -> That helps but has gaps for top-level sends, siblings, and subdomains,
  and evaporates on a permissive setting or older client; confirm it withholds the cookie for this exact
  shape, and prefer a token as well.
- *"There is a token on the form."* -> Confirm it is unpredictable, bound to the session, and validated on
  every mutation; a static, shared, optional, or unchecked token defends nothing.
- *"It requires a JSON content-type."* -> If the endpoint also accepts a simple content-type or honors a
  method override, a cross-site form reaches it; enforce the content-type and reject simple submissions.
- *"It is only a preference change."* -> Preferences include email, recovery address, and access settings that
  lead to takeover; audit every state change, not only the obviously sensitive ones.

## Executing this in practice

You need every state-changing endpoint, what each requires beyond the session cookie, the token's
unpredictability and session binding and whether it is validated on every mutation, the cookie policy's
behavior for each request shape, and whether GET or simple cross-site forms reach any mutation. For each,
decide whether a foreign origin can drive the change with only the ambient cookie. Reading the token
validation and cookie policy settles most leads; issuing the change from a foreign origin on test accounts on
an isolated instance settles the rest.

## Related

- `auditing-session-lifecycle-and-fixation` - the session cookie that CSRF abuses is the same one that skill
  audits for lifecycle and fixation, so the cookie's flags and scope are examined together.
- `auditing-cors-and-cross-origin-trust` - the mirror question of which cross-origin callers may act, from the
  server's own headers rather than the browser's ambient cookie.
- `hunting-broken-object-level-authorization` - CSRF forges a request the victim is allowed to make, while
  that skill hunts requests the caller should not be allowed to make; together they cover who may change what.
- `reviewing-rate-limiting-and-abuse-controls` - a mutating endpoint without origin proof is also worth
  checking for abuse volume, so the two controls are assessed on the same actions.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the cross-site request on the victim's session, sink
  = the state-changing endpoint, evidence = the mutation performed from a foreign origin without valid
  session-bound proof, on test accounts on an isolated instance.
