---
name: auditing-terraform-state-and-backend-trust
description: >-
  Audit infrastructure-state storage and its backend for exposure and tampering: a state file holding
  plaintext secrets in a backend readable by too many principals, a state bucket or backend without
  encryption, versioning, or access scoping, a missing or unenforced state lock that allows concurrent
  corrupting writes, and a backend configuration that lets an attacker redirect state to a location they
  control. Covers Terraform and similar tools where the state file records real resource attributes and
  often secret values, and where whoever can read or write state can read those secrets or subvert the next
  apply. Use when infrastructure state is stored in a shared backend and that backend is the boundary. The
  principal who can read or write state is the source, the state store is the sink, and the exposed secret or
  the tamperable state is the bug.
license: MIT
---

# Auditing Terraform state and backend trust: when the state file is the crown jewels

Infrastructure state is not a build artifact; it is a live record of every resource and, very often, the
plaintext of the secrets those resources were created with, database passwords, generated keys, tokens. It
also drives the next apply: whatever the state says exists is what the tool reconciles against. That makes
the state backend a high-value target on two axes. Anyone who can read the state file can read the secrets in
it, so a backend readable by too many principals is a secret-disclosure bug. Anyone who can write or redirect
the state can corrupt the next apply or point it at resources they control, so a missing lock or a mutable
backend configuration is a tampering bug. You audit these by checking who can read the state, how it is
protected at rest, and whether writes are locked and the backend is fixed.

## When to use

- Infrastructure state is stored in a shared backend such as an object store or a remote state service.
- The state file may contain plaintext secrets from resources it manages.
- State access, encryption, versioning, locking, or backend configuration may be loose or unenforced.

## Scope check

Audit state and backends only for infrastructure you own or are authorized to assess, on non-production
state. Reading a state file exposes real secrets, so stay inside the authorized backend and treat any secret
in state as sensitive within scope, never exfiltrating it. If you can't name the authorization, stop.

## The loop

1. **Establish who legitimately needs state access first.** Name the principals that must read and write the
   state for this infrastructure: the specific pipeline identity, the specific operators. This is the
   false-positive killer: a backend scoped to exactly those principals with encryption and locking is correct.
   Name the legitimate set, then compare actual access to it.

2. **Determine what secrets the state actually holds.** Read whether the managed resources write secret values
   into state (generated passwords, keys, connection strings). If they do, every principal who can read the
   state reads those secrets, so the read-access set is a secret-disclosure boundary, not just an operational
   one. Confirm the sensitivity of what state contains.

3. **Scope the read access to the backend.** Compute who can read the state store: the bucket or backend
   policy, identity policies, and any broad grant. Compare that set to the legitimate readers. A state bucket
   readable by a broad role, an entire account, or a wildcard is a direct path to the secrets in state.

4. **Check protection at rest and history.** Confirm the backend encrypts state at rest, retains versions so a
   corrupting write can be recovered, and does not expose old versions to a wider set than current state. A
   backend without encryption or versioning leaves the secrets and the recovery path unprotected.

5. **Check locking and backend integrity.** Confirm state locking is configured and enforced so two applies
   cannot write concurrently and corrupt the state, and that the backend configuration cannot be redirected by
   an attacker to a state location they control. A missing lock allows a corrupting race; a mutable backend
   pointer lets an attacker feed a poisoned state into the next apply.

6. **Confirm and record.** Confirm by reading a secret from the state file as a principal that should not have
   state access but is admitted, or by showing state writes are unlocked or the backend is redirectable,
   within scope and without exfiltrating secrets or corrupting real state. Kill the lead if state read access
   is scoped to legitimate principals, state is encrypted and versioned, locking is enforced, and the backend
   configuration is fixed. Record the admitted principal, the state store sink, and the exposed secret or the
   tamperable state.

## Where state and backend trust leak

- **State holds plaintext secrets.** Generated passwords and keys land in state, so read access to the backend
  is read access to those secrets.
- **A broadly readable backend is secret disclosure.** A state bucket open to a wide role or account exposes
  everything in state to that set.
- **No encryption or versioning removes protection and recovery.** Unencrypted state exposes secrets at rest;
  no versioning means a corrupting write cannot be rolled back.
- **A missing lock allows corrupting races.** Concurrent applies without a lock can leave state inconsistent
  with reality, breaking the next reconcile.
- **A redirectable backend poisons the next apply.** If an attacker can change where state is read from, they
  feed a state that drives the tool to act on resources they control.

## Worked example (a confirm and a kill)

> **Confirm.** State is stored in an object bucket readable by a broad operations role, and the managed
> database resource writes its generated password into state in plaintext. The operations role, never intended
> to hold database credentials, reads the password from the state file. **Confirmed** secret disclosure through
> over-broad state access, `high`, remediation = scope the state bucket to the pipeline identity and named
> operators only, enable encryption at rest and versioning, and move generated secrets out of state into a
> managed secrets store where feasible.
>
> **Kill.** The state backend is readable and writable only by the pipeline identity and two named operators,
> is encrypted at rest with versioning enabled, enforces state locking on every apply, and its backend
> configuration is fixed and not attacker-redirectable. A principal outside the set cannot read state and
> concurrent writes are blocked. **Killed**, `kill_reason` = "state access scoped to legitimate principals with
> encryption, versioning, and enforced locking, and a fixed backend; no unintended reader sees secrets and no
> unlocked or redirected write corrupts state."

## Rationalizations to reject

- *"State is just infrastructure metadata."* → State commonly holds plaintext generated secrets; read access to
  the backend is read access to those secrets.
- *"The bucket is internal."* → Internal is not scoped; compute who can actually read the backend and compare
  to the legitimate reader set.
- *"We do not need versioning."* → Without it, a corrupting write is unrecoverable; versioning is part of state
  integrity, not just convenience.
- *"Applies never overlap."* → Confirm locking is enforced; an unlocked backend allows a corrupting race the
  moment two runs coincide.
- *"The backend config is in the repo."* → Confirm it cannot be redirected by an attacker; a mutable backend
  pointer poisons the next apply regardless of where it lives.

## Executing this in practice

You need the legitimate state readers and writers, what secrets the state holds, the read and write access to
the backend, the encryption/versioning/locking posture, and whether the backend configuration is fixed. For
each backend, compare the access set to the legitimate one and confirm protection and locking. Reading the
backend policy shows the intended boundary; reading a secret from state as an unintended principal, or showing
an unlocked write, shows whether it holds.

## Related

- `auditing-infrastructure-as-code-exposures` - the resource-declaration side; state is the runtime record of
  what those declarations produced, secrets included.
- `reviewing-secrets-manager-access-policy-trust` - the better home for the secrets that leak through state;
  moving them there is the durable remediation.
- `auditing-iac-module-and-provider-supply-chain` - the code side of the same pipeline; state trust and module
  trust are the two halves of IaC security.
- `hunting-non-human-identity-and-secret-reachability` - the pipeline identity that reads and writes state is a
  machine credential; that skill traces what it can reach.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the principal who can read or write state, sink = the
  state store, evidence = the exposed secret or the tamperable state.
