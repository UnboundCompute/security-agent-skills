---
name: hunting-setuid-and-capability-escalation
description: >-
  Hunt local privilege escalation through setuid and setgid binaries and per-file
  capabilities: programs that run as a more privileged identity, or files granted a
  capability such as changing user id, overriding file permissions, raw disk or memory
  access, or loading kernel modules, that expose an exec, file-read, file-write, or
  library-load primitive an unprivileged caller can reach. Covers known dangerous tools
  left with the bit set, custom or bundled setuid programs that shell out or trust a
  writable path, and over-broad capabilities that are privilege in all but name. Use when
  auditing a host, image, or package for local privilege escalation. The elevated
  identity is the source, the primitive it exposes is the sink, and the missing
  confinement is the bug.
license: MIT
---

# Hunting setuid and capability escalation: elevated identity plus a primitive

A program that runs as a more privileged identity is only safe if it does exactly one
narrow thing and exposes no way to turn that privilege into an arbitrary action. The bug
is the pairing: an elevated identity (the setuid or setgid bit, or a file capability)
next to a reachable primitive (spawn a shell or command, read or write a file, load a
library) that an unprivileged user can steer. You find it by enumerating everything that
carries elevated identity on the host, and for each asking what an unprivileged caller can
make it do as that identity.

## When to use

- You are auditing a host, container image, or package for local privilege escalation.
- Binaries carry the setuid or setgid bit, or files are granted per-file capabilities.
- Some of those programs are custom, bundled by a vendor, or left over from install.

## Scope check

Audit local privilege escalation only on hosts or images you own or are authorized to
test, with an unprivileged account you may escalate from. If you can't name the
authorization, stop.

## The loop

1. **Enumerate every carrier of elevated identity.** Inventory all setuid and setgid
   binaries and all files granted a capability, across the whole filesystem including
   mounted images and package payloads. Note the owning identity each escalates to and the
   specific capability granted (changing user id, overriding file-permission checks, raw
   disk or memory access, module loading, binding low ports). This inventory is the whole
   attack surface; do not scope it to a guessed shortlist.

2. **Classify each by the primitive it exposes.** For each carrier, determine what an
   unprivileged caller can drive it to do as the elevated identity: run a shell or an
   arbitrary command, read a file it chooses, write a file it chooses, load a library, or
   only a single fixed action with no attacker-influenced input. A carrier with an exec,
   read, write, or library-load primitive is the lead; one with none is likely fine.

3. **Trace the primitive to attacker input.** Follow the exposed primitive to what the
   caller controls: an argument, an environment variable, a config or data file it reads,
   a subprocess it launches by an unqualified name resolved through the caller's path, or a
   library it loads from a writable directory. The escalation is real only where caller
   input reaches the primitive; establish that hop, do not assume it.

4. **Judge the capability as privilege, not a lesser grant.** A capability that overrides
   file-permission checks reads and writes any file; one that gives raw disk or memory
   access, or loads modules, is root by another route. Treat these as full privilege on the
   binary that holds them, and check whether the binary confines them to its one purpose or
   lets a caller redirect them at an arbitrary target.

5. **Check the custom and bundled carriers hardest.** A stock system tool with the bit set
   is a known quantity; a vendor helper or an in-house setuid program is where the unsafe
   subprocess, the trusted writable path, the format-string, or the missing argument check
   lives. Read these as privileged code handling untrusted input, because that is what they
   are.

6. **Confirm and record.** Confirm by escalating on a test host: drive the primitive to run
   a command, read a protected file, or write one as the elevated identity. Kill the lead if
   every carrier is a single-purpose program with no attacker-reachable exec, read, write, or
   library-load, and every capability is confined to that purpose. Record with the exact
   binary or capability, the primitive, and the input path that reached it.

## Where setuid and capability privilege leaks

- **Identity without confinement is the whole bug.** Elevated identity is only dangerous
  next to a primitive a caller can steer; the audit is finding that pairing, not listing
  every setuid file.
- **A file-override or raw-access capability is root wearing a smaller name.** Do not rank
  it as partial; it reads or writes anything.
- **Custom setuid code is privileged code with untrusted input.** The subprocess launched by
  an unqualified name, the config read from a writable path, the argument passed unchecked -
  that is where escalation lives.
- **The environment crosses the boundary unless something strips it.** A path or loader
  variable the elevated program honors turns its every subprocess and library load into your
  primitive.
- **Left-over bits accumulate.** A tool that needed the bit for one release and not the next,
  or a debugging helper shipped by accident, is a standing escalation nobody re-audits.

## Worked example (a confirm and a kill)

> **Confirm.** A vendor-bundled setuid helper runs a maintenance step by invoking a
> compression tool by its short name, relying on the caller's path to find it. An
> unprivileged user prepends a writable directory holding a program of that name; the helper
> executes it as the owning privileged identity. **Confirmed** setuid-to-command escalation
> via unqualified subprocess and caller-controlled path, `high`, remediation = invoke the
> subprocess by absolute path with a sanitized environment, or drop the setuid bit and gate
> the action behind an authenticated service.
>
> **Kill.** Every setuid and setgid binary is a stock single-purpose tool with no
> shell-out, no file argument the caller controls, and no honored path or loader variable;
> the two files with capabilities hold a low-port-bind grant confined to a fixed listener,
> not a file-override or raw-access capability. Escalation attempts through each primitive
> fail. **Killed**, `kill_reason` = "no carrier exposes an attacker-reachable exec, read,
> write, or library-load; capabilities confined to a single non-file-override purpose."

## Rationalizations to reject

- *"It is a standard system binary, it must be safe."* → Safe with the bit set for its
  intended use, not when a caller controls its path, environment, or a file argument. Check
  the primitive, not the reputation.
- *"It is only a capability, not full root."* → A file-override, raw-disk, raw-memory, or
  module-load capability is root by another door. Rank it as privilege.
- *"The program drops privileges."* → After it reads a file, spawns a subprocess, or loads a
  library? Drop-order is the bug. Check what runs while still elevated.
- *"No user could write to that directory."* → Verify it. A single group-writable or
  world-writable entry on the path or in the library search is the whole escalation.
- *"We inherited the setuid binary from the base image."* → Then you inherited its escalation.
  Inventory the image, not just what you added.

## Executing this in practice

You need a full filesystem inventory of setuid and setgid binaries and capability-bearing
files across the host and any base image, the ability to read each carrier's behavior (which
primitive it exposes and what input reaches it), and an unprivileged test account to confirm.
A view that lists every privileged binary and traces its exec, file, and library operations
to caller-controlled input turns this into a coverage pass over the whole carrier set; the
identity-plus-primitive question is the method by hand.

## Related

- `hunting-scheduled-job-and-search-path-hijacks` - the other local-escalation surface:
  privileged jobs and the writable paths and globbed arguments they trust.
- `hunting-dynamic-linker-hijacks` - the library-load primitive in depth, through the loader's
  search path and preload behavior.
- `auditing-guard-gaps` - the unguarded-peer pattern, here the setuid program that skips a
  check its safer sibling applies.
- `detecting-memory-safety-bugs` - a memory-corruption primitive inside a setuid binary is a
  direct escalation; pair the two.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the elevated identity, sink = the
  primitive it exposes, evidence = the escalation performed as that identity.
