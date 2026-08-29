---
name: hunting-dns-rebinding-and-ssrf-pivots
description: >-
  Hunt server-side request forgery and DNS rebinding that turn a server into a proxy for the internal network:
  a URL or hostname a caller controls that the server fetches, a validation that checks the hostname once but
  connects later so a rebinding answer swaps it for an internal address, a redirect the fetcher follows into
  internal space, and a reached internal service (metadata endpoint, admin port, database) that trusts callers
  by network position. Covers server-side fetchers, webhooks, importers, and previewers that resolve and
  connect to a caller-influenced destination. Use when a server makes outbound requests to destinations a
  caller can influence and internal services trust the server's network position. The caller-controlled
  destination is the source, the internal service the server reaches is the sink, and the time-of-check
  rebinding or unvalidated fetch that reaches it is the bug.
license: MIT
---

# Hunting DNS rebinding and SSRF pivots: turning the server into a proxy for the inside

Server-side request forgery is a position bug: a server sits inside the network with a fetcher that will
connect to a destination the caller influences, and internal services around it trust callers by where they
come from rather than who they are. So if an attacker can steer the server's outbound request, the server
becomes their proxy into internal space, the cloud metadata endpoint, an admin port, a database that assumes
anyone who can reach it belongs. DNS rebinding is the technique that defeats the usual defense: the server
validates the hostname, sees a public address, and allows it, but between that check and the actual connection
the attacker's DNS answer changes to an internal address, so the connection lands somewhere the check never
approved. The same slip happens through a redirect the fetcher follows. The hunt is for caller-influenced
destinations and for the gap between validating and connecting. You hunt this by finding server fetchers and
testing whether the destination they connect to can be moved after the check.

## When to use

- A server fetches, imports, previews, or calls out to a URL or hostname a caller can influence.
- Validation of the destination may happen at check time while the connection resolves the name again later.
- Internal services (metadata endpoint, admin ports, datastores) trust callers by network position.

## Scope check

Test SSRF and rebinding only against systems you own or are authorized to assess, on non-production endpoints.
A confirmed pivot reaches real internal services and credentials, so stop at proof of reach, do not read or
exfiltrate internal data, and use benign markers. If you can't name the authorization, stop.

## The loop

1. **Establish the fetcher's intended destinations first.** Name where each server-side fetch is legitimately
   supposed to go: a specific allowed host, a public API, never internal space. This is the false-positive
   killer: a fetcher restricted to an allowlist that is re-checked at connect time, with internal services that
   authenticate rather than trust position, cannot be pivoted. Name the intended destinations, then find where
   the caller can influence them.

2. **Find caller-influenced destinations.** Locate every place a caller sets or shapes an outbound target: a
   URL parameter, a webhook or callback address, an import-from-URL, a link previewer, an avatar or document
   fetch. Each is an SSRF candidate; note whether the value is a full URL, a hostname, or an address the caller
   controls end to end.

3. **Check validation versus connection timing.** For each fetch, determine whether the destination is
   validated once and then resolved again at connect time. If the hostname is checked (public, allowed) but the
   socket resolves the name independently later, a rebinding DNS answer swaps the address between check and
   connect. A single resolution used for both check and connect closes this; two separate resolutions open it.

4. **Test redirects and address forms.** Check whether the fetcher follows redirects into internal space, and
   whether address encodings (alternate IP notations, IPv6 mappings, wildcard DNS names) slip past a
   blocklist. A validator that blocks obvious internal literals but follows a redirect to them, or misses an
   encoded form, is bypassed without rebinding at all.

5. **Map what the reached internal service grants.** For each internal destination the server can be steered
   to, determine what it exposes to a caller by position: the metadata endpoint yields node or instance
   credentials, an admin port yields control, an internal datastore yields data. This sets severity, and it is
   high whenever the reached service trusts the network rather than authenticating.

6. **Confirm and record.** Confirm by steering a server fetch to a benign internal marker you control, or
   demonstrating a rebinding or redirect reaches an internal address the check should have blocked, without
   reading real internal data. Kill the lead if destinations are allowlisted and re-checked at connect time,
   redirects into internal space are refused, encoded internal forms are blocked, and internal services
   authenticate rather than trust position. Record the caller-controlled destination, the internal-service
   sink, and the time-of-check rebinding or unvalidated fetch that reached it.

## Where SSRF and rebinding leak

- **Check-then-connect lets the name rebind.** Validating the hostname and resolving it again at connect time
  gives the attacker a window to swap in an internal address.
- **Followed redirects reach the inside.** A fetcher that validates the first URL but follows a redirect lands
  on the internal target the check refused.
- **Encoded addresses slip past blocklists.** Alternate IP notations, IPv6-mapped forms, and wildcard DNS names
  evade a validator that only blocks obvious literals.
- **The metadata endpoint is the prize.** A server steered to the cloud metadata endpoint hands back node or
  instance credentials to the caller.
- **Position-trusting services have no second gate.** An internal service that trusts callers by network
  location grants everything the moment the server reaches it.

## Worked example (a confirm and a kill)

> **Confirm.** A link-preview feature fetches a caller-supplied URL and validates that the hostname resolves to
> a public address, then the HTTP client resolves the name again to connect. Using an attacker-controlled domain
> whose DNS answer flips to an internal address after the check, the preview fetch connects to the cloud
> metadata endpoint and returns instance role credentials to the caller. **Confirmed** SSRF via DNS rebinding to
> the metadata endpoint, `critical`, remediation = resolve the destination once and connect to that exact
> validated address, block internal ranges including the metadata endpoint at connect time, refuse redirects
> into internal space, and require the metadata endpoint to use a session mechanism that a proxied fetch cannot
> satisfy.
>
> **Kill.** The fetcher resolves each destination once, validates that resolved address against an allowlist
> that excludes all internal ranges and the metadata endpoint, connects to that same address so no rebinding
> window exists, refuses redirects that leave the allowlist, normalizes and blocks encoded internal forms, and
> the internal services it can reach authenticate callers rather than trusting position. A rebinding domain is
> refused at connect time. **Killed**, `kill_reason` = "single-resolution allowlist enforced at connect time
> with no internal ranges, redirects and encoded forms blocked, and position-independent internal auth; the
> destination cannot be moved after the check."

## Rationalizations to reject

- *"We validate the hostname is public."* → If the socket resolves the name again to connect, a rebinding answer
  changes it after the check; validate and connect to one resolved address.
- *"We block internal IP literals."* → Confirm redirects and encoded forms are blocked too; a redirect or an
  alternate notation reaches the internal target without a literal.
- *"The metadata endpoint is protected."* → Confirm it requires a session token or header a proxied server fetch
  cannot supply; position alone must not authorize it.
- *"It only fetches user avatars."* → Any caller-influenced fetch is an SSRF vector; the feature's purpose does
  not limit where the URL points.
- *"Internal services are firewalled off."* → The server is inside the firewall; SSRF makes it the proxy, so
  internal services must authenticate, not trust the network.

## Executing this in practice

You need every caller-influenced outbound destination, whether validation and connection use one resolution or
two, the redirect and address-encoding handling, and what each reachable internal service grants by position.
For each fetch, test whether the connected address can be moved after the check. Reading the fetcher code shows
the intended destinations; steering a fetch to a benign internal marker shows whether the boundary holds.

## Related

- `mapping-pod-to-cloud-credential-reach` - the metadata endpoint SSRF reaches is that skill's subject; SSRF is
  one way a workload's position becomes cloud credentials.
- `auditing-webhook-authenticity-and-callback-trust` - webhook and callback URLs are prime SSRF sources; the
  two skills meet at the caller-supplied outbound address.
- `hunting-http-request-smuggling-and-desync` - a smuggled request can target an internal path much as SSRF
  does; both abuse where a request appears to originate.
- `adjudicating-taint-paths` - use it to confirm a caller-controlled value reaches the fetcher's connect call
  through validation and redirect indirection.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the caller-controlled destination, sink = the internal
  service the server reaches, evidence = the time-of-check rebinding or unvalidated fetch that reached it.
