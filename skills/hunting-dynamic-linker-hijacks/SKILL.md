---
name: hunting-dynamic-linker-hijacks
description: >-
  Hunt local privilege escalation and code execution through the dynamic loader: a preload
  environment variable honored across a privilege boundary, a writable directory on the
  runtime library search path, an embedded run-path that points at a writable or
  origin-relative location, and libraries loaded by an unqualified name. Covers preload
  variables that survive a privilege transition through a service manager or delegation
  rule, world- or group-writable library directories a privileged binary searches, run-path
  entries relative to a writable component, and dynamic loads of a short name. Use when
  auditing a privileged binary, service, or image for loader-based hijacking. The
  attacker-controlled library or variable is the source, the loader mapping it into the
  privileged process is the sink, and the unstripped or writable search path is the bug.
license: MIT
---

# Hunting dynamic-linker hijacks: the binary is fine, its search path is not

A trusted program can be perfectly written and still run your code, because it does not
choose most of the code it executes: the dynamic loader does, resolving library dependencies
at startup and on demand from a search path and a set of environment variables. If any input
to that resolution crosses a privilege boundary under attacker influence - a preload variable
the privileged process still honors, a writable directory on its search path, a run-path
relative to a location you can write - the loader maps your library into the privileged
process and runs your initializer as its identity. You find it by asking, for each privileged
binary, where the loader looks for code and which of those places an attacker can control.

## When to use

- You are auditing a privileged binary, service, container image, or package for local
  escalation.
- Programs run as a more privileged identity and load libraries dynamically.
- The launch environment, library search path, or embedded run-path may be attacker-influenced.

## Scope check

Audit loader-based escalation only on hosts, images, or binaries you own or are authorized to
test, with an unprivileged account you may escalate from. If you can't name the authorization,
stop.

## The loop

1. **Identify the privileged binaries and how they launch.** Inventory the programs that run as
   a more privileged identity - services, setuid binaries, jobs - and determine the environment
   each starts with: what a service manager passes, what a delegation rule preserves or clears,
   what a parent process hands down. The launch environment decides whether the loader's
   attacker-facing controls are live across the boundary.

2. **Check whether preload variables survive the boundary.** The loader ignores preload and
   library-path variables for a setuid binary launched directly, but a service manager, a
   delegation rule that preserves the environment, or a wrapper script can carry them across.
   Determine, per launch path, whether an attacker-set preload or library-path variable reaches
   the privileged process. Where it does, any preloaded library runs as the privileged identity.

3. **Map the runtime library search path and its writability.** Reconstruct the ordered search
   path the loader uses for each binary: the configured system library directories, any cache,
   and the binary's embedded run-path. For each entry, check whether a lower-privileged user can
   write it, or write a library that resolves ahead of the intended one. A writable directory
   anywhere the loader searches is a planted-library escalation.

4. **Inspect embedded run-path and origin-relative entries.** Read the run-path and search-path
   entries baked into the binary. Flag any that are writable, that are relative rather than
   absolute, or that are expressed relative to the binary's own location where that location - or
   a component of it - is writable by a lesser user. An origin-relative run-path into a directory
   a lesser user controls is a standing hijack.

5. **Find unqualified and attacker-named dynamic loads.** Where the program loads a library by an
   unqualified or partial name at runtime, or builds a library name from input, check whether the
   resolved path can be steered to attacker content through the search path or the name itself. A
   dynamic load of a short name resolves through the same writable-path exposure as a startup
   dependency.

6. **Confirm and record.** Confirm by escalating on a test host: set the surviving preload
   variable, plant a library in a writable search-path or origin-relative directory, or satisfy an
   unqualified load with attacker content, and observe your initializer run as the privileged
   identity. Kill the lead if preload and library-path variables are stripped across every
   boundary, every search-path and run-path entry is root-owned and non-writable and absolute, and
   no dynamic load resolves through an attacker-controllable path. Record with the binary, the
   loader input, and the escalation.

## Where the loader leaks

- **The program does not pick its code; the loader does.** A flawless binary still runs a planted
  library if the search path or environment feeds one in. Audit the resolution, not just the source.
- **The privilege boundary is where preload should die.** Direct setuid launches strip it; service
  managers, delegation rules, and wrapper scripts often do not. Check each launch path.
- **One writable directory on the path is the whole bug.** Anywhere the loader searches that a
  lesser user can write, or write an earlier-resolving name, is a planted-library escalation.
- **Origin-relative run-path is only as safe as the binary's directory.** If that directory or a
  component is writable, the run-path points at attacker content.
- **A cache or config can redirect the search.** The path the loader actually uses is the built one
  at run time, including any cache and override; reconstruct that, not the documented default.

## Worked example (a confirm and a kill)

> **Confirm.** A privileged service is started by a manager unit that preserves the invoking
> environment, and the service binary loads a plugin library by an unqualified name. An
> unprivileged user sets the loader's preload variable to a library in a directory they own; the
> manager carries the variable across, and the loader maps the attacker library into the service,
> running its initializer as root. **Confirmed** preload-across-boundary escalation, `high`,
> remediation = clear preload and library-path variables in the unit and delegation rules, and load
> the plugin by absolute path from a root-owned directory.
>
> **Kill.** Every privileged launch clears preload and library-path variables; all configured
> library directories and every binary's run-path are absolute, root-owned, and non-writable; no
> run-path is origin-relative into a writable location; the one runtime dynamic load uses an absolute
> path. Planted libraries and preload variables never load. **Killed**, `kill_reason` = "preload and
> library-path stripped across all boundaries, all search-path and run-path entries absolute,
> root-owned, non-writable, dynamic load by absolute path."

## Rationalizations to reject

- *"The loader ignores preload for setuid binaries."* → For a direct setuid launch, yes. Through a
  service manager, a delegation rule that keeps the environment, or a wrapper, it does not. Check the
  actual launch path.
- *"The library directories are standard."* → Standard and non-writable? One group-writable directory
  on the search path, or a writable cache, breaks the assumption.
- *"The run-path is relative to the binary, that is self-contained."* → Only if the binary's directory
  is not writable, at every component. Origin-relative into a writable location is a hijack.
- *"It loads its plugins by name, that is normal."* → By unqualified name resolved through a
  controllable path? That is the same exposure as a startup dependency.
- *"It is inside a container, so it is isolated."* → From the host, maybe; from a lesser user inside the
  image, no. Loader escalation is a same-host, cross-identity bug.

## Executing this in practice

You need the set of privileged binaries and their real launch environments, the reconstructed runtime
search path and embedded run-path for each, the writability of every search-path component for lesser
identities, and an unprivileged account to confirm. A view that resolves, per binary, where the loader
looks for code - configured directories, cache, run-path, origin expansion - and flags which entries a
lower-privileged user can influence turns this into a coverage pass over the loader surface; the
where-does-it-look-and-who-controls-it question is the method by hand.

## Related

- `hunting-setuid-and-capability-escalation` - the binaries that carry elevated identity, whose library
  loads this skill examines in depth.
- `hunting-scheduled-job-and-search-path-hijacks` - the command-path analog of this library-path bug,
  for scripts and scheduled jobs.
- `auditing-declared-vs-used-permissions` - the environment a service is granted versus what it needs;
  a preserved environment is an over-grant here.
- `detecting-memory-safety-bugs` - a loaded library is native code in the process; the two escalation
  surfaces compound.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the attacker library or preload variable, sink
  = the loader mapping it into the privileged process, evidence = the initializer run as that identity.
