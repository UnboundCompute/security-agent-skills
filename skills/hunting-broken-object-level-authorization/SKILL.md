---
name: hunting-broken-object-level-authorization
description: >-
  Hunt broken object-level authorization (BOLA, also called IDOR): endpoints that
  accept a client-supplied object reference - a numeric id, UUID, key, slug, filename,
  or an id nested in a request body or token - and read or mutate that object without
  checking the authenticated caller is entitled to it. Covers direct references,
  enumerable and guessable ids, references buried in nested or batch payloads,
  second-order ids stored then trusted later, and ownership checks that run on one path
  but not its siblings. Use when reviewing any API or handler that fetches or changes a
  record by an id the client controls. The reference is the source, the data access is
  the sink, and the missing owner binding is the bug.
license: MIT
---

# Hunting broken object-level authorization: follow the id to the access

The most common serious API bug is also the simplest: a handler takes an object
reference from the client, fetches or mutates that object, and never checks that the
caller is allowed to touch *this* object. Authentication passes, the route filter
passes, the query runs, and user A reads user B's record by changing one id. The bug is
not in the routes that are protected; it is in every access that trusts a client-supplied
reference without binding it to the caller. You find it by following the reference from
the request to the data access and asking, at the access, what proves this is the
caller's object.

## When to use

- You are reviewing an API or handler that fetches or changes a record by an
  id, key, slug, filename, or reference the client supplies.
- References travel in the path, query, body, a nested field, a batch item, or a token.
- Access control depends on the object belonging to the caller, not just on the caller
  being logged in.

## Scope check

Test object access only in systems you own or are authorized to test, with test
accounts across at least two users and, where relevant, two tenants. If you can't name
the authorization, stop.

## The loop

1. **Enumerate every client-supplied object reference.** Inventory each endpoint that
   accepts an object identifier and where it arrives: path segment, query parameter,
   body field, an id nested inside a JSON object or array, a batch of ids, a value
   carried in a token or cookie, or a filename. Each reference that selects an object is
   a source. Do not stop at the obvious path id; references hide in nested and batch
   payloads.

2. **Follow each reference to the data access.** Trace the reference from where it
   enters to where it selects the object: the query filter, the key lookup, the file
   open, the mutation. That access is the sink. The question the whole audit turns on is
   asked here, so find the exact access each reference reaches.

3. **Ask what binds the object to the caller.** At the access, is there a predicate that
   ties the object to the authenticated caller, scoping by owner, tenant, or an explicit
   membership check? A lookup keyed only by the client's id (`find(id)`) with no
   `owner == caller` constraint is unbound. Authentication proves who the caller is;
   it does not prove the object is theirs.

4. **Check the siblings and the write side.** An ownership check on `GET /object/{id}`
   means nothing if `PUT`, `DELETE`, the export path, the batch endpoint, or a newer
   sibling handler reaches the same object without it. Enumerate every access to the
   object type and confirm each one binds the caller; the unbound peer is the bug, and
   write and bulk paths are where it hides.

5. **Test the reference substitution.** Confirm by requesting another user's or tenant's
   object: swap the id for one you do not own, walk enumerable or guessable ids, embed a
   foreign id in a nested or batch field, or replay a second-order id captured earlier.
   A response that returns or mutates the other party's object is the proof. A predictable
   id (sequential, timestamped) raises severity but is not required; an unbound access
   with random ids is still the bug once you have a valid foreign reference.

6. **Confirm and record.** Confirm with the cross-user or cross-tenant access that
   succeeds, capturing request and response. Kill the lead if every access to the object
   binds it to the caller at the data layer, including writes, batches, and exports.
   Record with the exact endpoint, the reference field, and the foreign object reached.

## Where object authorization leaks

- **The reference is trusted, not the caller's right to it.** The check that must exist
  is at the access, binding the object to the caller. Its absence is the whole bug.
- **Ownership on read, nothing on write.** Read paths get the check; `PUT`, `DELETE`,
  bulk, and export paths reach the same object unbound. Audit every verb.
- **Nested and batch references escape the pass.** A single path id is easy to see; an
  id three levels into a body, or one item in an array of a hundred, is where the
  unbound access lives.
- **Second-order references defeat the front check.** An id validated on the way in,
  stored, then used later on a different path is only as safe as that later access.
- **Random ids are not access control.** Unguessable references slow enumeration; they
  do not bind the object to the caller. One leaked or shared reference and the access
  fires.

## Worked example (a confirm and a kill)

> **Confirm.** `GET /api/invoices/{id}` scopes the query to the caller. The sibling
> `POST /api/invoices/export` takes `{"ids": [...]}` and streams each invoice with a
> lookup keyed only by id, no owner predicate. Logged in as user A, exporting user B's
> invoice ids returns B's invoices. **Confirmed** BOLA on the batch export path, `high`,
> remediation = filter every id in the batch by the caller's ownership at the query, and
> add a test that the export path rejects foreign ids.
>
> **Kill.** Every access to the record type, across `GET`, `PUT`, `DELETE`, the batch
> endpoint, and the export, runs through one repository method whose query is scoped to
> the caller's tenant and owner id; nested and second-order references pass through the
> same method. Cross-user and cross-tenant substitution returns nothing on every path.
> **Killed**, `kill_reason` = "single owner-scoped access path used by every reference,
> including writes, batch, and second-order; no unbound peer."

## Rationalizations to reject

- *"The id is a random UUID, no one can guess it."* → Unguessable is not unbound. The
  access still needs an owner check; one shared or leaked reference fires it.
- *"The route requires authentication."* → Authentication is who, not which. The bug is
  reaching another user's object as a valid user.
- *"The read path checks ownership."* → And the write, batch, and export paths? Audit
  every access to the object, not the one that is protected.
- *"The id is validated on input."* → Validated for shape, or for ownership? A well-formed
  foreign id passes shape validation and reaches the object.
- *"It is an internal endpoint."* → Internal to whom? If an authenticated user reaches
  it, the object binding still has to hold.

## Executing this in practice

You need every endpoint that accepts an object reference and where the reference enters,
a trace from each reference to the exact data access it reaches, and test accounts across
users and tenants to substitute references. A dataflow view that carries a client-supplied
reference to the query or mutation it selects, and shows which accesses apply an ownership
predicate and which do not, turns this from endpoint-by-endpoint clicking into a coverage
pass over every object access; the follow-the-reference method is the same by hand.

## Related

- `auditing-declarative-authorization` - the coverage-and-correctness view of authz as
  config; this skill is the per-request object-reference dataflow underneath it.
- `hunting-mass-assignment-and-property-authz` - the sibling bug where the client controls
  which *fields* of an object it may write, not which object it may reach.
- `auditing-guard-gaps` - the unguarded-peer pattern, here the access that skips the owner
  check its siblings apply.
- `hunting-business-logic-flaws` - authorization of outcomes reached through a permitted
  path, above the object-access layer.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the client-supplied reference,
  sink = the data access it reaches, evidence = the cross-user or cross-tenant response.
