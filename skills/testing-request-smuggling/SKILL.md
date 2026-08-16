---
name: testing-request-smuggling
description: >-
  Test whether a chain of HTTP servers disagrees about where one request ends and
  the next begins, letting an attacker smuggle a request past the front end into the
  back end. Covers front-end and back-end desync from conflicting length signals,
  connection-reuse poisoning, single-packet and timing detection, and adjacent
  boundary confusion where a proxy and origin parse framing differently. Use when
  reviewing a reverse proxy, load balancer, CDN, or any multi-hop HTTP path where two
  parsers sit in series. The bug is a disagreement between parsers, not one flaw.
license: MIT
---

# Testing request smuggling: the bug is between two servers

When two HTTP servers sit in series and parse a request's boundaries differently, the
front end forwards what it thinks is one request while the back end sees two, and the
smuggled second request is attributed to the next client on the reused connection.
That desync lets an attacker prefix a victim's request, bypass front-end controls,
poison the connection, and capture other users' traffic. The bug is a disagreement
between parsers, not a single flaw in one.

## When to use

- You are reviewing a reverse proxy, load balancer, CDN, or gateway in front of an
  origin.
- Requests pass through more than one HTTP parser in series.
- Connections between hops are kept alive and shared across clients.

## Scope check

Test chains you own or are authorized to test, and keep every probe on your own
traffic. Do not target other users' live requests. If you can't name the
authorization, stop.

## The loop

1. **Map the server chain and the reused connections.** Identify every hop a request
   passes through (edge, proxy, balancer, cache, origin) and where connections between
   hops are kept alive and shared across clients. A desync only bites where a
   downstream connection is reused for another user, so map that reuse first.

2. **Find conflicting length signals.** A request's body length can be stated more
   than one way, and smuggling happens when two hops resolve a conflict differently:
   one honors one signal, the other honors the other. Check whether the chain
   normalizes or rejects conflicting or duplicated length signals, or passes them
   through for the next hop to reinterpret.

3. **Probe for desync safely.** Using benign, self-targeted probes, test whether a
   crafted boundary makes the back end treat trailing bytes as the start of the next
   request, observable as a delayed or mis-attributed response. Prefer timing-based
   and single-packet detection that confirms the split without harming other users. A
   measurable difference between "rejected" and "the next request was affected" is the
   signal.

4. **Escalate to impact on your own traffic.** If a split is confirmed, demonstrate
   the concrete effect against your own second request: prefixing it, bypassing a
   front-end path restriction, or capturing the response to a request you control. Do
   not target other users' live traffic; prove the primitive on requests you own.

5. **Check adjacent boundary confusion.** Does the chain hand off to another protocol
   or downgrade a connection where the two ends parse framing differently (a proxy and
   origin disagreeing on a message boundary)? The same parser-disagreement class
   extends beyond one protocol version. Note any hop that reframes requests.

6. **Rate and record.** A confirmed desync that bypasses controls or captures
   cross-user requests is critical; a reliably rejected boundary is a kill. Record the
   exact conflicting signals and the observed split, and recommend the fix: normalize
   or reject ambiguous framing at the edge, prefer one connection per client to the
   origin, and reject conflicting length signals outright.

## Where the chain desyncs

- **The vulnerability is between two servers.** Neither is wrong alone; they disagree,
  and the disagreement is the bug.
- **Connection reuse is what makes it dangerous.** Without a shared downstream
  connection, a split affects only you.
- **Ambiguous framing should be rejected, not resolved.** Any hop that helpfully
  reinterprets a conflicting boundary is a desync source.
- **Detection can be safe.** Timing and single-packet methods confirm a split without
  poisoning real users.

## Worked example (a confirm and a kill)

> **Confirm.** An edge proxy and the origin resolve a conflicting length signal
> differently; a crafted request leaves trailing bytes the origin treats as the next
> request. A benign timing probe shows the following self-owned request is prefixed by
> the smuggled bytes, and a front-end-blocked path becomes reachable through the
> smuggled request. **Confirmed** request smuggling, `critical`, remediation = reject
> conflicting length signals at the edge, normalize framing, and avoid sharing origin
> connections across clients.
>
> **Kill.** The edge rejects any request with duplicated or conflicting length signals
> with a 400, normalizes all forwarded requests to a single unambiguous framing, and
> uses a fresh origin connection per client. Every desync probe is rejected and no
> trailing bytes reach the origin as a new request. **Killed**, `kill_reason` =
> "ambiguous framing rejected and normalized at the edge; no shared downstream
> connection to poison."

## Rationalizations to reject

- *"Both servers are standards-compliant."* → Compliance still leaves boundary
  ambiguities they can resolve differently. Test the pair, not each alone.
- *"There's a filter at the edge."* → Smuggling bypasses the edge by hiding the request
  from it. The control it enforces is exactly what gets skipped.
- *"We use modern HTTP."* → Newer versions have their own desync and reset classes. The
  parser-disagreement problem moves, it does not vanish.
- *"We couldn't reproduce it once."* → Desync can be timing-sensitive. Use single-packet
  and timing detection before calling it clean.

## Executing this in practice

You need to see the server chain, control a client that can send precisely-framed
requests, and observe response timing and attribution on connections you own. Any
tooling that sends byte-exact requests and measures timing works; the
connection-reuse map and the conflicting-signal analysis are the method. Keep every
probe on your own traffic.

## Related

- `mapping-attack-surface` - enumerating the proxy, cache, and origin hops in the
  chain.
- `testing-web-cache-attacks` - a smuggled response can poison a shared cache entry.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the ambiguously-framed
  request, sink = the back-end request it splits into or the control it bypasses.
