---
name: hunting-windows-credential-material-exposure
description: >-
  Hunt exposure of Windows credential material, where secrets that authenticate a user or machine, cached
  logon verifiers, stored service and task passwords, protected-store secrets, saved connection credentials,
  and credentials left in memory of a privileged process, are readable by a principal that should not reach
  them because of a weak permission, an over-privileged token, or storage that does not bind the secret to
  the intended account. Use when reviewing where Windows keeps authentication secrets and which principals can
  read each store. Covers cached verifiers, service and scheduled-task secrets, protected-store material,
  saved credentials, and process memory. The readable credential store or process is the source, the read that
  recovers usable secret material is the sink, and obtaining a credential the reader was not entitled to is
  the bug.
license: MIT
---

# Hunting Windows credential material exposure: when a secret is readable by the wrong principal

Windows keeps a lot of authentication material on a host: verifiers that let a user log on when the domain is
unreachable, passwords for services and scheduled tasks that must start unattended, secrets in a protected
store, credentials saved for connections, and live credentials in the memory of processes that authenticated.
Each store is meant to be readable only by a specific principal, the account that owns it, the system, or an
administrator. The exposure is a store readable by a principal that should not reach it: a weak permission on
a file or registry location, an over-privileged token that opens a protected process or store, or storage
that protects the secret with a key the wrong account can also derive. The bug is not that the secret exists;
it is that a principal below the intended one can read usable material from it. You hunt these by listing
where credentials live and checking, for each, which principals can read the material and whether what they
read is usable.

## When to use

- A Windows host stores authentication secrets and you can enumerate the stores and their permissions.
- A principal may hold a token or privilege that opens a protected credential store or process.
- Service, task, saved-connection, or protected-store secrets may be readable below the intended account.

## Scope check

Hunt credential exposure only on hosts you own or are authorized to assess, on non-production or a snapshot,
proving a secret is recoverable and usable without authenticating as the victim account beyond the
authorization. A recovered credential authenticates as another principal, so keep every probe within scope and
handle any recovered material as sensitive. If you can't name the authorization, stop.

## The loop

1. **Establish which principal can actually read usable material from the store first.** For each credential
   store, determine the principal it is meant to be readable by and whether a lower-privileged principal can
   read it and recover usable secret material, through a weak permission, an over-privileged token, or a
   protection key the reader can also derive. This is the false-positive killer: a store readable only by the
   intended account, or one whose contents are bound so the reader cannot derive a usable secret, is not an
   exposure. Name the reading principal and the usable material before claiming impact.

2. **Enumerate the credential stores on the host.** List the places secrets live: cached logon verifiers,
   service and scheduled-task secrets, the protected store, saved connection credentials, and the memory of
   privileged processes that hold live credentials. Note the intended reader of each.

3. **Check the permission and token on each store.** Determine whether a lower-privileged principal holds a
   file or registry permission, or a token or privilege, that opens the store or the process. A weak
   permission on a file store or a privilege that opens a protected process both expose the material to a
   principal below the intended one.

4. **Check whether what is read is usable.** A verifier that only allows an offline guessing attack differs
   from a directly reusable secret; a stored password recovered in the clear is immediately usable, while a
   value bound to a key the reader cannot derive is not. Determine what the reader actually obtains and whether
   it authenticates as another principal.

5. **Check the protection binding.** Determine whether the store binds the secret to the intended account, so
   another principal reading the bytes cannot unprotect them, or whether the protecting key is derivable by a
   lower-privileged reader. A store whose protection the reader can defeat exposes the secret even when the
   file permission looks adequate.

6. **Confirm and record.** Confirm by recovering usable material as the lower-privileged principal on an
   isolated host and showing it authenticates or unlocks the intended resource, without acting as the victim
   beyond the proof. Kill the lead if the store is readable only by the intended account, if the recovered
   material is not usable, or if the protection binds the secret to an account the reader cannot assume.
   Record the store, the reading principal, the usable material, and the proof, or set a `kill_reason`.

## Where credential material leaks

- **The reading principal is the finding.** A credential store is expected to hold secrets; the exposure is a
  principal below the intended one that can read usable material. Name who can read and what they recover.
- **Weak permissions on secret files and keys.** A file or registry store of service, task, or saved
  credentials readable by a standard user hands over whatever it protects.
- **Over-privileged tokens open protected stores.** A privilege that opens a protected process or store lets a
  principal read live or at-rest credentials the permission alone would deny.
- **Protection that the reader can derive is no protection.** A secret protected by a key a lower-privileged
  account can also derive is exposed even behind a correct file permission.
- **Process memory holds live credentials.** A privileged process that authenticated may hold reusable secret
  material in memory, readable by a principal that can open the process.

## Worked example (a confirm and a kill)

> **Confirm.** A scheduled task stores the password of a privileged account in a location readable by a
> standard user, and the value is recovered in a form that authenticates as that account. The recovered
> secret unlocks the intended resource on an isolated host without further action as the victim. **Confirmed**
> credential exposure recovering a usable privileged secret, `high`, remediation = store service and task
> secrets in a protected store bound to the intended account, restrict the store's permissions to that account
> and administrators, and prefer managed accounts that do not expose a reusable password.
>
> **Kill.** The same task uses a managed account whose secret is not stored in a reusable form, the protected
> store binds each secret to its owning account, and no file, registry location, or process holding credentials
> is readable by a standard user. A lower-privileged principal recovers no usable material. **Killed**,
> `kill_reason` = "every credential store is readable only by the intended account and binds its secret to
> that account; the lower-privileged principal recovers nothing usable."

## Rationalizations to reject

- *"The secret is encrypted at rest."* -> Encryption protects only if the key is not derivable by the reader;
  confirm the protection binds the secret to an account the lower-privileged principal cannot assume.
- *"Only an administrator can read that."* -> Confirm the actual permission and the token required; a weak
  file permission or an over-privileged token can open a store meant for administrators only.
- *"A cached verifier is not a password."* -> A verifier may still enable an offline attack or reuse; judge it
  by what the reader can do with it, not by whether it is the plaintext password.
- *"The password is only in memory."* -> A principal that can open the privileged process reads live
  credentials from memory; in-memory is not out of reach if the process is openable.
- *"That account is not privileged."* -> Trace what the recovered credential authenticates to; a low-tier
  account's secret can still open resources or chain toward a privileged one.

## Executing this in practice

You need every credential store on the host, cached verifiers, service and task secrets, the protected store,
saved credentials, and privileged process memory, with the intended reader, the permission or token required,
and the protection binding of each. For each, decide whether a lower-privileged principal can read usable
material and what it authenticates to. Reading the permissions and protection settles most leads; recovering
usable material as the lower-privileged principal and showing it unlocks the intended resource on an isolated
host settles the rest.

## Related

- `auditing-windows-token-and-privilege-abuse` - the backup and debug privileges open protected stores and
  processes, the token side of the reads this skill pursues from the storage side.
- `auditing-group-policy-and-sysvol-trust` - a credential stored in a policy file on the domain share is the
  same exposure over a network share, which that skill covers.
- `hunting-non-human-identity-and-secret-reachability` - the general reachable-secret hunt across sources, of
  which the Windows credential stores here are the platform-specific case.
- `hunting-ntlm-coercion-and-relay` - a recovered machine or account credential feeds authentication as that
  identity, which that skill escalates through relay.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the readable credential store or process, sink = the
  read that recovers usable secret material, evidence = the recovered secret authenticating or unlocking the
  intended resource on an isolated host.
