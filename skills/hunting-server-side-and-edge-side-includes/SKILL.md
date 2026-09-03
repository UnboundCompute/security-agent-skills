---
name: hunting-server-side-and-edge-side-includes
description: >-
  Hunt server-side include and edge-side include injection, where untrusted input is reflected into a
  document that a processor later interprets for include and exec directives. When an origin server has
  server-side includes enabled, a reflected directive can read a server file or run a command. When a
  cache, content-delivery network, or proxy in front of the origin parses edge-side include tags, a
  reflected tag can fetch an internal URL for server-side request forgery, poison a shared cache, or copy
  a victim's cookie into a page. The edge case is easy to miss because the origin developer never sees the
  processor that evaluates the tag. Use when reflected input reaches a document that an include-capable
  server or edge tier processes. The reflected input is the source, the include or exec directive the
  processor evaluates is the sink, and the resulting file read, command, request forgery, or cache
  poisoning is the bug.
license: MIT
---

# Hunting server-side and edge-side includes: when a directive is reflected

Two different tiers can turn reflected text into an executed directive. On the origin, a server with
server-side includes enabled scans documents for directives and will read a file or run a command when it
finds one, so untrusted input reflected into such a document and interpreted as a directive becomes file
disclosure or command execution. At the edge, a cache, content-delivery network, or reverse proxy may
parse edge-side include tags in responses and act on them, fetching a URL, splicing a fragment, or reading
a request variable, so a reflected tag becomes server-side request forgery into the internal network, a
poisoned shared cache, or a page that copies the victim's cookie back to the attacker. The edge variant is
insidious: the application developer never wrote include handling and cannot see it in the origin code,
because a separate tier performs it. You find both by locating reflection into processed documents and
determining which processor sits in the path.

## When to use

- Untrusted input is reflected into a page, template, or fragment served by an include-capable server.
- A cache, content-delivery network, or reverse proxy in front of the origin may parse edge-side includes.
- Response surfaces (headers, error pages, cached fragments) echo input into documents processed downstream.

## Scope check

Test include injection only against systems you own or are authorized to assess, on non-production
infrastructure, because a confirming directive can read server files, reach internal services, or poison a
cache shared by other users. Prefer benign, observable probes over destructive commands, and coordinate
before touching a shared cache. If you can't name the authorization, stop.

## The loop

1. **Establish that an include processor is actually in the path first.** Determine whether the origin has
   server-side includes enabled for the served document type, or whether a downstream cache, delivery
   network, or proxy parses edge-side include tags. This is the false-positive killer: if no processor
   interprets includes, reflected directive text is inert markup and there is no bug. Name the processor
   and the tier that runs it before treating any reflection as a lead.

2. **Find the reflection point.** Locate where untrusted input is echoed into a document the processor will
   scan: page body, a template fragment, an error page, a header value copied into a cached response.
   Confirm the input reaches the document before the processor runs, not after.

3. **Check for neutralization before the processor.** Read whether the reflected value is entity-encoded,
   whether the include syntax's delimiters are stripped or escaped, and whether the document type is one
   the processor ignores. Encoding that turns the directive's brackets into inert entities before the
   processor sees them closes the class; determine whether it is applied on this path.

4. **Test the server-side directive impact.** Where the origin processes includes, probe with a benign
   directive that reflects an observable value (a document variable, a size) and, only within scope,
   assess whether file-include and command-exec directives are permitted. The reachable impact is exactly
   the directive set the server allows.

5. **Test the edge-side directive impact.** Where the edge parses tags, probe whether an include tag makes
   the edge fetch a URL you control or an internal address (request forgery), whether the result is stored
   in a shared cache (poisoning that affects other users), and whether request variables such as cookies
   can be read into the page (token theft). Confirm which behaviors the specific edge honors.

6. **Confirm and record.** Confirm server-side by reflecting a benign directive that returns an observable
   value, and edge-side by causing an interaction with a host you control or an observable cache change, on
   an isolated instance or an isolated cache key. Kill the lead if no include processor is in the path, if
   the reflected value is encoded so directive delimiters are inert before processing, or if the processor
   ignores the served document type. Record the tier, the processor, the reflection point, the directive,
   and the effect. Set `kill_reason` when killing.

## Where include injection leaks

- **The edge processor is invisible to the origin.** The application code has no include handling, so a
  source-only review of the origin misses an edge-side vulnerability performed by a separate tier.
- **Reflection into cached responses is shared.** A directive spliced into a response stored under a shared
  cache key affects every user who receives that cached entry.
- **Headers and error pages are reflection points too.** Input echoed into a header the edge copies into the
  body, or into a verbose error page, reaches the processor just as a normal page does.
- **Edge includes reach the internal network.** An include tag that fetches a URL runs from the edge or
  origin's vantage, making it a request-forgery primitive against internal services.
- **Encoding must precede the processor.** Output encoding applied after the include tier has already run
  does not help; the neutralization has to happen before the directive is scanned.

## Worked example (a confirm and a kill)

> **Confirm.** A delivery tier in front of the application parses edge-side include tags, and a search
> page reflects the query into the response body without encoding the tag delimiters. A query containing an
> include tag pointed at a host the tester controls causes the edge to fetch that host, and pointing it at
> an internal metadata address returns internal data into the page; the response is stored under a shared
> cache key. **Confirmed** edge-side include injection to server-side request forgery and cache poisoning,
> `high`, remediation = disable edge-side include parsing on responses built from user input, or
> entity-encode tag delimiters at the origin before the response reaches the edge.
>
> **Kill.** An error page reflects the requested path, but the origin has server-side includes disabled for
> the served type and no cache or proxy in the path parses edge-side includes; the value is additionally
> entity-encoded. A reflected directive renders as visible inert text. **Killed**, `kill_reason` = "no
> include processor runs on this path at either tier and the delimiters are entity-encoded, so the reflected
> directive is never interpreted."

## Rationalizations to reject

- *"We do not use server-side includes."* -> The origin may not, but a cache or delivery network in front of
  it can parse edge-side includes; confirm the whole path, not just the application server.
- *"The tag just shows up as text."* -> That means this document type or tier does not process it; verify the
  processor, because a different surface or content type may be parsed.
- *"Output is encoded."* -> Only if the encoding runs before the include processor; encoding applied after the
  edge has already interpreted the tag is too late.
- *"It is only reflected, not stored."* -> Reflection into a response cached under a shared key becomes stored
  for every user who reads that cache entry.
- *"Internal URLs are not reachable."* -> An edge-side fetch runs from inside the perimeter; that is exactly
  the request-forgery vantage that reaches internal services.

## Executing this in practice

You need the served document types and whether the origin has includes enabled, the presence and behavior
of any cache, delivery network, or proxy that parses edge-side includes, every reflection point into
processed documents, and the encoding applied before each processor. For each reflection, decide which tier
interprets includes, whether delimiters survive to the processor, and what a directive could read, fetch,
or poison. Reading the config and code shows the processor and encoding; a benign directive probe shows
whether it is interpreted.

## Related

- `testing-web-cache-attacks` - the shared-cache poisoning that an edge-side include can cause is that
  skill's core concern; the include tag is one way input reaches a cached response.
- `hunting-reflected-and-stored-xss` - the same reflection points feed a browser sink there and an include
  processor here; check both interpreters for a given reflection.
- `hunting-host-header-and-url-parsing-trust` - an edge-side fetch is a request-forgery primitive; that skill
  covers where such internal requests are trusted.
- `adjudicating-taint-paths` - use it to connect the reflection to the processor across the origin and edge
  tiers.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the reflected untrusted input, sink = the include
  or exec directive the processor evaluates, evidence = a benign directive returning an observable value or
  an out-of-band interaction on an isolated instance.
