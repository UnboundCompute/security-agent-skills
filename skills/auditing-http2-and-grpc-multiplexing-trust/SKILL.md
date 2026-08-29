---
name: auditing-http2-and-grpc-multiplexing-trust
description: >-
  Audit HTTP/2 and gRPC edges for framing and multiplexing trust that breaks when a stream is translated or
  reused: an h2c or HTTP/2-to-HTTP/1.1 downgrade that reintroduces request smuggling, pseudo-header and header
  handling that lets a stream forge its path or authority, multiplexed streams on one connection whose
  authentication or rate limit is applied per connection rather than per stream, and a gRPC gateway that
  trusts metadata or a method name a caller controls. Covers HTTP/2 front ends, gRPC services, and gateways
  that translate between protocols. Use when an edge terminates or downgrades HTTP/2 or multiplexes gRPC calls
  and per-stream trust is assumed. The crafted stream or metadata is the source, the back-end request or
  method it reaches is the sink, and the downgrade or per-connection trust that admits it is the bug.
license: MIT
---

# Auditing HTTP/2 and gRPC multiplexing trust: when a stream is not what the connection claims

HTTP/2 and gRPC move the unit of a request from a connection to a stream: one connection carries many
interleaved streams, each with its own headers, pseudo-headers, and path. Trust decisions that were written
for one-request-per-connection quietly break here. A downgrade seam, HTTP/2 translated to HTTP/1.1 at the
origin, or cleartext h2c smuggled past a front end, reintroduces the framing ambiguity of request smuggling
in a protocol that was supposed to have solved it. Pseudo-headers like the authority and path are attacker-set
per stream, so a stream can claim a path or host the front end never authorized. And per-connection controls,
authentication established once, a rate limit counted per connection, are undercounted when many streams share
the connection. A gRPC gateway adds method names and metadata a caller controls. The audit asks whether trust
is enforced per stream and whether translation preserves the framing. You audit this by testing each stream
and each downgrade seam rather than trusting the connection.

## When to use

- An edge terminates, downgrades, or translates HTTP/2, including h2c or HTTP/2-to-HTTP/1.1 to an origin.
- gRPC calls are multiplexed on shared connections and authentication or rate limits may be per connection.
- A gRPC gateway trusts caller-supplied metadata, method names, or pseudo-headers to route or authorize.

## Scope check

Test HTTP/2 and gRPC edges only on systems you own or are authorized to assess, on non-production endpoints. A
downgrade smuggling proof can poison other streams or requests, so use a dedicated test origin and keep
payloads benign. If you can't name the authorization, stop.

## The loop

1. **Establish the intended per-stream trust first.** Name what each stream should be allowed to do and how
   authentication, path authorization, and rate limits are meant to apply: per stream, on verified attributes.
   This is the false-positive killer: an edge that authenticates and authorizes every stream independently on
   attributes it verifies, and downgrades without reintroducing framing ambiguity, is correct. Name the
   intended per-stream trust, then test against it.

2. **Test the downgrade and translation seam.** Where HTTP/2 is translated to HTTP/1.1 or h2c is accepted,
   test whether the translation reintroduces request smuggling: a stream whose translated framing the origin
   parses differently, or an h2c upgrade smuggled past a front end that only inspects HTTP/1.1. The translation
   is a parser seam, so the smuggling discipline applies here in HTTP/2 clothing.

3. **Check pseudo-header and authority handling.** For each stream, confirm the edge validates the authority and
   path pseudo-headers and does not let a stream forge a host or path the connection was not authorized for. A
   stream that sets an authority pointing at an internal service, or a path that bypasses front-end routing
   rules, is using per-stream attributes to escape connection-level trust.

4. **Check per-stream versus per-connection enforcement.** Determine whether authentication, authorization, and
   rate limits are applied per stream or once per connection. Authentication established at connection setup and
   then trusted for every stream lets one authenticated connection carry streams that should be checked
   individually; a rate limit counted per connection is bypassed by multiplexing many streams. Confirm the
   controls count and check streams, not connections.

5. **Check gRPC gateway metadata and method trust.** For a gRPC gateway, confirm it does not authorize or route
   on caller-controlled metadata or the method name without verification: metadata a client sets is not an
   identity, and a method name is a request, not a permission. Trace whether attacker-supplied metadata reaches
   an authorization decision or a back-end call.

6. **Confirm and record.** Confirm on a dedicated test origin by downgrading a stream into a smuggled request,
   forging an authority or path that reaches an unauthorized back end, exhausting a per-connection rate limit
   with multiplexed streams, or authorizing on forged metadata, without touching real traffic. Kill the lead if
   the downgrade preserves framing, pseudo-headers are validated, trust is enforced per stream, and gateway
   metadata is not trusted. Record the crafted stream or metadata, the back-end request or method sink, and the
   downgrade or per-connection trust that admitted it.

## Where multiplexing trust leaks

- **A downgrade reintroduces smuggling.** HTTP/2-to-HTTP/1.1 translation or accepted h2c re-creates the framing
  disagreement across the translation seam.
- **Pseudo-headers forge path and authority.** A stream sets its own authority and path, so weak validation
  lets it claim a host or route the connection was not authorized for.
- **Per-connection auth trusts every stream.** Authentication established once and applied to all streams on the
  connection skips per-stream checks that should run individually.
- **Per-connection rate limits undercount streams.** A limit counted per connection is bypassed by multiplexing
  many streams on one connection.
- **Gateway metadata and method names are caller input.** gRPC metadata a client sets is not an identity, and a
  method name is a request, not an authorization.

## Worked example (a confirm and a kill)

> **Confirm.** A front end accepts an h2c cleartext upgrade it does not fully inspect and forwards to an origin
> that re-parses the stream as HTTP/1.1. On a dedicated test origin, a stream smuggled through the upgrade is
> parsed by the origin as a separate request reaching an internal path the front end would have blocked.
> **Confirmed** HTTP/2 downgrade smuggling to an unauthorized back end, `high`, remediation = reject or fully
> inspect h2c upgrades at the edge, ensure HTTP/2-to-HTTP/1.1 translation preserves framing unambiguously, and
> apply path authorization to the translated request at the origin.
>
> **Kill.** The edge rejects unexpected h2c upgrades, translates HTTP/2 to the origin with unambiguous framing
> the origin parses identically, validates the authority and path pseudo-headers on every stream against the
> allowed routes, authenticates and rate-limits per stream rather than per connection, and the gRPC gateway
> authorizes on verified identity rather than caller metadata. A crafted stream reaches only what it is
> authorized for. **Killed**, `kill_reason` = "downgrade preserves framing, pseudo-headers validated per stream,
> auth and limits enforced per stream, and gateway metadata not trusted; no stream escapes its authorized
> scope."

## Rationalizations to reject

- *"HTTP/2 fixed request smuggling."* → Only end to end; a downgrade or h2c seam to an HTTP/1.1 origin brings
  the framing ambiguity right back.
- *"The connection is authenticated."* → Authentication per connection is not per stream; confirm each stream is
  checked, not just the connection it rides.
- *"We rate-limit the connection."* → Multiplexing many streams on one connection bypasses a per-connection
  limit; count streams.
- *"The gateway passes our metadata."* → Client-set metadata is caller input, not identity; confirm authorization
  rests on verified attributes, not what the caller sent.
- *"The path is set by the client, that is normal."* → The authority and path pseudo-headers are attacker-set
  per stream; validate them or a stream forges its route.

## Executing this in practice

You need the downgrade and translation points, how framing is preserved across each, the pseudo-header
validation, whether authentication, authorization, and rate limits are per stream or per connection, and the
gRPC gateway's metadata and method trust. For each stream and seam, test whether trust holds independently.
Reading the edge and gateway configuration shows the intended per-stream trust; a downgrade smuggle or a
per-connection bypass on a test origin shows whether it holds.

## Related

- `hunting-http-request-smuggling-and-desync` - the HTTP/1.1 foundation; the downgrade seam here is that same
  desync reached through HTTP/2.
- `auditing-grpc-service-authorization` - the gRPC authorization companion; this skill covers the multiplexing
  and gateway-metadata edge, that one the per-method authorization.
- `auditing-service-mesh-mtls-and-authz-trust` - meshes carry gRPC over HTTP/2; per-stream trust and mesh
  authorization are the same question at different layers.
- `mapping-attack-surface` - use it to find HTTP/2 edges, h2c listeners, and gRPC gateways before testing their
  per-stream trust.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the crafted stream or metadata, sink = the back-end
  request or method it reaches, evidence = the downgrade or per-connection trust that admitted it.
