---
name: auditing-api-key-and-token-lifecycle
description: >-
  Audit the lifecycle of API keys and access tokens for weaknesses that let one keep working past its intended
  bounds: a key issued with broader scope than the caller needs so a leak grants far more than one function, a
  key or token with no expiry that stays valid indefinitely, a revocation path that does not actually stop the
  key so a rotated or compromised credential keeps authenticating, a key that leaks into logs, URLs, client-side
  code, or error messages and is never rotated, and a token whose scope or audience is not enforced on use so it
  works against endpoints it was never meant for. Use when a long-lived programmatic credential authenticates a
  caller and the bounds on that credential (scope, expiry, revocability) are the boundary. The over-scoped,
  unexpiring, or leaked key is the source, the access it grants past its intended bounds is the sink, and the
  missing scope limit, expiry, or working revocation is the bug.
license: MIT
---

# Auditing API key and token lifecycle: a key is a standing credential, so bound it and be able to kill it

An API key or long-lived access token is a bearer credential: whoever holds it is the caller, with no second
factor and often no user behind it. That makes the whole security of the key a question of its lifecycle, how
narrowly it is scoped, whether it expires, whether you can actually revoke it, and where it might leak, because
the moment one of those is loose, a single leaked string becomes standing access. A key issued with broader
scope than its caller needs means a leak grants far more than the one function it was for. A key with no expiry
stays valid forever, so a credential stolen once works until someone notices, which is often never. A revocation
path that does not truly stop the key, a rotation that leaves the old key live, a disable that a cache ignores,
means a compromised credential keeps authenticating after you believed you killed it. A key that leaks into
logs, URLs, client-side code, or error messages and is never rotated is a published password. And a token whose
scope or audience is not enforced on use works against endpoints it was never issued for. The audit follows each
key from issuance to revocation and checks it is minimally scoped, expiring, truly revocable, and enforced on
use. You audit this by holding a key and testing what it can reach, how long it lasts, and whether you can stop
it.

## When to use

- A service issues API keys or long-lived access tokens that authenticate programmatic callers.
- Keys may be issued with broad scope, no expiry, or without a working revocation path.
- A key may leak into logs, URLs, or client code, or a token's scope or audience may not be enforced on use.

## Scope check

Test key and token lifecycle only against services and credentials you own or are authorized to assess, with
test keys. Issuing, exercising, and revoking keys changes real access state, so use test credentials and never
use, rotate, or revoke a key that is not yours. If you can't name the authorization, stop.

## The loop

1. **Establish each key's intended bounds first.** For every kind of key or token, name the minimal scope its
   caller needs, its intended lifetime, how it is meant to be revoked, and where it is supposed to live. This is
   the false-positive killer: a key issued with least-privilege scope, a bounded expiry, a revocation path that
   provably stops it, and no exposure in logs or URLs is behaving correctly. Name the intended bounds, then test
   each one.

2. **Test scope against need.** Compare what each key can actually do against what its caller needs. Exercise the
   key against endpoints and operations outside its stated purpose and confirm they are refused. A key that
   carries broad or account-wide scope for a narrow task means any leak of it grants far more than the one
   function, so confirm keys are minimally scoped.

3. **Test expiry.** Determine whether each key or token has an expiry and confirm it is enforced: use a key past
   its intended lifetime and confirm it is rejected. A key with no expiry stays valid indefinitely, so a
   credential stolen once works forever; confirm long-lived keys have a bounded, enforced lifetime.

4. **Test revocation and rotation.** Revoke and rotate a key, then attempt to keep using the old value. Confirm
   the revoked or rotated key stops working promptly everywhere, including behind any caching or replica. A
   revocation that leaves the old key live, or a rotation that does not disable the previous key, means a
   compromised credential you believed you killed still authenticates.

5. **Check for leakage and enforcement on use.** Look for keys in logs, URLs and query strings, client-side
   code, error messages, and referrer headers, and confirm any exposed key is rotated. Separately, for tokens
   with a scope or audience claim, confirm it is enforced on use so the token only works against its intended
   endpoints. A leaked-and-never-rotated key and a token whose scope is not checked on use are both live access
   past the intended bounds.

6. **Confirm and record.** Confirm with test keys by using an over-scoped key beyond its function, an unexpiring
   key past its intended life, a revoked or rotated key that still works, or a token against an endpoint outside
   its audience, without touching real credentials. Kill the lead if keys are least-privilege scoped, expiring,
   provably revocable, unexposed, and scope-enforced on use. Record the over-scoped, unexpiring, or leaked key,
   the access it grants past its bounds, and the missing scope limit, expiry, or working revocation.

## Where key lifecycle leaks

- **Over-scoped keys.** A key with broader scope than its caller needs turns any leak into access far beyond the
  one function it was issued for.
- **No expiry.** A key or token with no bounded lifetime stays valid indefinitely, so a credential stolen once
  keeps working until someone happens to notice.
- **Revocation that does not stop the key.** A revoke or rotate that leaves the old value live, or that a cache
  or replica ignores, means a compromised credential still authenticates after you believed it was killed.
- **Leakage without rotation.** A key exposed in logs, URLs, client code, or error messages and never rotated is
  a published credential anyone who saw it can use.
- **Scope or audience not enforced on use.** A token whose scope or audience claim is not checked when it is used
  works against endpoints it was never issued for.

## Worked example (a confirm and a kill)

> **Confirm.** A service issues API keys that all carry account-wide scope regardless of the integration, never
> expire, and are logged in full in the request access log. On a test account, a key issued for a
> single read-only integration is used to perform account-wide write operations, still works months later with
> no expiry, and is recovered verbatim from a log line, and rotating it leaves the previous key authenticating.
> **Confirmed** over-scoped, unexpiring, leaked, and non-revoking key lifecycle, `high`, remediation = issue
> least-privilege scoped keys, set and enforce a bounded expiry, stop logging key values, and make rotation
> immediately disable the previous key everywhere.
>
> **Kill.** Each key is issued with least-privilege scope enforced on every call, carries a bounded enforced
> expiry, is never written to logs or URLs, and rotation or revocation disables the old value promptly across all
> nodes and caches; tokens' scope and audience are checked on use. An over-scope call is refused, an expired key
> is rejected, a revoked key stops working immediately, and no key appears in any log. **Killed**, `kill_reason`
> = "keys are least-privilege scoped, expiring, provably revocable, unexposed, and scope-enforced on use; no key
> grants access past its intended bounds."

## Rationalizations to reject

- *"The key is secret."* → Bearer keys leak, into logs, URLs, and repos; scope it minimally, expire it, and be
  able to revoke it so a leak is bounded rather than total.
- *"It is a machine-to-machine key, it needs to last."* → Long-lived is not the same as unexpiring or
  unrevocable; give it a bounded lifetime and a working rotation so a compromise has an end.
- *"We rotated the key."* → Confirm the old key actually stopped working everywhere; a rotation that leaves the
  previous value live has not revoked anything.
- *"Broad scope is simpler."* → Broad scope makes every leak catastrophic; issue a narrow key per integration so
  a leak grants only that one function.
- *"The token says what it is for."* → A scope or audience claim only protects if it is enforced on use; confirm
  the endpoint rejects a token issued for a different audience or scope.

## Executing this in practice

You need each key type's scope, expiry, and revocation mechanism, where keys are stored and whether they appear
in logs or URLs, and whether token scope and audience are enforced on use. With test keys, exercise a key beyond
its scope, past its expiry, and after revocation, and search logs and client code for exposed values. Reading
the issuance and enforcement path shows the intended bounds; a key that works out of scope, past expiry, or
after revocation shows where the bounds fail.

## Related

- `auditing-service-account-key-lifecycle` - the service-account-specific case; a service account's key is a
  long-lived credential with the same scope, expiry, and rotation questions.
- `auditing-oauth-token-audience-and-scope-trust` - the audience and scope enforcement for OAuth tokens; this
  skill covers the broader key lifecycle around it.
- `hunting-non-human-identity-and-secret-reachability` - where a leaked or reachable key can be found; that skill
  hunts the exposure this one bounds.
- `reviewing-secrets-manager-access-policy-trust` - the storage and access policy for the keys; a well-scoped key
  still needs a sound policy on where it lives.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the over-scoped, unexpiring, or leaked key, sink = the
  access it grants past its intended bounds, evidence = the missing scope limit, expiry, or working revocation.
