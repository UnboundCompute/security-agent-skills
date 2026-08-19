---
name: auditing-graphql-attack-surface
description: >-
  Audit the attack surface a GraphQL API exposes that a plain endpoint does not: schema
  introspection left open, unbounded query depth and recursion, aliasing and field
  duplication that multiply cost, query batching that defeats rate limits and enables
  brute force, field-level authorization that a resolver skips even when the object check
  passed, and mutations reached without the guard their action needs. Covers the query
  and variables as the source, the resolver and the data or work it triggers as the sink,
  and the missing depth, cost, batch, or field guard as the bug. Use when reviewing a
  GraphQL schema, its resolvers, or a gateway that fronts one. Introspection and cost
  limits are one audit; per-field and per-mutation authorization is the other.
license: MIT
---

# Auditing GraphQL attack surface: the schema is the map, the resolver is the guard

A GraphQL API gives the client a query language over your data graph, and with it a set
of failure modes a fixed endpoint never had. The client, not the server, decides the
shape and depth of each request, so cost is client-controlled. The schema describes every
type and field, so leaving introspection open hands an attacker the map. And authorization
now lives per field and per resolver, so a check that held at the object level can be
skipped one field deeper. Two audits sit here: the surface and cost audit (what the client
can see and how expensive it can make a request), and the authorization audit (whether
every field and mutation enforces the access its data needs).

## When to use

- You are reviewing a GraphQL schema, its resolvers, or a gateway that fronts one.
- The client controls query shape, depth, aliasing, or batching.
- Authorization is enforced in resolvers, per field or per mutation, not only at a route.

## Scope check

Audit a GraphQL surface only in systems you own or are authorized to test, with accounts
across roles and tenants and permission to send shaped and batched queries. If you can't
name the authorization, stop.

## The loop

1. **Recover the schema and the surface.** Determine what the client can learn: is
   introspection enabled, do field suggestions leak type names on error, is there a
   published schema? Recover the types, fields, and mutations reachable. The schema is the
   attacker's map; if it is open, inventory it, and if it is closed, note whether errors
   still leak it piecemeal.

2. **Map query cost to the client.** Trace how a query's shape drives work: nested and
   recursive selections that fan out, a relation that can be walked in a cycle, aliasing
   that requests the same expensive field many times in one query, and connection fields
   with no page bound. The client chooses depth and repetition, so the ceiling on cost is
   whatever the server does not cap.

3. **Check the depth, complexity, and rate limits.** Is there a maximum query depth, a
   complexity or cost budget, a cap on aliases, and a page-size limit on lists? Absent any
   of these, a single crafted query is a denial-of-service, and batching or aliasing
   multiplies it. Confirm the limits exist and bind before the resolvers run.

4. **Check batching against rate and brute-force limits.** Can one request carry many
   operations, or one query alias the same mutation dozens of times? If rate limiting
   counts requests rather than operations, batching defeats it, turning one request into a
   password-spray, coupon-brute-force, or enumeration engine. The unit the limit counts is
   the bug.

5. **Audit authorization per field and per mutation.** For each sensitive field and every
   mutation, is there a resolver-level check that the caller may read that field or perform
   that action, or does authorization stop at the object or the route? A type whose object
   check passes can still expose a field (another user's email, an internal flag) whose
   resolver skips the check, and a mutation reached in a batch can skip a guard the primary
   path applied. The unguarded field or mutation is the bug.

6. **Confirm and record.** Confirm a surface issue by recovering the schema or driving cost
   with a shaped query; confirm an authorization issue by reading a field or invoking a
   mutation you should not, across users or tenants. Kill the lead if introspection is
   closed, depth and cost and batch are bounded, and every field and mutation enforces its
   own authorization. Record with the exact query and the schema, cost, or data it exposed.

## Where a GraphQL surface leaks

- **Introspection open in production is the map handed over.** It is not a bug alone, but
  it turns every other weakness into a targeted one.
- **Cost is client-controlled unless you cap it.** Depth, recursion, aliasing, and unpaged
  lists each multiply work; without a complexity budget one query is an outage.
- **Batching changes the unit the limit must count.** Per-request rate limits are meaningless
  when one request carries a hundred operations. Count operations, not requests.
- **Authorization moved to the field.** An object-level check does not cover a field resolver
  that fetches deeper; per-field and per-mutation checks are where horizontal access hides.
- **Errors leak the schema a closed introspection hid.** Field suggestions and verbose type
  errors rebuild the map one message at a time.

## Worked example (a confirm and a kill)

> **Confirm.** Introspection is disabled, but the `user` type's `email` field resolver
> fetches the address with no check that the caller is that user or an admin. Querying
> `user(id: <other>) { email }` returns another user's email; aliasing the field across a
> batch dumps many at once. **Confirmed** field-level authorization gap with batch
> amplification, `high`, remediation = enforce per-field authorization in the `email`
> resolver and count batched operations against the rate limit.
>
> **Kill.** Introspection is off in production and errors do not suggest fields; query depth,
> complexity, and alias count are bounded before resolvers run; lists are paged; the rate
> limiter counts operations, so batching gains nothing; every sensitive field and every
> mutation enforces its own caller check, verified across users and tenants. Shaped, batched,
> and cross-user queries all fail. **Killed**, `kill_reason` = "closed introspection, bounded
> depth/complexity/batch, operation-counted limits, per-field and per-mutation authorization."

## Rationalizations to reject

- *"Introspection is off, the schema is secret."* → Field suggestions and error messages
  leak it, and a determined client rebuilds it. Treat the schema as known and guard the
  resolvers.
- *"We rate-limit the endpoint."* → Per request or per operation? Batching turns one request
  into many; the limit has to count operations.
- *"Authorization is handled at the route."* → GraphQL has one route. The check has to live in
  the resolver, per field and per mutation.
- *"Nobody would write a query that deep."* → The client writes the query. Depth and cost are
  yours to cap, not theirs to spare.
- *"It is only a read."* → A read of a field the caller may not see is a data-exposure bug, and
  aliasing makes it a bulk one.

## Executing this in practice

You need the schema (recovered or provided), the resolver map from each field and mutation to
the data or work it triggers, the server's depth/complexity/alias/page and rate limits, and
accounts across roles and tenants to test field and mutation access. A view that traces a query
field to its resolver and the data access beneath it, and flags which resolvers apply an
authorization check and which do not, turns the authorization half into a coverage pass; the
surface-and-cost half is the schema recovery and the shaped-query test.

## Related

- `mapping-attack-surface` - the general recon this specializes for a GraphQL schema and its
  resolvers.
- `hunting-broken-object-level-authorization` - the object-reference bug, here reached through
  a query argument instead of a path id.
- `hunting-mass-assignment-and-property-authz` - the mutation-input analog: which fields a
  mutation lets the client write.
- `hunting-business-logic-flaws` - batching and replay as an abuse of permitted operations at
  scale.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the query and variables, sink = the
  resolver and the data or work it triggers, evidence = the schema, cost, or field exposed.
