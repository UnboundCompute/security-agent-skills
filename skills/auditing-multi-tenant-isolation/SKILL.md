---
name: auditing-multi-tenant-isolation
description: >-
  Audit whether every data operation is scoped to the caller's tenant, so a request in one tenant cannot
  read or write another's data. Covers a query or object lookup with the object identifier but no tenant
  predicate, a tenant taken from client-controlled input at the operation rather than the authenticated
  session, scoping applied on the list path but dropped on the detail, update, delete, or export path, a
  cache or storage key with no tenant segment, and a background job, report, or privileged connection
  that runs across tenants or bypasses the mandatory scope. Frames isolation as a systemic invariant,
  not a single-object reference bug, which a separate skill covers. Use when reviewing data access in a
  system that serves multiple tenants. The tenant used at the operation is the source, the tenant-scoped
  data operation is the sink, and a sink scoped by anything but the authenticated tenant is the bug.
license: MIT
---

# Auditing multi-tenant isolation: is every operation scoped to the caller's tenant

When one system holds many tenants' data, isolation is an invariant that has to hold everywhere at
once: every query, cache read, storage access, search, and background job is scoped to the tenant the
request is authenticated as, and to no other. A single unscoped operation is not one bug but a whole
class of cross-tenant access at that point. This is the systemic view, distinct from asking whether one
object reference is owner-checked; here the question is whether the boundary is enforced by
construction across the codebase. You audit it by finding the tenant value at each data operation and
tracing it back: does it come from the authenticated session, or from something the client controls,
and is there a mandatory scope that covers this path at all. The decisive false positive is a central,
invisible enforcement, a global scope or a row-level policy, so establish that first.

## When to use

- One deployment or database serves multiple tenants, organizations, or workspaces, separated by a scope rather than by isolation.
- Data operations take an object identifier, a tenant selector, or run in a background job or export.
- You want to know whether any authenticated tenant can reach another tenant's data.

## Scope check

Test cross-tenant access only in systems you own or are authorized to assess, using tenants you
control. A confirmed bleed exposes another customer's data, so use only test tenants and coordinate
disclosure. If you can't name the authorization, stop.

## The loop

1. **Establish the enforcement model first.** Determine how tenant scoping is meant to be enforced: a
   mandatory global scope or base query that injects the tenant on every call, a database row-level policy
   bound to a per-request tenant, or per-query discipline where each query must add its own predicate.
   Which model is in force decides what an unscoped-looking query means, so settle it before flagging
   anything.

2. **Find operations with no tenant predicate.** Under a per-query model, look for a query or object lookup
   that filters by the object identifier but carries no tenant condition, and its equivalent through a data
   layer that does not inject one. This is the most common bleed: the row is fetched by identifier alone, so
   any tenant that can name the identifier reads it.

3. **Find the tenant sourced from client input.** Look for a tenant taken at the operation from a path
   segment, query parameter, body field, header, or subdomain and used to scope the query, with no
   reconciliation against the authenticated session. Scoping to an attacker-chosen tenant is scoping in
   appearance only. Include the case where the session tenant is resolved but the query still uses the
   client value or a default.

4. **Find the asymmetric paths.** Look for scoping that is present on one path and dropped on a sibling: a
   list endpoint that filters by tenant while the detail, update, delete, or export endpoint for the same
   resource does not. Asymmetry is a strong true-positive signal, because it shows the team knew the filter
   and missed a path.

5. **Find the shared keys and the boundary-crossing jobs.** Look for a cache, storage, or search key built
   without a tenant segment, so one tenant's value serves or poisons another's, and for a background job,
   scheduled report, export, or privileged or admin connection that queries across all tenants or runs
   under credentials that bypass the mandatory scope. Async and privileged paths are where scoping is most
   often forgotten.

6. **Confirm and record.** Confirm by acting as tenant A, naming or influencing a resource, key, or job
   belonging to tenant B, and showing the operation reads or writes B's data. Kill the lead if a mandatory
   global scope or a row-level policy provably covers this path and is not bypassed here by a raw query or
   an unscoped or admin connection, if a client tenant selector is reconciled against the session before
   use, if a cross-tenant job is intentional and carries its own authorization, or if the deployment is
   single-tenant. Record the operation, the tenant source, and the cross-tenant data reached.

## Where tenant isolation leaks

- **One unscoped operation is a class of cross-tenant access.** Isolation is an all-points invariant; a
  single query without the tenant predicate opens every object it can name.
- **A client-supplied tenant is not a scope.** Filtering by a tenant the caller chose is theater; the scope
  has to come from the authenticated session, and any client selector reconciled against it before use.
- **Asymmetry is the tell.** A list path that scopes and a detail or export path that does not proves the
  filter exists and was missed; that gap is the finding.
- **Async and privileged paths forget the tenant.** Background jobs, exports, and admin or service
  connections routinely run across tenants or bypass the mandatory scope; scoping must be re-established
  from the job's owner and the connection must not bypass the policy.
- **A cache or storage key without a tenant segment bleeds sideways.** A key of only an object identifier
  serves one tenant's value to another and lets one poison the other's.

## Worked example (a confirm and a kill)

> **Confirm.** A resource detail endpoint fetches the record by the identifier in the path with no tenant
> condition, and there is no global scope or row-level policy in force; the list endpoint for the same
> resource does filter by the session tenant. Acting as tenant A, requesting an identifier owned by tenant
> B returns B's record. **Confirmed** cross-tenant read through an unscoped detail path, `high` rising to
> `critical` where identifiers are enumerable, remediation = enforce tenant scoping centrally through a
> mandatory scope or a row-level policy so every operation, including this one, is bound to the
> authenticated tenant.
>
> **Kill.** The detail query carries no explicit tenant predicate, but every request sets a per-connection
> tenant from the verified session and a row-level policy on the table restricts rows to that tenant; this
> path uses the standard scoped connection, not an admin one, and no raw-query or unscoped sibling bypasses
> it, and a client tenant selector elsewhere is reconciled against session membership before use. Acting as
> tenant A cannot read tenant B's record. **Killed**, `kill_reason` = "tenant boundary enforced at the query
> layer by a row-level policy bound to the verified session, no bypassing connection or raw query on this
> path, and client selectors reconciled before use."

## Rationalizations to reject

- *"The query has no tenant filter, so it bleeds."* -> Not if a mandatory global scope or a row-level policy
  covers it. Establish the enforcement model before flagging an unscoped-looking query.
- *"We scope by the tenant in the URL."* -> That tenant is the caller's to choose. Scoping by it is scoping
  to the attacker unless it is reconciled against the authenticated session.
- *"The list endpoint is scoped, so the resource is safe."* -> The detail, update, delete, and export paths
  are separate. Asymmetry is exactly where the bleed hides.
- *"It is a background job, it runs as the system."* -> Then it likely bypasses the scope. Re-establish the
  tenant from the job's owner and run it under scoped, not bypassing, credentials.
- *"The identifiers are unguessable, so no one can cross."* -> Unguessability is defense-in-depth, not the
  control. Add the tenant predicate; do not rely on the identifier being secret.

## Executing this in practice

You need the enforcement model in force, the data operations and whether each carries or inherits a
tenant scope, the origin of the tenant value at each operation, the list-versus-detail and read-versus-
write symmetry for each resource, the cache and storage key shapes, and the background jobs and
privileged connections. For each operation, decide whether it is bound to the authenticated tenant by
something the caller cannot change. Reading the model and the operation tells you whether a scope covers
it; acting as one tenant against another's resource tells you whether the boundary holds.

## Related

- `hunting-broken-object-level-authorization` - the single-object manifestation of this systemic property;
  a tenant-scoping failure is a whole class of it at once, and the two audits complement each other.
- `auditing-declarative-authorization` - the mandatory scope or policy this audit checks for is often a
  declarative rule; both ask whether the central control actually covers every path.
- `adjudicating-taint-paths` - the discipline of tracing the tenant value from the operation back to its
  source and confirming reachability before calling a gap a bug.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the tenant used at the operation, sink = the
  tenant-scoped data operation, evidence = one tenant reading or writing another's data.
