---
name: testing-web-cache-deception
description: >-
  Test web cache deception, where an attacker crafts a static-looking URL, using an added extension, a path
  delimiter, or a query variant, that maps to the same authenticated dynamic response while a shared cache,
  keying on the apparent extension or path, stores that private response and serves it to the attacker. Use
  when a shared cache sits in front of authenticated content and its rule for what is cacheable can disagree
  with what the origin treats as the same resource. Covers extension-based caching rules, path and delimiter
  confusion, and cache-key versus origin-routing mismatches. The crafted static-looking URL is the source,
  the shared cache that stores the authenticated response is the sink, and the victim's private response
  being cached and retrieved by the attacker is the bug.
license: MIT
---

# Testing web cache deception: when a fake file extension caches a private page

A shared cache and an origin server can disagree about what a URL is. The origin routes by matching a
pattern and often ignores a trailing segment, so a request for an account page with a made-up static suffix
still returns the account page, authenticated and personal. The cache, meanwhile, decides what to store by a
simpler rule, often the apparent file extension or path prefix, and treats that same URL as a harmless
static asset it may cache for everyone. When both are true at once, an attacker lures a logged-in victim to
the crafted URL, the origin serves the victim's private response, and the cache stores it under a key the
attacker can then request unauthenticated, retrieving the victim's data. This is distinct from cache
poisoning: nothing in the response is altered; a private response is simply cached where it should never be.
The bug is a cacheability decision that diverges from the origin's notion of the same, authenticated
resource. You find it by comparing what the cache stores against what the origin actually served.

## When to use

- A shared cache or content delivery layer sits in front of authenticated, per-user dynamic responses.
- The cache decides cacheability by file extension, path prefix, or a static-looking pattern.
- The origin ignores or tolerates extra path segments, suffixes, or delimiters when routing a request.

## Scope check

Test cache deception only against systems you own or are authorized to assess, on non-production
infrastructure, using two test accounts you control so any cached private response is your own, never a real
user's data. A confirmed case exposes an authenticated response to an unauthenticated request, so keep every
probe within the authorized scope and prefer an isolated instance. If you can't name the authorization, stop.

## The loop

1. **Establish whether the origin serves authenticated content for the crafted path and the cache stores it,
   both at once, first.** Confirm two things together: that the origin returns the same private, authenticated
   response for the static-looking URL as for the real one, and that the shared cache actually stores that
   response. This is the false-positive killer: if the origin returns a not-found or a login redirect for the
   crafted path, or the cache does not store the response, there is no deception, and a URL that merely looks
   static is not a finding. Name both behaviors before claiming impact.

2. **Map the cache's cacheability rule.** Determine what makes the cache store a response: a list of
   extensions, a static path prefix, the absence of certain headers, or a pattern. This is the rule an
   attacker will try to satisfy with a URL the origin still treats as the authenticated resource.

3. **Craft the static-looking variant.** Depending on the rule, append a cacheable-looking extension after the
   real path, add a path segment or a delimiter the origin ignores but the cache reads as a different or
   static resource, or use an encoded separator that the two parse differently. The goal is one URL the cache
   deems cacheable and the origin routes to the private response.

4. **Verify the origin still returns the private response.** Request the crafted URL as an authenticated user
   and confirm the response is the personal, authenticated content, not an error or a static file. If the
   origin instead returns a generic asset or refuses, the deception does not hold for that variant.

5. **Verify the cache stores and serves it to a different requester.** Request the crafted URL again without
   the victim's session, from a second account or unauthenticated, and confirm the cache returns the first
   user's private response rather than re-authenticating. Cache-status headers, timing, and the returned
   personal content distinguish a stored hit from a fresh origin fetch.

6. **Confirm and record.** Confirm end to end on an isolated instance with two test accounts: user one loads
   the crafted URL, user two (or an unauthenticated request) retrieves user one's private response from the
   cache. Kill the lead if the origin does not serve authenticated content for the crafted path, if the cache
   does not store the response, if the cache re-validates authorization, or if the cache key already
   distinguishes the authenticated resource. Record the crafted URL, the cache rule, the origin behavior, and
   the retrieved private content, or set a `kill_reason`.

## Where cache deception leaks

- **The cache and origin disagree about the resource.** The whole bug is a cacheability rule that says static
  while the origin serves a private page for the same URL; find the gap between the two notions of the URL.
- **Extension rules are the classic lever.** A cache that stores anything ending in a known static extension
  will store an account page when the origin ignores an appended suffix.
- **Delimiters and extra segments confuse routing.** A path delimiter or an extra segment the origin tolerates
  but the cache reads differently produces one URL with two meanings.
- **It is theft, not alteration.** Unlike poisoning, the response is unchanged; the harm is that a private
  response is stored under a key an attacker can fetch, so the finding is where it is cached, not what it says.
- **The victim only has to visit.** The attacker supplies the crafted URL; the victim's own authenticated
  request populates the cache, after which the attacker reads it, so a single visit is the trigger.

## Worked example (a confirm and a kill)

> **Confirm.** An account page is served by the origin for its path with an appended made-up static
> extension, returning the same authenticated content, while the shared cache stores any response whose URL
> ends in that extension. User one visits the crafted URL and the cache stores their personal account page;
> an unauthenticated request for the same URL returns user one's data from the cache on an isolated instance.
> **Confirmed** web cache deception exposing authenticated content, `high`, remediation = cache by the
> origin's content type and explicit cache-control rather than the apparent extension, do not cache responses
> to authenticated requests, and make the origin reject or normalize unexpected suffixes and delimiters
> instead of ignoring them.
>
> **Kill.** The origin returns a not-found for the appended-extension URL rather than the account page, the
> cache honors the origin's no-store on authenticated responses and keys on the normalized path, and a
> repeat request without the session re-authenticates at the origin. The crafted URL yields no stored private
> response. **Killed**, `kill_reason` = "the origin does not serve authenticated content for the crafted path
> and the cache does not store authenticated responses; no private response is ever cached for an attacker to
> retrieve."

## Rationalizations to reject

- *"That URL is not a real page."* -> If the origin ignores the suffix and returns the real authenticated
  page anyway, the URL is real enough; test what the origin serves, not what the route table intends.
- *"The cache only stores static files."* -> It stores what its rule calls static, which a crafted extension
  or delimiter can satisfy while the origin serves a private page; the rule, not the reality, decides.
- *"Authenticated responses are not cached."* -> Confirm that for the crafted URL specifically; the deception
  works precisely when the cacheability rule fires before the authenticated nature is considered.
- *"This is just cache poisoning."* -> It is the reverse: nothing in the response is altered, a private
  response is stored where an attacker can read it; the analysis and fix differ.
- *"The cache key includes the session."* -> Then confirm it does for this URL shape; deception thrives where
  the key is the apparent-static path and omits the session for URLs the rule deems cacheable.

## Executing this in practice

You need the cache's cacheability rule, the origin's routing tolerance for suffixes, extra segments, and
delimiters, and the cache key it computes for each URL shape. For each authenticated resource, craft a
static-looking variant, confirm the origin still returns the private response, and confirm a second requester
retrieves it from the cache. Two test accounts on an isolated instance make the theft observable end to end;
cache-status headers and the returned personal content confirm a stored hit rather than a fresh fetch.

## Related

- `testing-web-cache-attacks` - the poisoning side of shared-cache abuse, where the response is altered rather
  than stolen; the two share the cache-key and cacheability analysis from opposite directions.
- `hunting-host-header-and-url-parsing-trust` - the origin-versus-cache disagreement about a URL is a
  parser-differential problem that skill treats generally, including cache keying.
- `hunting-broken-object-level-authorization` - cache deception discloses another user's authenticated
  response, reaching the same private data that skill pursues through direct authorization flaws.
- `auditing-error-handling-and-information-exposure` - a cached authenticated response is sensitive
  information reaching an unauthorized requester, the exposure that skill audits by another route.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the crafted static-looking URL, sink = the shared
  cache storing the authenticated response, evidence = a second requester retrieving the first user's private
  response from the cache on an isolated instance.
