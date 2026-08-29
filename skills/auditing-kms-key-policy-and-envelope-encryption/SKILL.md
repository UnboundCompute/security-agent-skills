---
name: auditing-kms-key-policy-and-envelope-encryption
description: >-
  Audit key-management policies and envelope-encryption design for a decrypt path broader than intended: a
  key policy or grant that admits a principal who should never decrypt, a wildcard key resource in an
  identity policy that covers unrelated keys, an encryption context that is not enforced so a data key
  decrypts outside its intended scope, and a cross-account key grant that widens the decrypt set. Covers
  cloud key-management services, key policies and grants, and envelope encryption where a data key protects
  the payload and the key policy protects the data key. Use when data is protected by a managed key and the
  key policy plus encryption context are the boundary on who can decrypt. The principal the key policy admits
  is the source, the decrypt operation is the sink, and the decryptor beyond the data's intended readers is
  the bug.
license: MIT
---

# Auditing KMS key policy and envelope encryption: who can actually decrypt

Envelope encryption protects data with a data key and protects the data key with a managed master key, so
the real access boundary is the master key's policy plus whatever encryption context constrains the data
key. That indirection hides over-broad access. A key policy or grant can admit a principal who should never
decrypt this data; a wildcard key resource in an identity policy can cover keys the author never considered;
and an encryption context, meant to bind a data key to a specific scope, does nothing if the decrypt path
does not enforce it. Cross-account grants widen the set further. The payload can be perfectly encrypted while
the decrypt set quietly includes the wrong principals. You audit these by computing who can decrypt and
comparing that to who should read the data.

## When to use

- Data is protected by a managed key through envelope encryption, with a data key under a master key.
- Decrypt access is governed by a key policy, key grants, and identity policies that may use wildcards.
- An encryption context is used to scope a data key, and you need to confirm it is enforced on decrypt.

## Scope check

Audit key access only in accounts and key stores you own or are authorized to assess. Confirming a decrypt
recovers real plaintext, so stay inside the authorized keys and treat recovered data as sensitive within
scope, never exfiltrating it. If you can't name the authorization, stop.

## The loop

1. **Establish the data's intended readers first.** For the data a key protects, name the principals that
   legitimately need to decrypt it. This is the false-positive killer: a key policy admitting exactly those
   readers is correct however sensitive the key is. Name the intended readers, then compute the decrypt set
   against them.

2. **Read the key policy and grants for who they admit.** The master key's policy and any grants name the
   principals allowed to decrypt. Determine the true admitted set, noting broad principals (an account-wide
   grant, a wildcard) and any condition intended to scope it. Grants are easy to miss because they live apart
   from the policy document.

3. **Add identity policies with key wildcards.** A principal whose identity policy grants decrypt on a
   wildcard key resource can decrypt with keys the key policy never named, if the key policy also delegates to
   identity policies. Expand wildcard key resources against the real key inventory and add every principal they
   admit to the decrypt set.

4. **Check that the encryption context is enforced.** An encryption context binds a data key to a scope (a
   tenant, a purpose) only if every decrypt call supplies and the policy requires the matching context. If the
   context is set on encrypt but not required on decrypt, any holder of decrypt permission recovers the data
   regardless of scope. Confirm the context is a required condition, not a convention.

5. **Resolve cross-account and service grants.** A grant to another account or to a service principal widens
   the decrypt set beyond the key's home account. Confirm each cross-account decrypt grant corresponds to an
   intended reader and is scoped, not a broad delegation that admits an entire external account.

6. **Confirm and record.** Confirm by decrypting the data as a principal that should not be a reader but is
   admitted by the combined policy, or by decrypting a data key without supplying the intended encryption
   context, within scope and without exfiltrating plaintext. Kill the lead if the union of key policy, grants,
   and identity policies admits only intended readers, the encryption context is required on decrypt, and no
   cross-account grant widens the set beyond intent. Record the admitting term, the decrypt sink, and the
   decryptor beyond intended readers.

## Where KMS and envelope encryption leak

- **The decrypt set is a union, and grants hide.** Key policy, grants, and identity policies each add
  principals; a grant living apart from the policy is the common miss.
- **A wildcard key resource covers unrelated keys.** An identity policy granting decrypt on all keys reaches
  keys the author never considered.
- **An unenforced encryption context scopes nothing.** If decrypt does not require the matching context, the
  data key decrypts outside its intended scope.
- **Cross-account grants widen the set beyond the home account.** A broad delegation admits an entire external
  account to the decrypt path.
- **Encrypted is not the same as access-controlled.** Strong encryption with a loose key policy protects the
  bytes on disk but not against an admitted decryptor.

## Worked example (a confirm and a kill)

> **Confirm.** A per-tenant data key is encrypted with a tenant-scoped encryption context, but the decrypt
> path does not require the context, and an analytics role holds decrypt on the master key. The analytics
> role decrypts any tenant's data key without supplying that tenant's context, recovering cross-tenant
> plaintext. **Confirmed** cross-tenant decrypt through unenforced encryption context, `high`, remediation =
> require the tenant encryption context as a condition on every decrypt, scope the analytics role's decrypt to
> the keys it needs, and confirm the effective decrypt set matches intended readers per tenant.
>
> **Kill.** The master key policy, grants, and identity policies together admit only the per-tenant service
> roles, decrypt requires the matching tenant encryption context as a policy condition, and no cross-account
> grant widens the set. A principal outside a tenant's readers, or a decrypt without the tenant context, is
> denied. **Killed**, `kill_reason` = "combined key policy, grants, and identity policies admit only intended
> readers, the encryption context is required on decrypt, and no cross-account grant widens access; no
> unintended principal recovers plaintext."

## Rationalizations to reject

- *"The data is encrypted."* → Encryption protects the bytes; the key policy decides who decrypts. Compute the
  decrypt set separately.
- *"The key policy is tight."* → Add grants and wildcard identity policies; a grant apart from the policy or a
  broad identity policy widens the set.
- *"We set an encryption context."* → Confirm decrypt requires it; a context set on encrypt but not enforced
  on decrypt scopes nothing.
- *"It is only shared with a partner account."* → A cross-account grant to a whole account admits every
  principal in it; scope it to the intended reader.
- *"Analytics needs to read across the board."* → Broad decrypt on a master key makes analytics a reader of
  everything it protects; scope to the specific keys and require the context.

## Executing this in practice

You need the data's intended readers, the master key policy and grants, identity policies with key
wildcards, the encryption context and whether decrypt enforces it, and any cross-account grants. Compute the
effective decrypt set as the union and compare it to the readers. Reading the policies and the context
requirement shows the intended boundary; decrypting as an unintended-but-admitted principal shows whether it
holds.

## Related

- `reviewing-secrets-manager-access-policy-trust` - the secret side of the same union; a broad decrypt grant
  here is a back door there, so audit the two together.
- `hunting-non-human-identity-and-secret-reachability` - the reachability companion; once the decrypt set is
  known, that skill traces which identity actually performs the decrypt.
- `finding-crypto-misuse` - envelope-encryption design errors beyond access, such as reusing a data key across
  scopes, are that skill's territory.
- `adjudicating-taint-paths` - use it to confirm an admitted principal's credentials reach the decrypt call
  with or without the required context.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the principal the key policy admits, sink = the
  decrypt operation, evidence = the decryptor beyond the data's intended readers.
