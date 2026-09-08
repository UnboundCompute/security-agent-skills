---
name: auditing-mobile-backend-and-firebase-exposure
description: >-
  Audit the backend a mobile app talks to for authorization that lives only in the client, where a
  mobile-backend service, a hosted datastore, a storage bucket, or a cloud function trusts the app to enforce
  access and so lets any client read or write another user's data, call a privileged operation, or reach
  records it should not, because the service rules are permissive, the operation authorizes by client-set
  values, or the backend assumes only the genuine app calls it. Use when a mobile app uses a hosted backend
  or datastore whose access rules and operations you can exercise directly. Covers permissive datastore
  rules, unauthenticated or over-scoped reads and writes, storage bucket exposure, and client-trusting cloud
  functions. The direct client request bypassing the app is the source, the backend rule or operation serving
  it is the sink, and reaching data or actions the user is not entitled to is the bug.
license: MIT
---

# Auditing mobile backend and Firebase exposure: when the server trusts the app to say no

A mobile app usually talks to a hosted backend, a mobile-backend service, a cloud datastore, a storage bucket,
or a set of cloud functions, and it is easy to build that backend as though the app were the only client. The
app checks what the user may see and calls the backend accordingly, so the backend rules are left permissive
because the app is trusted to enforce access. But the app is not the only client: anyone can take the app's
credentials and endpoints and call the backend directly, skipping every check the app performs. When the
backend authorizes by rules that are too broad, by values the client sets, or by the assumption that only the
genuine app calls it, a direct request reads or writes another user's data, invokes a privileged operation,
or reaches records the user should never see. The bug is authorization that lives in the client while the
backend serves whoever asks. You audit these by calling the backend directly and checking whether the server,
not the app, enforces access.

## When to use

- A mobile app uses a hosted backend, datastore, storage bucket, or cloud functions you can call directly.
- The backend's access rules or operations may authorize by client-set values or assume only the app calls.
- Datastore rules or storage permissions may be permissive enough to serve another user's data.

## Scope check

Audit backend exposure only on services you own or are authorized to assess, on non-production or test
projects with test accounts, reading and writing only your own test data when you exercise a rule directly. A
confirmed gap reaches other users' data or privileged actions, so keep every probe within scope and prefer an
isolated project. If you can't name the authorization, stop.

## The loop

1. **Establish whether the backend enforces authorization itself or trusts the app first.** For each backend
   operation and datastore path, determine whether the server authorizes the request against the
   authenticated caller and its own rules, or whether it relies on the app to have checked and serves any
   direct request. This is the false-positive killer: a backend whose rules restrict every read and write to
   the entitled caller cannot be bypassed by calling it directly, whatever the app does. Name where
   authorization is enforced before crafting a direct request.

2. **Enumerate the backend surface.** List the datastore paths, storage locations, and cloud functions the app
   uses, with the credentials and endpoints it presents. These, recovered from the app or its traffic, are
   what a direct client will use to call the backend without the app.

3. **Check the datastore and storage rules.** Determine whether the datastore rules and storage permissions
   restrict each read and write to the entitled caller, or are permissive, world-readable paths, writes not
   scoped to the owner, or a bucket that lists or serves objects to any client. A permissive rule serves data
   directly.

4. **Check the operation authorization.** Determine whether each cloud function or backend operation
   authorizes by the authenticated caller and server-side state, or by values the client supplies, an
   identifier, a role, or an amount the client sets, so a direct caller sets them freely and reaches a
   privileged action.

5. **Check the only-the-app assumption.** Determine whether any part of the backend assumes the request comes
   from the genuine app, treating the app's credential or a client-set marker as authorization, when any
   client presenting the same values is served identically.

6. **Confirm and record.** Confirm by calling the backend directly, without the app, and reading or writing
   test data, or invoking a privileged operation, that the caller should not be entitled to, on an isolated
   project with test accounts. Kill the lead if the backend authorizes every read, write, and operation
   against the authenticated caller and its own rules, and if no operation trusts client-set authorization or
   the app's mere presence. Record the operation or path, the permissive rule or trusted value, and the direct
   access observed, or set a `kill_reason`.

## Where backend exposure leaks

- **Server-side enforcement is the finding.** A backend that authorizes each request itself is safe from a
  direct caller; the bug is authorization left in the app while the server serves whoever asks. Name where the
  check is missing.
- **Permissive datastore rules serve data directly.** A world-readable path or a write not scoped to the owner
  lets any client read or alter another user's records without the app.
- **Storage buckets list and serve to anyone.** A bucket that lists objects or serves them to any client
  exposes files the app would have gated, reachable by a direct request.
- **Client-set values are not authorization.** An operation that authorizes by an identifier, role, or amount
  the client supplies is driven by a direct caller who sets those values freely.
- **The only-the-app assumption is false.** Any client can present the app's credential and endpoints, so
  treating the app's presence as authorization serves every direct caller identically.

## Worked example (a confirm and a kill)

> **Confirm.** A cloud datastore path holding user records is readable by any authenticated client because the
> rule checks only that the caller is signed in, not that the record belongs to them. Calling the backend
> directly with a test account reads another test user's records without the app, on an isolated project.
> **Confirmed** broken authorization exposing other users' data through a permissive datastore rule, `high`,
> remediation = restrict each read and write to the record's owner in the server-side rules, authorize every
> operation against the authenticated caller, and never rely on the app to enforce access.
>
> **Kill.** The same datastore restricts each path to the owning caller in its server-side rules, storage
> objects are served only to their owner, and every cloud function authorizes by the authenticated caller and
> server-side state rather than client-set values. A direct request with a test account reaches only that
> account's own data. **Killed**, `kill_reason` = "the backend rules restrict every read, write, and operation
> to the entitled caller server-side; a direct request reaches nothing the caller is not entitled to."

## Rationalizations to reject

- *"Only our app calls the backend."* -> Anyone can extract the app's credentials and endpoints and call the
  backend directly; the app is not the only client, so the server must enforce access.
- *"The user is authenticated."* -> Authentication is not authorization; confirm the rule checks the caller is
  entitled to the specific record or operation, not merely signed in.
- *"The rules are the defaults."* -> Default rules are frequently permissive; read the actual rules and test a
  direct read and write, since defaults are exactly what this audit catches.
- *"The client sends its user id."* -> A direct caller sets any user id it likes; authorize by the
  authenticated identity the server establishes, not by a client-supplied value.
- *"The bucket is not linked anywhere."* -> An unlinked bucket that lists or serves objects to any client is
  still reachable by a direct request; confirm the permissions, not the obscurity.

## Executing this in practice

You need the backend surface the app uses, datastore paths, storage locations, and cloud functions, with its
credentials and endpoints, and the server-side rule or authorization for each. For each, call the backend
directly with a test account and decide whether it authorizes against the caller or serves the request on
client trust. Reading the rules settles some leads; calling the backend directly and reading or writing test
data the caller should not reach, on an isolated project, settles the rest.

## Related

- `hunting-broken-object-level-authorization` - the general direct-object authorization flaw, of which a
  permissive mobile-backend rule is the hosted-datastore case.
- `hunting-hybrid-app-bundle-and-config-exposure` - the backend keys and endpoints a direct caller uses are
  recovered from the bundle that skill unpacks, feeding this audit.
- `auditing-datastore-exposure-and-abuse` - the broader datastore exposure audit, sharing the permissive-rule
  and direct-access reasoning with this mobile-backend case.
- `hunting-mobile-tls-pinning-and-trust-gaps` - the backend credentials and endpoints are also recoverable
  from traffic when transport trust fails, the gap that skill covers.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the direct client request bypassing the app, sink =
  the backend rule or operation serving it, evidence = reaching test data or an action the caller is not
  entitled to by calling the backend directly on an isolated project.
