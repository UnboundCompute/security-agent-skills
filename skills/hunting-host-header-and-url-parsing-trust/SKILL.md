---
name: hunting-host-header-and-url-parsing-trust
description: >-
  Hunt trust placed in the Host or a forwarded host header and in inconsistently parsed URLs, where the app
  builds absolute links, keys a cache, routes a request, or matches an allowlist from a header or a parsed
  URL an attacker can influence or that two components parse differently. Use when a service derives its own
  external URL from the request, when a cache or router keys on the host, or when a security decision parses
  a URL. Covers password-reset and verification-link poisoning, cache poisoning, routing to internal virtual
  hosts, and redirect or fetch allowlist bypass through parser differentials. The attacker-influenced host
  or ambiguous URL is the source, the link builder, cache key, router, or allowlist parse is the sink, and
  trusting a host or a parse the attacker controls is the bug.
license: MIT
---

# Hunting host header and URL parsing trust: when the app believes the request about its own name

An application often needs to know its own address, to put a link in an email or to decide where a request
belongs. The easy way is to read it from the incoming request: the Host header, or a forwarded-host header a
proxy added. But those are attacker-controlled, and a server that builds a password-reset link from the Host
sends the victim a link pointing at the attacker's domain. A shared cache that keys on a header the origin
reflects serves the poisoned response to everyone. A router that trusts a forwarded host reaches an internal
virtual host. The related trap is URL parsing: when the component that checks a URL and the component that
uses it disagree about where the host ends, an allowlist passes a URL that resolves somewhere else. The bug
is trusting the request's account of the host, or one parser's reading of a URL, as if it were authoritative.
You find it by tracing where self-URLs, cache keys, routes, and allowlists get their host.

## When to use

- The app builds absolute URLs (reset, verification, invite links) from the request's Host or forwarded host.
- A shared cache or a router keys on, or routes by, the Host or a forwarded-host header.
- A redirect or fetch allowlist parses a URL, and different components may parse that URL differently.

## Scope check

Test host and URL-parsing trust only against systems you own or are authorized to assess, on non-production
infrastructure, using a benign host you control and benign markers rather than poisoning a live cache or
sending real reset links to third parties. A confirmed cache-poisoning or reset-link case reaches other
users, so keep every probe in scope and prefer an isolated instance. If you can't name the authorization,
stop.

## The loop

1. **Establish where the host actually comes from first.** For each place the app names itself, keys a cache,
   routes, or matches an allowlist, determine whether it uses a fixed configured base URL and a trusted
   parser, or derives the host from the incoming Host or forwarded header and parses the URL ad hoc. This is
   the false-positive killer: a service that builds links from a configured base and routes on a trusted
   value cannot be poisoned through the header, while one that reflects the request host can. Name the source
   of the host before crafting a request.

2. **Map every host consumer.** Trace the Host and any forwarded-host header, and every parsed URL, into
   absolute-link builders, cache keys, routing decisions, and redirect or fetch allowlist checks. Each
   consumer that takes the host from the request or a differently-parseable URL is a candidate sink.

3. **Separate header trust from parser differentials.** One class is trusting the header value: the app copies
   the Host or forwarded host into a link, a cache key, or a route. The other is a URL that the checking
   parser and the using parser read differently, so the host the allowlist approves is not the host the fetch
   or redirect reaches. Decide which class each sink belongs to.

4. **Follow the consumer to its impact.** A poisoned self-URL in a reset or verification email points the
   victim at the attacker; a header reflected into a cached response poisons every client on that key; a
   trusted forwarded host routes to an internal virtual host the client should not reach; a parser
   differential passes an allowlist and resolves to an internal or attacker address. Identify which impact the
   sink yields.

5. **Check the defense that actually holds.** The reliable controls are building external URLs from a fixed
   configured base rather than the request, validating the Host against an allowlist of known hostnames,
   trusting forwarded headers only from a known proxy and stripping client-supplied ones at the edge, keying
   caches on the values that actually vary the response, and parsing URLs once with a single strict parser
   whose result both the check and the use share. Determine which stands at each sink.

6. **Confirm and record.** Confirm by sending a request with a crafted host or ambiguous URL and observing the
   poisoned link in the outbound message, the reflected host in a cached response, the internal route, or the
   allowlist passing a URL that resolves elsewhere, on an isolated instance with benign markers. Kill the lead
   if the sink uses a configured base and a validated host, if forwarded headers come only from a trusted
   proxy, or if one strict parser governs both check and use. Record the header or URL, the sink, the class,
   and the impact, or set a `kill_reason`.

## Where host and parsing trust leaks

- **Self-URLs built from the request are poisonable.** A reset or verification link built from the Host or a
  forwarded header points wherever the attacker sets that header, delivering an attacker link in a trusted
  email.
- **Caches key on the wrong thing.** A shared cache that keys on a header the origin reflects into the body
  stores the attacker's value and serves it to everyone on that key.
- **Forwarded headers are client input.** Unless stripped at a trusted edge, a client can supply a
  forwarded-host that the app treats as authoritative for routing or link building.
- **Parser differentials split check from use.** When the allowlist parser and the fetching or redirecting
  parser disagree about the host, the approved host and the reached host differ; the URL must be parsed once.
- **Routing on host reaches internal vhosts.** A router that selects a backend by host can be steered to an
  internal virtual host by a crafted Host, exposing services meant to be unreachable.

## Worked example (a confirm and a kill)

> **Confirm.** A password-reset flow builds the reset link from the incoming Host header. A request with the
> Host set to a domain the tester controls causes the reset email to contain a link to that domain carrying
> the victim's reset token, on an isolated instance with a test mailbox. **Confirmed** password-reset
> poisoning through host-header trust, `high`, remediation = build the reset URL from a fixed configured base
> URL, validate the Host against an allowlist of known hostnames, and strip client-supplied forwarded-host
> headers at a trusted edge.
>
> **Kill.** The same flow builds the reset link from a configured base URL, validates the incoming Host
> against an allowlist and rejects unknown hosts, trusts a forwarded host only from the known proxy, and the
> redirect allowlist parses each URL once with a strict parser shared by check and fetch. A crafted Host is
> rejected and the email link always points at the configured domain. **Killed**, `kill_reason` = "external
> URLs come from a configured base with a validated Host, forwarded headers are trusted only from the edge
> proxy, and one strict parser governs the allowlist; the attacker cannot influence the host the app uses."

## Rationalizations to reject

- *"The Host header is set by the browser."* -> A client, not a browser, sends the request, and any value can
  be placed in the Host or a forwarded header; treat it as untrusted input, not a fact about your deployment.
- *"The proxy sets the forwarded host."* -> Only if the edge strips the client's version first; otherwise a
  client-supplied forwarded header reaches the app and is trusted. Confirm the edge overwrites it.
- *"Our URL parser is standard."* -> The question is whether the same parser governs both the check and the
  use; two standard parsers can still disagree on userinfo, backslashes, or empty hosts, splitting the two.
- *"The cache only keys on the path."* -> If the origin reflects a header into the body but the cache does not
  key on it, that header poisons every response on the path; align the key with what varies the response.
- *"Internal vhosts are not routable."* -> If routing selects the backend by host, a crafted Host can select
  an internal one; confirm the router does not trust the request's host for backend selection.

## Executing this in practice

You need every builder of a self-URL, every cache key and routing decision, and every allowlist parse, with
the source of the host each uses and whether one strict parser governs both check and use. For each, decide
whether the attacker can set the host through a header or split the parse through an ambiguous URL, and what
the consumer then does. Reading where the host and the parse come from settles most leads; a crafted host or
ambiguous URL producing a poisoned link, a reflected cache entry, an internal route, or an allowlist bypass on
an isolated instance settles the rest.

## Related

- `auditing-open-redirect-and-forced-navigation` - the redirect allowlist that a parser differential defeats
  is the sink that skill audits, so URL-parsing trust and open redirect meet on the target check.
- `testing-web-cache-attacks` - a header reflected into a cached response is the poisoning primitive that skill
  treats, and the cache-key analysis is shared.
- `hunting-unicode-normalization-and-canonicalization-bypass` - encoded or ambiguous hosts that split check
  from use are a canonicalization failure that skill analyzes directly.
- `hunting-dns-rebinding-and-ssrf-pivots` - a fetch allowlist beaten by a parser differential feeds the
  server-side request forgery reach that skill covers.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the attacker-influenced host or ambiguous URL, sink =
  the link builder, cache key, router, or allowlist parse, evidence = the poisoned link, reflected cache
  entry, internal route, or allowlist bypass observed on an isolated instance.
