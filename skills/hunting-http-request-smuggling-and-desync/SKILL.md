---
name: hunting-http-request-smuggling-and-desync
description: >-
  Hunt for request smuggling where two HTTP processors on the same path disagree about where one request ends
  and the next begins: a front end and back end that resolve conflicting Content-Length and Transfer-Encoding
  headers differently, a proxy that forwards a body the origin re-parses, a keep-alive connection where a
  smuggled prefix poisons the next user's request, and a rewrite or normalization difference that desyncs the
  stream. Covers chained HTTP/1.1 processors, proxies, load balancers, and origins where request framing is
  parsed more than once. Use when a request crosses more than one HTTP parser and their framing agreement is
  the boundary. The desyncing request is the source, the poisoned next request or back-end path is the sink,
  and the framing disagreement between the two parsers is the bug.
license: MIT
---

# Hunting HTTP request smuggling and desync: when two parsers disagree on where a request ends

Request smuggling is a disagreement bug, not a parsing bug in isolation. When a request passes through more
than one HTTP processor, a proxy or load balancer in front of an origin, each one has to decide where one
request ends and the next begins, and if they decide differently, an attacker can hide a second request
inside the first. The classic lever is conflicting length signals: a Content-Length and a Transfer-Encoding
header, or two of either, that the front end and back end resolve in opposite orders. The consequence is
severe because these connections are reused: a smuggled prefix left on a keep-alive connection prepends to
the next user's request, so it poisons other people's traffic, bypasses front-end access controls, and
captures requests. The hunt is about the seam between two parsers, so you look for where their framing rules
differ. You hunt this by finding chained processors and testing whether a crafted request desyncs them.

## When to use

- A request passes through more than one HTTP/1.1 processor: a proxy or load balancer in front of an origin.
- The processors may resolve Content-Length, Transfer-Encoding, or duplicate framing headers differently.
- Keep-alive connections are reused across requests, so a desync poisons the next request on the connection.

## Scope check

Test smuggling only against systems you own or are authorized to assess, on non-production endpoints. A
successful desync poisons other users' requests on a shared connection, so use a dedicated test origin, keep
payloads benign, and never capture or alter real users' traffic. If you can't name the authorization, stop.

## The loop

1. **Establish the processor chain first.** Identify every HTTP processor on the path and their order: the
   front-end proxy or load balancer, any intermediate cache, and the origin. This is the false-positive killer:
   a single processor, or a chain that speaks a framing-unambiguous protocol end to end and rejects conflicting
   length signals, cannot desync. Name the chain, then look for framing disagreement between adjacent hops.

2. **Test conflicting length signals.** Send requests that carry both Content-Length and Transfer-Encoding, or
   duplicate copies of one, and observe whether the front end and back end agree on the body length. A
   difference, one honoring chunked encoding while the other honors the byte count, means a smuggled body
   crosses the seam. Probe the obfuscation variants (spacing, casing, malformed chunk markers) each hop handles
   differently.

3. **Distinguish the desync direction.** Determine whether the desync leaves bytes the back end treats as the
   start of the next request (a request prefix that prepends to the following request) or causes the front end
   to read the back end's response boundary wrong. The direction sets the impact: prefixing the next request
   poisons other users; a response desync captures or misroutes responses.

4. **Check normalization and rewrite differences.** Look for header rewriting, normalization, or
   request-line handling that one hop applies and the next re-parses: a header the proxy adds or strips, a path
   the origin re-normalizes, or a downgrade between protocol versions. A rewrite that changes framing after the
   front end has decided is a desync source even without duplicate length headers.

5. **Assess connection reuse and blast radius.** Confirm whether the back-end connection is shared across users
   or requests. A smuggled prefix on a reused connection affects whoever's request comes next, so shared reuse
   turns a self-inflicted oddity into an attack on other users, control bypass, or request capture. No reuse
   narrows the impact.

6. **Confirm and record.** Confirm on a dedicated test origin by showing a crafted request desyncs the chain,
   a smuggled prefix affecting a subsequent request you also control, without touching real users' traffic.
   Kill the lead if adjacent processors agree on framing, reject conflicting or duplicate length signals, and
   no rewrite changes framing across the seam. Record the desyncing request, the poisoned-next-request or
   back-end-path sink, and the framing disagreement between the two parsers.

## Where desync leaks

- **Conflicting length signals split the parsers.** A Content-Length and a Transfer-Encoding, or duplicates of
  either, resolved differently by two hops hide a second request in the first.
- **Obfuscation targets the weaker parser.** Spacing, casing, and malformed chunk markers that one hop
  tolerates and the next rejects create the disagreement.
- **A rewrite after the framing decision desyncs.** A header the proxy adds, strips, or normalizes that the
  origin re-parses can move the request boundary.
- **Connection reuse spreads the damage.** A smuggled prefix on a shared keep-alive connection prepends to the
  next user's request, so the victim is someone else.
- **A protocol downgrade re-frames the stream.** Translating between HTTP versions across the seam reintroduces
  ambiguous framing the endpoints handle differently.

## Worked example (a confirm and a kill)

> **Confirm.** A load balancer forwards to an origin, and a request carrying both a Transfer-Encoding chunked
> header and a Content-Length is treated as chunked by the balancer but length-delimited by the origin. On a
> dedicated test origin, the trailing bytes of the crafted request are parsed by the origin as the start of the
> next request, prefixing a follow-up request the tester also sends on the reused connection. **Confirmed** HTTP
> request smuggling via CL/TE disagreement, `high`, remediation = make the front end and origin resolve framing
> identically, reject requests carrying both Content-Length and Transfer-Encoding or duplicate length headers,
> and prefer a framing-unambiguous protocol between the hops.
>
> **Kill.** The front end and origin apply identical framing rules, any request carrying both Content-Length and
> Transfer-Encoding or duplicate length headers is rejected at the edge, chunked parsing is strict and matched
> across hops, and no rewrite alters framing after the front end's decision. A crafted dual-signal request is
> refused rather than split. **Killed**, `kill_reason` = "adjacent processors agree on framing and reject
> conflicting or duplicate length signals with no re-framing rewrite; no request desyncs the chain."

## Rationalizations to reject

- *"We are on HTTP/1.1 with a standard proxy."* → Standard proxies and origins routinely differ on duplicate or
  conflicting length headers; test the actual pair, do not assume agreement.
- *"The origin is internal."* → Smuggling attacks the seam between front end and origin, and the victim is the
  next request on the shared connection, not an external target.
- *"We reject malformed requests."* → Confirm both hops reject the same malformations; smuggling lives exactly
  where one is stricter than the other.
- *"Connections are not shared."* → Verify it; back-end keep-alive reuse is common, and it is what turns a
  desync into an attack on other users.
- *"There is only a CDN in front."* → A CDN is another parser in the chain; its framing rules versus the
  origin's are precisely the disagreement to test.

## Executing this in practice

You need the ordered processor chain, how each hop resolves Content-Length, Transfer-Encoding, and duplicate
framing headers, the rewrites and normalizations applied across each seam, and whether back-end connections are
reused. For each adjacent pair, test a conflicting-signal request and observe whether they agree on the body
boundary. Reading the proxy and origin configuration shows the intended framing; a controlled desync on a test
origin shows whether the seam holds.

## Related

- `auditing-http2-and-grpc-multiplexing-trust` - the HTTP/2 sibling; downgrade and stream-multiplexing framing
  reintroduce the same desync across a different protocol seam.
- `hunting-dns-rebinding-and-ssrf-pivots` - a smuggled request often targets an internal path; SSRF and
  smuggling both abuse trust in where a request appears to come from.
- `testing-web-cache-attacks` - cache poisoning is a common payload of a desync; the two share the
  proxy-versus-origin trust seam.
- `mapping-attack-surface` - use it to enumerate the proxy, cache, and origin hops on a path before testing
  their framing agreement.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the desyncing request, sink = the poisoned next
  request or back-end path, evidence = the framing disagreement between the two parsers.
