---
name: auditing-presigned-url-scope-abuse
description: >-
  Audit presigned object-storage URLs for scope that grants more than the request intended: a signature
  that covers a broader key, prefix, or bucket than the user should reach, an overlong expiry, a method or
  content-type left unconstrained, or a signer identity whose permissions exceed the caller's. Covers
  presigned GET and PUT URLs for S3 and compatible stores, where the signed policy is the only boundary
  once the URL leaves the server, and where an attacker who edits the key, reuses the URL, or uploads a
  different object escapes the intended scope. Use when a service mints presigned URLs so clients read or
  write storage directly. The caller-influenced key or policy input is the source, the signing call is the
  sink, and the signed scope wider than the caller's entitlement is the bug.
license: MIT
---

# Auditing presigned URL scope abuse: when the signature grants more than the request

A presigned URL turns a server-side permission into a standalone credential: once minted, it grants
whatever the signature covers to anyone holding the URL, with no further authorization check at the store.
That is convenient and precisely why it leaks. If the service signs a key the caller supplied without
confirming the caller owns it, the URL reads or writes another user's object. If the signed policy leaves
the key prefix, the HTTP method, the content type, or the expiry looser than the request needed, the holder
does more than intended: reads a sibling object, overwrites a path they were only meant to read, or replays
the URL long after it should have died. The signer's own identity matters too, because the URL inherits its
permissions, not the caller's. You audit these by comparing the signed scope against the caller's actual
entitlement.

## When to use

- A service mints presigned object-storage URLs so clients read or write a store directly.
- The object key, prefix, bucket, or upload parameters in a presign request come from the caller.
- Presigned URLs are returned to browsers or third parties where they can be inspected, edited, or replayed.

## Scope check

Test presigned-URL handling only against storage and applications you own or are authorized to assess, on
non-production data. A confirming request reads or writes real objects, so stay inside the authorized bucket
and account and never touch another tenant's data outside an owned test fixture. If you can't name the
authorization, stop.

## The loop

1. **Establish the caller's real entitlement first.** Before inspecting any signature, determine what object
   or prefix this authenticated caller is actually allowed to read or write. This is the false-positive
   killer: a presigned URL scoped exactly to what the caller owns is correct no matter how powerful presigning
   looks. Name the entitlement, then compare the signed scope to it.

2. **Trace caller input into the signed key and policy.** Follow the key, prefix, bucket, method, and
   content-type from the request into the signing call. The critical question is whether the service confirms
   the caller owns the requested key before signing, or signs whatever key it is handed. A signer that trusts
   a caller-supplied key lets the caller name another user's object and receive a valid URL for it.

3. **Compare the signed scope to the entitlement.** Read what the signature actually covers: is it pinned to
   one exact key, or does it permit a prefix or wildcard that reaches siblings? Is the method constrained to
   the one operation needed (GET for a read, PUT for an upload), or does it allow both? A URL that grants more
   keys, more methods, or a different bucket than the caller's entitlement is over-scoped.

4. **Check the expiry and the replay window.** A presigned URL is valid for its whole lifetime to anyone
   holding it. An expiry measured in hours or days when the operation needs seconds turns a one-time action
   into a durable credential that survives in logs, browser history, and referrer headers. Confirm the expiry
   matches the operation's real need.

5. **Confirm the signer identity is least-privilege.** The URL inherits the signing principal's permissions.
   If the service signs with a role that can read or write the whole bucket, every presigned URL is a
   potential whole-bucket credential regardless of the intended object. Determine whether the signer's own
   permissions are scoped to what presigned URLs should ever grant.

6. **Confirm and record.** Confirm by requesting a presign for an object the caller does not own and showing
   the returned URL succeeds against it, or by editing a signed key or replaying an expired-in-intent URL
   within scope, without touching data outside an owned fixture. Kill the lead if the service verifies key
   ownership before signing, pins the signature to the exact key and method needed, sets an expiry matching
   the operation, and signs with a least-privilege identity. Record the input, the signing call, and the
   scope beyond entitlement.

## Where presigned URL scope abuse leaks

- **A caller-supplied key signed without an ownership check is direct object access.** The service becomes a
  signing oracle for any key the caller names.
- **A prefix or wildcard scope reaches siblings.** Signing a prefix instead of the exact key lets the holder
  enumerate and fetch neighboring objects.
- **An unconstrained method turns a read grant into a write.** A signature that permits PUT when the caller
  needed GET lets the holder overwrite the object.
- **An overlong expiry is a lingering credential.** The URL keeps working from logs, history, and shared
  links long after the operation completed.
- **The URL carries the signer's power, not the caller's.** A broadly permissioned signing role makes every
  URL as strong as that role over the covered scope.

## Worked example (a confirm and a kill)

> **Confirm.** An avatar-download endpoint signs a GET URL for a key taken from the request without checking
> the key belongs to the caller. Requesting a presign for another user's object key returns a valid URL that
> fetches that object. **Confirmed** presigned URL scope abuse to cross-user object read, `high`, remediation
> = verify the requested key resolves to an object the authenticated caller owns before signing, pin the
> signature to that exact key and to GET, and sign with a role scoped to the user-content prefix only.
>
> **Kill.** The service resolves the object from the caller's own identity, never from a caller-supplied key,
> signs the exact resolved key for the single required method with a two-minute expiry, and signs with a role
> permitted only on the user-content prefix. A presign request naming another user's key is rejected before
> signing. **Killed**, `kill_reason` = "key derived from the caller's identity, signature pinned to the exact
> key and method with a short expiry, signer scoped to the user prefix; no URL grants beyond the caller's
> entitlement."

## Rationalizations to reject

- *"The client only sends its own key."* → The attacker is not using your client. Verify ownership of the
  requested key on the server before signing.
- *"The URL expires."* → Confirm the expiry matches the operation's seconds-long need; an hours-long window is
  a durable credential in logs and history.
- *"It is just a read URL."* → Check the signed method; a signature that also permits PUT is a write grant,
  and a prefix scope reaches other objects.
- *"Signing is internal."* → The signed URL leaves your boundary the moment it is returned; its scope is the
  only remaining control.
- *"The signer role is our standard service role."* → The URL inherits that role's power; a whole-bucket
  signer makes every URL a whole-bucket credential.

## Executing this in practice

You need each presign endpoint, the caller's real entitlement, which of the key/prefix/bucket/method/
content-type come from the caller, what the signature actually covers, the expiry, and the signer identity's
permissions. For each endpoint, compare the signed scope to the entitlement and confirm ownership is checked
before signing. Reading the signing call shows the intended scope; requesting a URL for an unowned key shows
whether the boundary holds.

## Related

- `hunting-broken-object-level-authorization` - the same missing ownership check, here expressed through a
  signed URL rather than a direct object reference.
- `auditing-s3-object-ownership-trust` - the bucket-side companion: who owns and can read the objects a
  presigned URL points at.
- `hunting-non-human-identity-and-secret-reachability` - the signer identity is a machine credential; its
  scope determines how much every presigned URL can grant.
- `adjudicating-taint-paths` - use it to confirm a caller-supplied key reaches the signing call without an
  intervening ownership check.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the caller-influenced key or policy input, sink =
  the signing call, evidence = the signed scope beyond the caller's entitlement.
