---
name: auditing-windows-token-and-privilege-abuse
description: >-
  Audit privilege escalation through Windows access token and privilege abuse, where a process holds a
  sensitive privilege or can obtain a more privileged token, letting a lower-privileged foothold impersonate
  a privileged caller, load a driver, take ownership, back up protected files, or debug another process to
  reach a higher context. Use when a service or process runs with a named sensitive privilege enabled or can
  receive and impersonate tokens from privileged callers. Covers impersonation privileges, token theft and
  duplication, and the backup, restore, ownership, load-driver, and debug privileges. The privilege or the
  capturable privileged token is the source, the operation the privilege authorizes is the sink, and reaching
  a higher-privileged context from a lower one is the bug.
license: MIT
---

# Auditing Windows token and privilege abuse: when a held privilege is a ladder

A Windows access token carries the privileges a process may exercise, and a handful of them are equivalent to
full control of the host even when the account is not an administrator. A process that can impersonate a
client can capture a privileged caller's token and act as that caller. One that holds the privilege to load a
driver runs code in the kernel. One that can back up or restore files reads and writes past every access
control. One that can take ownership or debug any process reaches into a more privileged context directly.
The bug is not that the privilege exists; it is that a lower-privileged foothold holds a privilege, or can
obtain a token, whose authorized operation lifts it to a higher context. You audit these by listing the
privileges each process holds and each token it can capture, and asking which of them crosses a trust
boundary the account was not meant to cross.

## When to use

- A service or process runs with a named sensitive privilege enabled that you can enumerate.
- A process accepts connections from privileged callers and can impersonate their tokens.
- A foothold account holds backup, restore, ownership, load-driver, or debug rights on a host you assess.

## Scope check

Audit token and privilege abuse only on hosts you own or are authorized to assess, on non-production or a
snapshot, using a benign proof that the privileged operation succeeded rather than performing a damaging one.
A confirmed abuse reaches a full-control context, so keep every probe within the authorized scope. If you
can't name the authorization, stop.

## The loop

1. **Establish which privileges the foothold actually holds and can enable first.** For the process or
   account under assessment, determine the exact set of privileges present in its token and which are enabled
   or can be enabled, and which privileged tokens it is positioned to capture. This is the false-positive
   killer: a privilege that is present but not held by the foothold, or an impersonation that only ever
   receives tokens no more privileged than the account itself, is not an escalation. Name the specific
   privilege or capturable token before claiming a path.

2. **Enumerate the sensitive privileges in scope.** List the privileges that are equivalent to escalation
   when held by a non-administrator: the impersonation privileges, the load-driver privilege, the backup and
   restore privileges, the take-ownership privilege, and the debug privilege. Note which of these the
   foothold's token carries and whether each is enabled or enable-able.

3. **Map impersonation to a captured token.** For an impersonation privilege, determine whether the process
   can induce a privileged caller to connect or authenticate to it and then impersonate that caller's token,
   turning a service foothold into the privileged caller's context. Identify the concrete way a privileged
   token is made to arrive.

4. **Map each direct privilege to its authorized operation.** Load-driver runs code in the kernel; backup and
   restore read and write files past their access control; take-ownership seizes an object and then rewrites
   its permissions; debug opens a handle into a more privileged process. For each held privilege, name the
   operation it authorizes and the higher context that operation reaches.

5. **Check the boundary the operation crosses.** An escalation exists only where the authorized operation
   reaches a context more privileged than the foothold: reading a protected credential store, writing a
   system binary, injecting into a system process, or loading kernel code. Confirm the operation actually
   crosses that boundary rather than staying within the account's own resources.

6. **Confirm and record.** Confirm by exercising the privilege for a benign proof on an isolated host: capture
   a token and read something the foothold could not, or use backup to open a protected file for read only, or
   load a benign signed test driver, observing the higher context without damage. Kill the lead if the
   foothold does not hold or cannot enable the privilege, if the only capturable tokens are no more privileged
   than the account, or if the operation stays within the account's own resources. Record the privilege, the
   operation, the boundary crossed, and the benign proof, or set a `kill_reason`.

## Where token and privilege abuse leaks

- **The held privilege is the finding, not its name.** A sensitive privilege matters only when the
  lower-privileged foothold actually holds and can enable it; confirm possession before impact.
- **Impersonation captures a caller's context.** A process that can impersonate turns any privileged caller
  it can induce to connect into that caller's token, so the reach is whoever can be made to authenticate.
- **Backup and restore ignore access control.** These privileges read and write files past their permissions,
  so a foothold holding them reads protected stores and writes protected binaries.
- **Load-driver reaches the kernel.** The privilege to load a driver runs attacker-chosen code in the most
  privileged context on the host, so it is escalation by definition when a non-admin holds it.
- **Take-ownership and debug reach across processes.** Ownership seizes an object to rewrite its permissions;
  debug opens a handle into a more privileged process to read or inject, each crossing the boundary directly.

## Worked example (a confirm and a kill)

> **Confirm.** A service account foothold holds an impersonation privilege, and the service can be induced to
> have a privileged caller authenticate to it. Capturing and impersonating that caller's token lets the
> foothold read a resource restricted to the privileged account, demonstrated for read only on an isolated
> host. **Confirmed** privilege escalation through token impersonation, `high`, remediation = remove the
> impersonation privilege from the account if it is not required, run the service under an account with only
> the privileges it needs, and prevent privileged callers from authenticating to a lower-privileged service.
>
> **Kill.** The same account's token does not carry the impersonation privilege, holds no backup, restore,
> ownership, load-driver, or debug privilege, and the only callers it can impersonate run as the same
> unprivileged account. No held privilege authorizes an operation that crosses to a higher context. **Killed**,
> `kill_reason` = "the foothold holds no sensitive privilege it can enable and can capture no token more
> privileged than itself; no authorized operation reaches a higher context."

## Rationalizations to reject

- *"The account is not an administrator."* -> Several privileges are administrator-equivalent on their own; a
  non-admin holding backup, load-driver, or an impersonation privilege escalates without being an admin.
- *"The privilege is present but disabled."* -> A privilege the foothold can enable is held for this purpose;
  check whether it can be enabled, not only whether it is enabled right now.
- *"Only system services impersonate."* -> If the assessed service can impersonate and a privileged caller can
  be induced to connect, the foothold captures that caller's token regardless of what usually connects.
- *"Backup is just for backups."* -> The backup privilege reads any file past its access control, so it reads
  protected credential and configuration stores the account could not otherwise open.
- *"Debugging needs a debugger attached."* -> The debug privilege opens a handle into a more privileged
  process directly; no interactive debugger is required for the escalation.

## Executing this in practice

You need the exact privilege set in the foothold's token, which privileges are enabled or enable-able, and,
for impersonation, which privileged callers can be induced to authenticate to the process. For each sensitive
privilege held, name the authorized operation and the higher context it reaches, then confirm the boundary is
actually crossed. Reading the token's privileges settles most leads; a benign proof, a read past access
control or a captured privileged token used read-only on an isolated host, settles the rest.

## Related

- `hunting-windows-service-privilege-escalation` - code running as a service account inherits that account's
  privileges, so a service escalation and the privileges the account then wields are consecutive steps.
- `hunting-windows-dll-hijacking-and-search-order` - code loaded into a privileged process gains that
  process's token, which this skill then audits for the privileges it carries.
- `auditing-windows-uac-and-integrity-boundaries` - integrity levels and elevation gate which privileges a
  token may exercise, the boundary that skill audits alongside this one.
- `hunting-windows-credential-material-exposure` - the backup and debug privileges read protected credential
  stores, feeding the material that skill pursues from the access side.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the held privilege or capturable privileged token,
  sink = the operation the privilege authorizes, evidence = the higher context reached through a benign proof
  on an isolated host.
