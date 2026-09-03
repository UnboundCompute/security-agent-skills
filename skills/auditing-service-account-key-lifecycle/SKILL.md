---
name: auditing-service-account-key-lifecycle
description: >-
  Audit cloud and platform service-account keys for lifecycle weaknesses that turn a non-human credential into
  standing access: a user-managed key file that never expires and is copied into repos, CI config, or developer
  laptops, a service account granted far more privilege than its workload needs so the key is a broad credential,
  a key that is never rotated and has no owner tracking who holds copies, a key that can be created or downloaded
  by someone who should only use the account, and a disabled or deleted service account whose outstanding keys
  still authenticate. Use when a non-human identity authenticates with a long-lived key and the scope, rotation,
  and containment of that key is the boundary. The over-privileged, unrotated, or exportable service-account key
  is the source, the workload-level access an attacker gains by holding it is the sink, and the excess privilege,
  missing rotation, or exportable key material is the bug.
license: MIT
---

# Auditing service-account key lifecycle: a downloaded key is a password for a robot with real power

A service account is a non-human identity a workload uses to call cloud and platform APIs, and its key is a
long-lived credential with no user, no second factor, and often broad platform privilege behind it. That makes a
downloaded service-account key one of the most dangerous credentials in a system: it is a bearer secret that
grants a workload's full authority to anyone who copies the file, and workloads have real power. The lifecycle
failures are specific. A user-managed key file that never expires gets copied into repositories, CI
configuration, and developer laptops and lingers there for years. A service account granted more privilege than
its workload needs makes the key a broad credential rather than a narrow one, so a leak grants the excess. A key
that is never rotated, with no owner tracking who holds copies, cannot be reasoned about or safely retired. A
service account whose key can be created or downloaded by someone who should only be allowed to use the account
lets a lesser principal export standing credentials. And a service account that was disabled or deleted but
whose outstanding keys still authenticate leaves live access after the account was supposedly gone. The audit
prefers keyless workload identity, and where keys exist, checks they are minimally privileged, rotated, owned,
non-exportable by lesser principals, and truly dead when the account is. You audit this by finding every key,
what it can do, who can export it, and whether it can be killed.

## When to use

- A cloud or platform service account authenticates a workload with a long-lived key or credential file.
- A service account may be over-privileged, its key may never expire or rotate, or the key may be exportable by
  a principal who should only use the account.
- Keys may be copied into repos, CI, or laptops, or outstanding keys may survive the account being disabled.

## Scope check

Test service-account key lifecycle only against accounts and projects you own or are authorized to assess, with
test service accounts. Creating, downloading, using, and deleting keys changes real access and identity state,
so use test accounts and never export, use, or delete a service-account key that is not yours. If you can't name
the authorization, stop.

## The loop

1. **Establish each service account's intended privilege and key posture first.** For every service account,
   name the minimal privilege its workload needs, whether it should use a key at all or a keyless workload
   identity, how any key is meant to rotate, and who may create or download it. This is the false-positive
   killer: a workload using keyless identity, or a service account with least-privilege roles whose key is
   rotated, owned, non-exportable by lesser principals, and truly revoked on disable, is behaving correctly. Name
   the intended posture, then test each account.

2. **Prefer keyless and inventory the keys that exist.** Determine which workloads could use a keyless workload
   identity but instead use a downloaded key. Inventory every user-managed key: where it lives, when it was
   created, and whether it has ever been rotated. A downloaded long-lived key where keyless identity was possible
   is avoidable standing exposure.

3. **Test privilege against workload need.** Compare each service account's granted roles against what its
   workload actually calls. Exercise the key against APIs and resources outside the workload's function and
   confirm they are refused. A service account with project-wide or broadly privileged roles for a narrow
   workload makes its key a broad credential, so confirm least privilege.

4. **Test rotation, ownership, and exposure.** Check whether each key has a bounded lifetime and a rotation
   process, and an owner who tracks who holds copies. Search repositories, CI configuration, and images for
   copied key files. A key that never rotates, has no owner, or sits in a repo is a credential nobody can safely
   retire or account for.

5. **Test who can export a key and whether disable revokes.** Check which principals can create or download a key
   for each service account, and confirm a principal who should only use the account cannot export its
   credentials. Then disable and delete a service account and confirm its outstanding keys stop authenticating. A
   lesser principal who can export a key gains standing credentials, and a key that outlives the account's
   deletion is live access to a ghost.

6. **Confirm and record.** Confirm with test accounts by using an over-privileged key beyond its workload,
   exporting a key as a principal who should not, finding an unrotated key in a repo, or authenticating with a
   key after the account was disabled, without touching real accounts. Kill the lead if workloads use keyless
   identity or least-privilege rotated owned keys, only authorized principals can export them, and disabling the
   account revokes them. Record the over-privileged, unrotated, or exportable key, the workload-level access it
   grants, and the excess privilege, missing rotation, or exportable key material.

## Where service-account key lifecycle leaks

- **Downloaded keys where keyless was possible.** A user-managed key file for a workload that could use keyless
  identity is avoidable long-lived exposure that ends up in repos and laptops.
- **Over-privileged service accounts.** Roles broader than the workload needs make the key a broad credential, so
  a leak grants the excess privilege, not just the one function.
- **No rotation or ownership.** A key that never rotates and has no owner tracking its copies cannot be reasoned
  about or safely retired.
- **Exportable by a lesser principal.** A principal who should only use the account but can create or download
  its key gains standing exportable credentials.
- **Keys that survive account disable.** Outstanding keys that still authenticate after the service account is
  disabled or deleted leave live access to an account that should be gone.

## Worked example (a confirm and a kill)

> **Confirm.** A workload authenticates with a downloaded user-managed service-account key; the account holds a
> broad project-editor role, the key has never rotated since creation, and a copy sits in a CI configuration
> file in a repository. On a test project, the key from the repo performs project-wide writes far outside the
> workload's function and continues to work long after creation with no rotation, and a developer with only
> use-the-account permission is able to download a fresh key. **Confirmed** over-privileged, unrotated,
> exportable, and exposed service-account key, `high`, remediation = move the workload to a keyless workload
> identity, scope the account to least privilege, rotate and remove the exposed key, and restrict key creation to
> authorized principals only.
>
> **Kill.** The workload uses a keyless workload identity, or where a key exists the service account holds only
> least-privilege roles, the key rotates on a bounded schedule with a named owner, no copy appears in any repo or
> CI config, only authorized admins can create or download a key, and disabling the account immediately stops its
> keys from authenticating. An out-of-scope call is refused, a lesser principal cannot export a key, and a
> disabled account's key is dead. **Killed**, `kill_reason` = "workloads use keyless or least-privilege rotated
> owned keys, only authorized principals can export them, and disable revokes them; no key grants workload access
> past its intended bounds."

## Rationalizations to reject

- *"The workload needs a key."* → Many workloads can use a keyless workload identity instead; prefer it, and where
  a key is unavoidable, scope, rotate, and contain it.
- *"It is only a service account."* → A service account often has broad platform privilege and no second factor;
  its key is a high-value bearer credential, not a low-stakes one.
- *"The key is in our private repo."* → A key in any repo, private or not, is copied, cloned, and cached beyond
  your control; keep key material out of repos and rotate any that lands there.
- *"We grant editor to keep it simple."* → Broad roles make the key a broad credential; scope the account to the
  exact APIs the workload calls so a leak is bounded.
- *"We deleted the service account."* → Confirm its outstanding keys actually stopped authenticating; a key that
  outlives the account is live access to an identity you think is gone.

## Executing this in practice

You need the inventory of service accounts and their keys, each account's granted roles versus its workload's
calls, each key's rotation and ownership, which principals can export a key, and whether disabling an account
revokes its keys. With test accounts, exercise a key beyond its scope, export one as a lesser principal, search
repos and CI for copies, and authenticate after disabling the account. Reading the account's roles and key
settings shows the intended posture; a key that works out of scope, exports to the wrong principal, or survives
disable shows where it fails.

## Related

- `auditing-api-key-and-token-lifecycle` - the general key-lifecycle skill; a service-account key is a specific
  long-lived credential with the same scope, expiry, and revocation questions.
- `hunting-non-human-identity-and-secret-reachability` - where a service-account key can be reached and reused;
  that skill hunts the exposure this one bounds.
- `mapping-service-account-impersonation-chains` - once a key is held, impersonation chains extend its reach;
  that skill maps where the account's authority leads.
- `hunting-iam-privilege-escalation-paths` - an over-privileged service account is an escalation edge; that skill
  traces the path its excess privilege opens.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the over-privileged, unrotated, or exportable
  service-account key, sink = the workload-level access it grants, evidence = the excess privilege, missing
  rotation, or exportable key material.
