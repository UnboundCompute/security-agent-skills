---
name: testing-web-cache-attacks
description: >-
  Test how a caching layer between users and an application can be turned against
  it: cache poisoning (getting a harmful response stored and served to other users)
  and cache deception (tricking the cache into storing a victim's private response
  where the attacker can read it). Covers finding the cache key and unkeyed inputs,
  identifying cacheable responses, poisoning through unkeyed headers, and deceiving
  path-based caching into storing authenticated content. Use when reviewing a CDN, a
  reverse proxy, or any shared HTTP cache in front of an app.
license: MIT
---

# Testing web cache attacks: the bug is the key, not the app

A shared cache trades correctness for speed by serving one stored response to many
users, and that sharing is the vulnerability. If an attacker can influence what gets
stored (poisoning), everyone downstream receives their payload; if an attacker can
make the cache store someone else's private response under a key the attacker
controls (deception), they read data that was never theirs. Both live in the gap
between what the cache keys on and what actually determines the response.

## When to use

- You are reviewing a CDN, a reverse proxy, or any shared HTTP cache in front of an
  app.
- Responses that depend on headers or user identity are served through a cache.
- You are assessing whether authenticated content can be cached and re-served.

## Scope check

Test caches and apps you own or are authorized to test. Use benign markers and your
own accounts; do not poison responses served to real users or read real users' data.
If you can't name the authorization, stop.

## The loop

1. **Map the cache and its key.** Identify the caching layer in front of the app and
   determine the cache key: which parts of the request (path, query, some headers)
   decide whether two requests share a stored response. Everything the response
   depends on but the key ignores is an unkeyed input, and unkeyed inputs are where
   both attacks live.

2. **Find what is cacheable.** Determine which responses the cache stores and for how
   long: static-looking paths, extensions, explicit cache headers, and any response
   the cache decides to keep. A response that is cached but depends on request data
   the key omits is a poisoning candidate.

3. **Test cache poisoning through unkeyed input.** Send a request whose unkeyed header
   or parameter changes the response (a reflected header, a header that sets a link or
   script origin, a header that triggers an error page), then confirm the poisoned
   response is stored and served to a normal request without your input. If a later
   clean request receives your payload, the cache is poisoned for every user of that
   key.

4. **Test cache deception.** Request an authenticated, private page but shape the URL
   so the cache treats it as a static cacheable resource (appending a path segment or
   an extension the cache stores while the app still returns the private content). If
   the private response is stored under that URL, an attacker who requests the same
   URL retrieves the victim's data. Confirm what the cache keys on versus what the app
   authorizes on.

5. **Rate impact and record.** Poisoning that injects script or redirects affects
   every downstream user of the key (high to critical); deception that exposes another
   user's authenticated data is a direct confidentiality breach. Record confirmed
   poisoning and deception paths, and configurations where the cache key covers every
   response-affecting input and refuses to cache authenticated content (killed) in the
   schema.

## Where caches leak

- **The bug is the key, not the app.** The application can be correct while the cache
  serves the wrong response to the wrong person.
- **Unkeyed input is the whole game.** Any header or parameter that changes the
  response but not the key is a lever.
- **The cache and the app disagree on identity.** The app authorizes per user; the
  cache serves per key. Deception exploits that disagreement.
- **One stored response, many victims.** Poisoning scales to everyone sharing the
  key, which is what makes it severe.

## Worked example (a confirm and a kill)

> **Confirm.** A page reflects an unkeyed header into an absolute script URL. The
> attacker sends a request with that header pointing at their host; the cache stores
> the response and serves it, with the attacker-controlled script origin, to every
> subsequent visitor of that path. **Confirmed** cache poisoning, `critical`,
> remediation = include the header in the cache key or stop reflecting it, and do not
> cache responses that vary on unkeyed input.
>
> **Kill.** A profile page returns private, authenticated content with a directive
> that forbids shared caching, the cache refuses to store any authenticated response,
> and the cache key includes every input that changes the response. A deception
> attempt (appended static extension) is not stored because the response is marked
> non-cacheable. **Killed**, `kill_reason` = "authenticated responses are
> non-cacheable; cache key covers all response-affecting inputs; no unkeyed lever."

## Rationalizations to reject

- *"The app is authorized correctly."* → Authorization is per user; the cache serves
  per key. Deception bypasses the app entirely.
- *"It's just a caching layer."* → It decides what every user receives. A poisoned
  entry is a stored, shared payload.
- *"That header doesn't affect the key."* → If it affects the response, that is
  exactly the poisoning lever.
- *"Only static files are cached."* → Deception makes a dynamic private page look
  static. Verify what the cache actually stores.

## Executing this in practice

You need to observe the caching layer's behavior: the cache key, which responses are
stored, and how the app responds to unkeyed inputs and to deception-shaped URLs.
Comparing keyed versus unkeyed requests and reading the cache hit/miss and age
signals on responses is enough; the key-versus-response analysis is the method.

## Related

- `mapping-attack-surface` - locating the caching layers and shared infrastructure in
  front of the app.
- `testing-llm-insecure-output-handling` - a poisoned cached response is one more sink
  that serves attacker content.
- `writing-vuln-reports` - a poisoning or deception finding to a reproducible writeup.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the unkeyed input or
  deception URL, sink = the stored response served to a victim.
