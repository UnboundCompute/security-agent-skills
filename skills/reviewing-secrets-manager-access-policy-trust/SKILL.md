---
name: reviewing-secrets-manager-access-policy-trust
description: >-
  Review who can actually read a managed secret: a secrets-manager or vault access policy that grants read
  to a broader principal set than the secret's consumers, a resource policy and an identity policy that
  combine to admit an unintended reader, a wildcard on the secret name or path that sweeps in unrelated
  secrets, and a decryption grant on the underlying key that widens access beyond the store's own policy.
  Covers cloud secrets managers and vault-style stores where the effective read set is the union of resource
  policy, identity policy, and key-decrypt permission. Use when applications fetch secrets from a managed
  store and the access policy is the boundary on who reads them. The principal the combined policy admits is
  the source, the secret read is the sink, and the reader beyond the secret's intended consumers is the bug.
license: MIT
---

# Reviewing secrets-manager access-policy trust: who can actually read the secret

A managed secrets store is only as strong as the policy that decides who reads each secret, and that
decision is rarely written in one place. The effective read set is the union of the secret's resource
policy, the identity policies of every principal in the account, and the permission to decrypt the key that
protects the secret. A secret can look locked down on its resource policy while a broad identity policy
elsewhere grants read on every secret, or while a wide key-decrypt grant lets a principal decrypt what the
store would otherwise deny. Wildcards on secret names or paths quietly sweep in unrelated secrets. You review
these by computing who the combined policy actually admits and comparing that to the secret's intended
consumers.

## When to use

- Applications fetch secrets from a managed secrets manager or a vault-style store.
- Access is governed by a combination of resource policy, identity policy, and key-decrypt permission.
- Secret names or paths are matched by wildcard in any policy that grants read.

## Scope check

Review secret access only in accounts and stores you own or are authorized to assess. Confirming a read
retrieves a real secret value, so stay inside the authorized store and treat any retrieved value as
sensitive within scope, never exfiltrating it. If you can't name the authorization, stop.

## The loop

1. **Establish the secret's intended consumers first.** For each secret, name the principals that legitimately
   need to read it: the specific service role, the specific workload. This is the false-positive killer: a
   policy admitting exactly those consumers is correct however powerful the store is. Name the consumers, then
   compute the actual read set against them.

2. **Read the resource policy for who it admits.** The secret's own policy grants read to named principals or
   patterns. Determine the true set it admits, noting any broad principal (an account, a wildcard) and any
   condition that is meant to scope it. This is one term of the union, not the whole answer.

3. **Add the identity policies.** A principal can read the secret if its own identity policy grants read on
   it, independent of the resource policy. Look for identity policies that grant secret-read broadly, on all
   secrets or a wide path, since these admit readers the resource policy never named. The effective read set
   includes every principal granted by either side.

4. **Add the key-decrypt permission.** The secret is protected by a key; reading the plaintext requires
   decrypting with that key. A principal with a broad decrypt grant on the key can obtain the plaintext even
   where the secret policy is tight, so the key policy is a third term of the union. Confirm the decrypt grant
   is no broader than the secret's read set.

5. **Resolve wildcards on names and paths.** A wildcard on the secret name or path in any granting policy
   matches every secret under it, so a grant meant for one secret can cover a family of unrelated ones. Expand
   the wildcards against the real secret inventory and confirm each match is an intended consumer.

6. **Confirm and record.** Confirm by reading the secret as a principal that should not be a consumer but is
   admitted by the combined policy, within scope, without exfiltrating the value. Kill the lead if the union
   of resource policy, identity policies, and key-decrypt permission admits only the intended consumers, and no
   wildcard sweeps in unrelated secrets. Record the admitting policy term, the secret-read sink, and the reader
   beyond the intended consumers.

## Where secrets-manager access trust leaks

- **The read set is a union, not the resource policy alone.** A tight secret policy is undone by a broad
  identity policy or a wide key-decrypt grant elsewhere.
- **A broad identity policy grants read on every secret.** A principal with secret-read on all resources reads
  secrets no resource policy ever named.
- **Key-decrypt is a back door to plaintext.** Decrypting the protecting key yields the value even where the
  secret policy denies; the decrypt grant must match the read set.
- **A wildcard path sweeps in unrelated secrets.** A grant scoped to a prefix covers every secret under it,
  intended or not.
- **Conditions must actually constrain.** A condition on a spoofable or caller-controlled value does not narrow
  the admitted set.

## Worked example (a confirm and a kill)

> **Confirm.** A database secret's resource policy names only the application role, but a broad identity policy
> on an operations role grants secret-read on all secrets, and that role also holds decrypt on the protecting
> key. The operations role, never an intended consumer, reads the database secret. **Confirmed** over-broad
> secret access through an identity policy, `high`, remediation = scope the operations identity policy to the
> specific secrets it needs, restrict key-decrypt to the secret's read set, and confirm the effective union
> admits only intended consumers.
>
> **Kill.** Each secret's effective read set, computed as the union of its resource policy, all identity
> policies, and key-decrypt permission, admits only the specific consuming role, and no wildcard path grants
> read beyond the intended secret. A principal outside the consumer set is denied by every term. **Killed**,
> `kill_reason` = "the combined resource, identity, and key-decrypt policies admit only the intended consumers
> and no wildcard sweeps in other secrets; no unintended principal can read the plaintext."

## Rationalizations to reject

- *"The secret policy only allows the app role."* → The read set is the union with identity policies and
  key-decrypt; compute all three before concluding it is tight.
- *"Operations needs broad access."* → Broad secret-read on an operations role makes it a reader of every
  secret; scope it to the specific secrets required.
- *"The key is separate."* → Decrypting the protecting key yields the plaintext; a broad decrypt grant is a
  read grant in effect.
- *"The wildcard is just for that service's secrets."* → Expand it against the real inventory; a prefix
  wildcard often matches secrets it was never meant to cover.
- *"There is a condition on it."* → Confirm the condition constrains an unspoofable value; otherwise it does
  not narrow the admitted set.

## Executing this in practice

You need each secret's intended consumers, its resource policy, the identity policies that grant read on it,
the key-decrypt permissions on its protecting key, and the wildcard matches across the secret inventory.
Compute the effective read set as the union and compare it to the consumers. Reading the policies shows the
intended boundary; reading the secret as an unintended-but-admitted principal shows whether it holds.

## Related

- `auditing-kms-key-policy-and-envelope-encryption` - the key side of the union; a broad decrypt grant on the
  protecting key is a back door this review must include.
- `hunting-non-human-identity-and-secret-reachability` - the reachability companion; once the read set is
  known, that skill traces which reachable identity actually pulls the secret.
- `hunting-iam-privilege-escalation-paths` - a principal that can read a secret granting further access is an
  escalation edge worth chaining from here.
- `adjudicating-taint-paths` - use it to confirm an admitted principal's credentials actually reach the
  secret-read call.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the principal the combined policy admits, sink = the
  secret read, evidence = the reader beyond the secret's intended consumers.
