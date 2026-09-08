---
name: hunting-windows-dll-hijacking-and-search-order
description: >-
  Hunt code execution through Windows library search order, where a privileged process loads a library by
  name and resolves it from a location a lower-privileged user can write, because the library is absent from
  its expected directory, the application directory or a path entry is writable, or the load uses an
  unsafe search that reaches the current or a user-controlled directory. Use when reviewing how a privileged
  executable or service resolves its dynamic libraries and whether any resolution step lands in a writable
  location. Covers phantom missing libraries, writable application directories, unsafe search modes, and
  side-loading through a copied trusted binary. The lower-privileged write into a resolved search location
  is the source, the privileged process loading the library by name is the sink, and executing
  attacker-supplied library code in that process is the bug.
license: MIT
---

# Hunting Windows DLL hijacking and search order: when a load-by-name lands in a writable directory

A Windows process that loads a library by name, rather than by a fixed full path, asks the loader to find it,
and the loader walks an ordered list of directories. That is safe only when every directory the loader
reaches before the real library is writable by no one below the process's own privilege. When a privileged
process requests a library that is missing from its own directory, or when its application directory or a
path entry is writable by a standard user, or when it uses a search mode that includes the current directory,
the loader can resolve the name to a file the attacker planted. The privileged process then maps and runs
that code as itself. The bug is not the load; it is a resolved search location that a lower-privileged user
can write, upstream of where the genuine library lives. You find these by tracing each privileged load to the
directories the loader searches and asking which of them a non-admin can write.

## When to use

- A privileged executable or service loads libraries by name and you can observe its load behavior.
- The application directory, a path entry, or a working directory in the search order may be user-writable.
- A trusted signed binary can be copied to a writable directory alongside a planted dependency.

## Scope check

Test library search order only on hosts you own or are authorized to assess, on non-production or a snapshot,
using a benign marker library that proves it was loaded rather than one that takes real action. A confirmed
hijack runs code inside a privileged process, so keep every probe within the authorized scope. If you can't
name the authorization, stop.

## The loop

1. **Establish where the loader actually resolves each library first.** For each privileged load, determine
   the ordered directories the loader searches and whether the genuine library sits in a protected directory
   ahead of any writable one, or whether a missing library, a writable application directory, a writable
   path entry, or a current-directory search lets resolution reach a location a standard user controls. This
   is the false-positive killer: a load that always resolves from a directory writable only by administrators,
   ahead of every user-writable one, cannot be hijacked. Name the resolved location before crafting a plant.

2. **Enumerate privileged loads and their libraries.** List the executables and services that run with
   elevated privilege and the libraries each loads by name, separating those loaded from a fixed full path
   (not in scope) from those left to the search order. The name-resolved loads are the candidates.

3. **Find the phantom and shadowed cases.** Determine which requested libraries are absent from the
   process's own directory, so the loader continues down the order into a writable location, and which are
   present but preceded in the order by a writable directory that a same-named file would shadow.

4. **Check the writable search locations.** For each candidate, determine whether the application directory,
   any directory on the path used, or the working directory the process runs with is writable by a standard
   user, and whether the search mode includes an unsafe location such as the current directory.

5. **Consider side-loading a trusted binary.** A signed, trusted executable copied into a writable directory
   still loads its dependencies by the search order from that directory, so planting a dependency beside a
   copied trusted binary runs attacker code under the trust of the copied executable; determine whether that
   route is reachable.

6. **Confirm and record.** Confirm by planting a benign marker library at the resolved writable location and
   observing the privileged process load it on an isolated host. Kill the lead if every directory the loader
   reaches before the genuine library is writable only by administrators, if the load uses a fixed full path
   or a safe search mode that excludes writable locations, or if the process is no more privileged than the
   attacker. Record the process, the library, the resolved writable location, and the load observed, or set a
   `kill_reason`.

## Where search-order hijacking leaks

- **The resolved location is the finding.** A load-by-name is only a hijack when the loader reaches a
  writable directory before the genuine library; trace the order, not just the name.
- **Missing libraries fall through.** A requested library absent from the protected directory sends the
  loader onward into whatever comes next, which may be writable.
- **Writable application directories shadow.** An application installed in a directory a standard user can
  write lets a same-named library there win over the intended one.
- **Unsafe search modes reach the current directory.** A load that includes the working or current directory
  in its order can resolve to a file the attacker placed where the process happens to run.
- **Side-loading borrows trust.** A trusted signed binary copied into a writable folder still resolves its
  dependencies from there, so a planted dependency runs under the copied binary's reputation.

## Worked example (a confirm and a kill)

> **Confirm.** A service running as a privileged account loads a helper library by name, and that library is
> absent from the service's own protected directory while an earlier directory on the search order is
> writable by a standard user. A benign marker library planted there is loaded into the privileged process at
> the next start on an isolated host. **Confirmed** DLL search-order hijack to code execution in a privileged
> process, `high`, remediation = load the library from a fixed full path in a protected directory, ship the
> genuine library in the process's own admin-only directory, and use a safe search mode that excludes
> user-writable and current directories.
>
> **Kill.** The same service loads the helper by a fixed full path from an admin-only directory where the
> genuine library is present, uses a safe search mode, and runs with a working directory writable only by
> administrators. No directory the loader reaches before the real library is user-writable. **Killed**,
> `kill_reason` = "the load resolves by full path from an admin-only directory with a safe search mode; no
> writable location precedes the genuine library, so no plant is loaded."

## Rationalizations to reject

- *"The library loads fine, so the path is correct."* -> A working load says nothing about whether a writable
  directory precedes the genuine library; trace the order to a plant, not to success.
- *"The application is signed."* -> Signing the executable does not sign what it loads by name; a planted
  unsigned library in a resolved writable directory still runs unless the loader enforces it.
- *"That library is always present."* -> Present in one directory does not mean present first; a writable
  earlier directory shadows a present library.
- *"Users cannot write to program directories."* -> Some applications install under user-writable paths, and
  path and working directories vary; confirm the actual permissions rather than assuming the default.
- *"We copy a trusted tool at runtime."* -> Copying it into a writable directory lets a planted dependency
  load beside it under its trust; the copy destination must be protected.

## Executing this in practice

You need every privileged load-by-name with the ordered search directories the loader uses, which requested
libraries are missing or shadowed, and the write permissions on each directory in the order including the
working and current directory. For each, decide whether the loader reaches a user-writable location before
the genuine library. Reading the resolution order and directory permissions settles most leads; planting a
benign marker library and observing the privileged process load it on an isolated host settles the rest.

## Related

- `hunting-windows-service-privilege-escalation` - a service whose binary or path is redirectable is often
  the same process whose library loads are hijackable, so the two findings frequently share a target.
- `hunting-dynamic-linker-hijacks` - the Unix counterpart, where the runtime linker's search and preload
  variables play the role the Windows search order plays here.
- `auditing-windows-token-and-privilege-abuse` - code loaded into a privileged process inherits that
  process's token and privileges, which that skill audits as the next lever.
- `auditing-container-image-build-hardening` - shipping libraries and binaries in protected, fixed locations
  is the build-time control that removes the writable-directory precondition.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the lower-privileged write into a resolved search
  location, sink = the privileged process loading the library by name, evidence = the privileged process
  loading a planted marker library on an isolated host.
