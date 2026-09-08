---
name: hunting-windows-service-privilege-escalation
description: >-
  Hunt local privilege escalation through Windows service misconfiguration, where a low-privileged user can
  influence what a service runs as a higher-privileged account, through a weak service object permission, a
  writable service binary or its directory, an unquoted service path with a writable intervening directory,
  or a weak permission on the service's registry configuration. Use when reviewing installed services and
  their binaries, paths, and permissions on a Windows host for a standard-user-to-SYSTEM path. Covers weak
  service DACLs, writable binaries and paths, unquoted paths, and writable service registry keys. The
  standard-user control over service configuration or its on-disk target is the source, the service manager
  starting the binary as a privileged account is the sink, and running attacker-chosen code as that account
  is the bug.
license: MIT
---

# Hunting Windows service privilege escalation: when a standard user decides what SYSTEM runs

A Windows service usually runs as a high-privileged account, often the local system account, and the service
manager starts whatever binary the service configuration names. That is safe only if a low-privileged user
cannot influence which binary runs or where it is found. When they can, the service becomes an escalation:
the user changes the configured binary because the service object grants them that right, or replaces the
binary because its file or directory is writable, or plants a file earlier on an unquoted path because an
intervening directory is writable, or rewrites the service's registry configuration because its key is
weakly permissioned. At the next start, the privileged account runs the attacker's code. The bug is not the
service running as system; it is a standard user holding a write that reaches what system executes. You find
these by listing services and asking, for each, whether a non-admin can change the binary, the path, or the
config.

## When to use

- A Windows host runs services as a privileged account and you can enumerate their config and permissions.
- Service binaries, their directories, or unquoted service paths may be writable by a standard user.
- Service objects or their registry keys may grant configuration rights to non-administrators.

## Scope check

Test service escalation only on hosts you own or are authorized to assess, on non-production or a snapshot,
using a benign marker payload that proves execution as the privileged account rather than taking real action
with it. A confirmed path yields code execution as a system account, so keep every probe within the
authorized scope. If you can't name the authorization, stop.

## The loop

1. **Establish the service's account and who can influence its target first.** For each service, determine
   the account it runs as and whether a standard user holds any write that reaches what it executes: a
   service-object right to reconfigure it, write access to the binary or its directory, a writable directory
   earlier on an unquoted path, or write access to the service's registry key. This is the false-positive
   killer: a service whose binary, path, and configuration are writable only by administrators cannot be
   redirected by a standard user, however privileged the account it runs as. Name the specific write before
   claiming a path.

2. **Enumerate services and their configured targets.** List installed services, the account each runs as,
   the full command line and binary path, and the start type. Services that run as a privileged account and
   start automatically or can be started by a standard user are the candidates; note the exact binary and
   path for each.

3. **Check the service object permissions.** Determine whether the service's access control grants a
   standard user or a broad group the right to change its configuration or start it. A right to reconfigure
   the binary path is a direct escalation; a right only to query is not. Read the granted rights, not the
   default assumption.

4. **Check the on-disk and path writes.** Determine whether the binary itself, the directory holding it, or,
   for an unquoted path containing spaces, any earlier directory on the resolution order is writable by a
   standard user. A writable binary is replaced; a writable directory lets a same-named file shadow the
   target; an unquoted path lets a file plant earlier in the sequence win.

5. **Check the registry configuration write.** Determine whether the service's registry key, which holds the
   image path and the account, is writable by a standard user. A write there rewrites what the service
   manager launches and under which account, independent of the file permissions.

6. **Confirm and record.** Confirm by placing a benign marker binary through the specific write and observing
   the privileged account execute it at the next service start on an isolated host. Kill the lead if the
   binary, its directory, the whole unquoted path, the service object, and the registry key are all
   writable only by administrators, or if the service runs as an unprivileged account no higher than the
   attacker. Record the service, the account, the specific write, and the execution observed, or set a
   `kill_reason`.

## Where service escalation leaks

- **The write is the finding, not the account.** A service running as system is normal; a standard user
  holding a write that reaches its binary, path, or config is the escalation. Name the write.
- **Service object DACLs drift.** A service whose access control was loosened to let an application or an
  installer manage it can grant a standard user the right to repoint the binary.
- **Writable binaries and directories replace the target.** A binary or its containing directory writable by
  a non-admin lets the attacker overwrite or shadow what the privileged account runs.
- **Unquoted paths with spaces resolve early.** A path with spaces and no quotes lets the service manager try
  earlier truncations first, so a planted file in a writable earlier directory runs instead.
- **The registry key is a second door.** The service's configuration key holding the image path and account
  is an escalation if it is writable, even when the file permissions are correct.

## Worked example (a confirm and a kill)

> **Confirm.** An installed service runs as the local system account with an unquoted image path under a
> directory a standard user can write to, and spaces in the path let an earlier truncation resolve first. A
> benign marker binary planted at the earlier name is executed by the system account at the next start on an
> isolated host. **Confirmed** local privilege escalation from standard user to system through an unquoted
> service path, `high`, remediation = quote the service image path, place service binaries only in
> directories writable exclusively by administrators, and restrict the service object and its registry key to
> administrators.
>
> **Kill.** The same service uses a fully quoted image path, its binary and directory are writable only by
> administrators, the service object grants standard users query rights only, and its registry key is
> admin-writable. A standard user can neither repoint the service nor plant a file the manager would run.
> **Killed**, `kill_reason` = "binary, path, service object, and registry key are all admin-only writable; no
> standard-user write reaches what the privileged account executes."

## Rationalizations to reject

- *"The service runs as system, so of course it is powerful."* -> Power is expected; the finding is a
  standard-user write that redirects it. Without such a write there is no escalation.
- *"The binary is not writable."* -> Check the directory, the unquoted path, the service object, and the
  registry key too; any one of them redirects execution even when the file is protected.
- *"The path has spaces but it usually works."* -> Usual resolution is not the point; an unquoted path with
  a writable earlier directory lets a planted file win at the next start.
- *"Only an installer touches that service."* -> If the installer loosened the service DACL to a broad group,
  a standard user in that group inherits the right to reconfigure it.
- *"You need admin to change a service."* -> Only if the service object and registry key enforce that; a
  drifted DACL or key permission can grant reconfiguration to a standard user.

## Executing this in practice

You need every service with its account, image path and command line, start type, service object
permissions, and registry key permissions, plus the write permissions on each binary and its directory. For
each privileged service, decide whether a standard user holds any write that reaches the executed target,
through the object, the file, the path, or the key. Reading the permissions settles most leads; planting a
benign marker through the specific write and observing execution as the privileged account on an isolated
host settles the rest.

## Related

- `hunting-scheduled-job-and-search-path-hijacks` - the scheduled-task and search-path equivalent of this
  write-reaches-execution reasoning, sharing the planted-file and path-resolution analysis.
- `hunting-windows-dll-hijacking-and-search-order` - a service that loads a missing or writable library is
  redirected by the search-order gap that skill covers, a common companion to a service path finding.
- `auditing-windows-token-and-privilege-abuse` - once code runs as a service account, the privileges that
  account holds are the next lever, which that skill audits.
- `hunting-setuid-and-capability-escalation` - the Unix counterpart, where a privileged binary rather than a
  service is the carrier, sharing the escalation-from-a-write shape.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the standard-user control over the service config or
  its on-disk target, sink = the service manager starting the binary as a privileged account, evidence = the
  privileged account executing a planted marker at service start on an isolated host.
