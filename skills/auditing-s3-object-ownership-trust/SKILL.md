---
name: auditing-s3-object-ownership-trust
description: >-
  Audit object-storage ownership and per-object access for trust the bucket policy does not cover: an object
  uploaded by another account that keeps that uploader's ownership and ACL, a bucket where object ACLs still
  grant access despite a restrictive bucket policy, a cross-account write that lands an object the bucket
  owner cannot read or that carries a public grant, and a policy that scopes by prefix while an ACL on the
  object overrides it. Covers S3 and compatible stores where object ownership, object ACLs, and the bucket
  policy interact to decide who reads and controls each object. Use when a bucket receives objects from more
  than one principal and access is meant to be governed centrally. The cross-account or ACL-granted principal
  is the source, the object read or control is the sink, and the access the bucket policy did not intend is
  the bug.
license: MIT
---

# Auditing S3 object-ownership trust: when the object, not the bucket, decides access

Object stores let access be decided in more than one place, and that is where ownership trust breaks. A
bucket policy is the central control most teams reason about, but each object also has an owner and, unless
disabled, its own ACL, and a cross-account upload can land an object owned by the uploader rather than the
bucket owner. The result is objects the bucket owner cannot fully control, ACLs that grant access the bucket
policy meant to deny, and public grants riding on individual objects under a bucket that looks private. When
a bucket receives objects from more than one principal, central governance is an assumption, not a fact. You
audit these by checking who owns each object, whether object ACLs are still in force, and whether any object
grants access the bucket policy did not intend.

## When to use

- A bucket receives objects from more than one principal, including cross-account or third-party writers.
- Access is meant to be governed centrally by the bucket policy, but object ACLs may still be enabled.
- Cross-account writes may land objects the bucket owner does not own or cannot fully read or control.

## Scope check

Audit object ownership only in buckets and accounts you own or are authorized to assess, on non-production
data. Confirming access reads or controls real objects, so stay inside the authorized bucket and treat any
object contents as sensitive within scope. If you can't name the authorization, stop.

## The loop

1. **Establish the intended access model first.** Name who should read and control the objects in this bucket
   and how it is meant to be enforced: central bucket policy, single owner. This is the false-positive killer:
   a bucket with ownership enforced to the bucket owner and object ACLs disabled, governed only by the bucket
   policy, is correct. Name the intended model, then check whether ownership and ACLs match it.

2. **Determine object ownership across writers.** For objects written by more than one principal, confirm who
   owns each. A cross-account upload can retain the uploader's ownership, leaving objects the bucket owner
   cannot fully control. Check whether bucket-owner-enforced ownership is set so every object is owned
   centrally, or whether uploader ownership persists.

3. **Check whether object ACLs are in force.** If object ACLs are enabled, each object can grant access
   independently of the bucket policy. Determine whether ACLs are disabled (so the bucket policy is the sole
   control) or still active, and if active, whether any object ACL grants read or control to a principal, or to
   the public, that the bucket policy would deny.

4. **Reconcile the bucket policy with the per-object grants.** A restrictive bucket policy does not override an
   object ACL that grants access; the two combine. Compare the effective access on individual objects to what
   the bucket policy intends, looking for objects reachable by a principal the policy excludes because an ACL
   admits them.

5. **Check cross-account write consequences.** A cross-account writer can land an object the bucket owner
   cannot read, or attach a public-read grant, or create objects under a prefix the owner's policy assumed it
   controlled. Confirm that cross-account writes are constrained to grant ownership to the bucket owner and
   cannot carry an ACL that widens access.

6. **Confirm and record.** Confirm by reading or controlling an object as a principal the bucket policy
   excludes but an ACL or ownership admits, or by showing a cross-account write lands an owner-unreadable or
   publicly-granted object, within scope and without exfiltrating data. Kill the lead if ownership is
   bucket-owner-enforced, object ACLs are disabled so the bucket policy is the sole control, and cross-account
   writes cannot widen access. Record the admitted principal, the object read or control sink, and the access
   the bucket policy did not intend.

## Where object-ownership trust leaks

- **Object ownership can escape the bucket owner.** A cross-account upload may keep uploader ownership, leaving
  objects the bucket owner cannot fully control.
- **Object ACLs override a restrictive bucket policy.** If ACLs are enabled, an object grant admits access the
  bucket policy meant to deny; the two combine, not override.
- **A public object hides under a private bucket.** An ACL public grant on one object exposes it even when the
  bucket looks locked down centrally.
- **Cross-account writes can be unreadable to the owner.** An object the bucket owner does not own may be
  outside their read and lifecycle control.
- **Central governance is an assumption with multiple writers.** The moment more than one principal writes, who
  owns and who can grant per object must be verified, not assumed.

## Worked example (a confirm and a kill)

> **Confirm.** A shared ingest bucket receives uploads from a partner account, object ACLs are enabled, and the
> partner's objects retain partner ownership with an ACL that grants read to a broad group. An object the bucket
> policy meant to keep internal is readable by a principal the policy excludes, via the object ACL. **Confirmed**
> object-ACL access beyond the bucket policy, `high`, remediation = enable bucket-owner-enforced ownership to
> disable object ACLs and centralize control, and require cross-account writes to grant ownership to the bucket
> owner without an access-widening ACL.
>
> **Kill.** The bucket enforces bucket-owner ownership on every object, object ACLs are disabled so the bucket
> policy is the sole access control, and cross-account writers must grant full ownership to the bucket owner on
> upload. No object grants access the bucket policy excludes, and every object is owned and controlled centrally.
> **Killed**, `kill_reason` = "ownership bucket-owner-enforced with object ACLs disabled and cross-account writes
> ceding ownership; the bucket policy is the sole control and no object widens access."

## Rationalizations to reject

- *"The bucket policy is restrictive."* → An enabled object ACL combines with the policy and can grant access it
  denies; disable ACLs or reconcile every object.
- *"It is our bucket."* → A cross-account upload can land an object you do not own and cannot fully control;
  confirm ownership per object.
- *"Nothing is public."* → Check per-object ACLs; a public grant on a single object hides under a private-looking
  bucket.
- *"Only our partner writes to it."* → A partner is a separate principal; their objects' ownership and ACLs
  decide access unless you enforce bucket-owner ownership.
- *"We set the policy centrally."* → Central policy is not central control while object ACLs are enabled and
  cross-account ownership persists; verify both.

## Executing this in practice

You need the intended access model, the object ownership across writers, whether object ACLs are enabled, the
per-object grants reconciled against the bucket policy, and the cross-account write constraints. For the bucket,
confirm ownership is enforced and ACLs disabled, or reconcile every object's effective access. Reading the
ownership and ACL settings shows the intended control; reading an object as a policy-excluded principal shows
whether it holds.

## Related

- `auditing-presigned-url-scope-abuse` - the signed-access companion; presigned URLs and object ACLs are two
  ways access escapes the bucket policy on individual objects.
- `auditing-infrastructure-as-code-exposures` - the declared-configuration side; ownership enforcement and ACL
  disabling are settings that skill checks in the resource definitions.
- `auditing-multi-tenant-isolation` - a shared bucket across tenants is an isolation boundary; object ownership
  is how that boundary is enforced or broken per object.
- `mapping-attack-surface` - use it to find buckets that receive cross-account or third-party writes before
  auditing any one bucket's ownership.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the cross-account or ACL-granted principal, sink = the
  object read or control, evidence = the access the bucket policy did not intend.
