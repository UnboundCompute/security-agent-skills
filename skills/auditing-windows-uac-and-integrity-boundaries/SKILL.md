---
name: auditing-windows-uac-and-integrity-boundaries
description: >-
  Audit elevation and integrity boundaries on Windows, where a medium-integrity process reaches a high-
  integrity context without a genuine consent prompt, through an auto-elevating binary that loads an
  attacker-influenced input, a writable resource an elevated process consumes, an over-broad elevation policy,
  or an integrity level that does not gate the operation it should. Use when reviewing how a host distinguishes
  elevated from non-elevated code and whether a non-administrator can cross that line without real consent.
  Covers auto-elevation abuse, elevated processes consuming writable inputs, permissive elevation policy, and
  integrity levels that fail to gate an operation. The medium-integrity influence over what an elevated
  process consumes is the source, the auto-elevating or elevated operation is the sink, and reaching a high-
  integrity context without genuine consent is the bug.
license: MIT
---

# Auditing Windows UAC and integrity boundaries: when elevation happens without a real prompt

Windows separates code that runs elevated from code that does not, marks processes with an integrity level,
and asks the user to consent before a process gains administrative rights. That boundary holds only when
crossing it requires either a genuine consent prompt or an existing high-integrity context. It breaks when an
auto-elevating binary, one the system elevates without prompting, can be steered by a medium-integrity process
into loading an input the attacker controls, when an already-elevated process consumes a file, registry value,
or path a non-administrator can write, when the elevation policy is broad enough to elevate more than it
should, or when an integrity level is set but does not actually gate the operation it is meant to protect. In
each case a medium-integrity foothold reaches a high-integrity context without the consent the boundary
promised. The bug is elevation without genuine consent. You audit these by finding every way a non-elevated
process influences what an elevated one does.

## When to use

- A host has auto-elevating binaries or an elevation policy and you can enumerate what they consume.
- An elevated process reads a file, registry value, or path that a non-administrator may write.
- An integrity level is relied on to gate an operation you can test from a medium-integrity context.

## Scope check

Audit elevation boundaries only on hosts you own or are authorized to assess, on non-production or a snapshot,
using a benign marker that proves the high-integrity context was reached rather than taking real
administrative action. A confirmed crossing yields administrative rights, so keep every probe within scope. If
you can't name the authorization, stop.

## The loop

1. **Establish whether crossing the boundary needs genuine consent or an existing high context first.** For
   each way a medium-integrity process might reach high integrity, determine whether it requires a real consent
   prompt or an already-elevated context, or whether an auto-elevating binary, a writable consumed input, or a
   permissive policy lets the crossing happen without consent. This is the false-positive killer: a boundary
   that always requires genuine consent or existing elevation is not crossed by a medium-integrity foothold.
   Name the consent-free crossing before crafting a proof.

2. **Enumerate auto-elevating binaries and elevated processes.** List the binaries the system elevates without
   prompting and the processes already running elevated, and for each note the inputs it consumes: files it
   loads, registry values it reads, paths it resolves, and objects it opens. These consumed inputs are where a
   medium-integrity process may exert influence.

3. **Check for medium-integrity influence over a consumed input.** Determine whether a non-administrator can
   write a file, registry value, or path that an auto-elevating or elevated process then consumes, so the
   elevated code acts on attacker-controlled input. A writable consumed input is the crossing.

4. **Check the elevation policy breadth.** Determine whether the policy elevates more than it should:
   auto-elevation that extends to a binary that loads external input, or a configuration that suppresses the
   prompt for operations that should require it. A broad policy turns an ordinary action into a consent-free
   elevation.

5. **Check that integrity levels actually gate.** Determine whether an operation relied on to be gated by
   integrity level is truly refused from medium integrity, or whether the object's permissions or the
   operation's implementation let a medium-integrity process perform it anyway. A level that is set but not
   enforced is no boundary.

6. **Confirm and record.** Confirm by influencing a consumed input or exercising the policy so a benign marker
   runs in the high-integrity context on an isolated host, without genuine consent. Kill the lead if every
   crossing requires a real prompt or existing elevation, if no consumed input is writable by a
   non-administrator, and if the integrity level actually refuses the operation. Record the crossing, the
   consumed input or policy, and the high-integrity marker, or set a `kill_reason`.

## Where the integrity boundary leaks

- **The consent-free crossing is the finding.** Elevation with a genuine prompt is the design; the bug is a
  medium-integrity foothold reaching high integrity without one. Name how consent is skipped.
- **Auto-elevating binaries that load input are levers.** A binary the system elevates without prompting
  becomes an escalation when a medium-integrity process steers what it loads.
- **Elevated processes consuming writable inputs.** An elevated process reading a file, registry value, or
  path a non-administrator can write acts on attacker input at high integrity.
- **Broad elevation policy skips the prompt.** A policy that elevates too much, or suppresses the prompt for
  operations that should require it, removes the consent the boundary depends on.
- **A set integrity level that does not gate.** An integrity level marked on an object but not enforced by its
  permissions or the operation lets a medium-integrity process act as though the boundary were absent.

## Worked example (a confirm and a kill)

> **Confirm.** An auto-elevating binary loads a configuration file from a path a standard user can write.
> Planting a benign marker configuration there causes the elevated binary to act on it in a high-integrity
> context at the next run, with no consent prompt, on an isolated host. **Confirmed** elevation without
> genuine consent through a writable input to an auto-elevating binary, `high`, remediation = load inputs to
> auto-elevating binaries only from administrator-writable locations, require a genuine consent prompt for the
> operation, and narrow the auto-elevation policy to binaries that consume no external input.
>
> **Kill.** The same binary loads its configuration only from an administrator-writable location, the
> elevation policy prompts for the operation, and the integrity level on the target object refuses a
> medium-integrity write. A standard user cannot influence what the elevated code consumes and cannot elevate
> without consenting. **Killed**, `kill_reason` = "the elevated binary consumes only admin-writable inputs, the
> operation requires a genuine consent prompt, and the integrity level refuses the medium-integrity write; no
> consent-free crossing exists."

## Rationalizations to reject

- *"The user has to approve elevation."* -> Only if the crossing actually prompts; an auto-elevating binary or
  a writable consumed input reaches high integrity without any prompt, which is the finding.
- *"That binary is signed and trusted."* -> Trust in the binary does not sanitize the input it loads; a
  writable configuration or resource it consumes carries the attacker's influence into the elevated context.
- *"Elevation is a convenience, not a security boundary."* -> Whether or not it is called a boundary, crossing
  it without consent reaches administrative rights, which is the impact this audit measures.
- *"The integrity level is set on that object."* -> Set is not enforced; confirm the operation is actually
  refused from medium integrity rather than only labeled.
- *"Only the installer writes there."* -> Confirm the live permissions; a path left writable by a
  non-administrator is a writable consumed input regardless of who was meant to write it.

## Executing this in practice

You need every auto-elevating binary and elevated process with the inputs it consumes, the write permissions
on each consumed file, registry value, and path, the elevation policy, and whether each integrity level
actually refuses the operation it gates. For each, decide whether a medium-integrity process can influence
what the elevated code consumes or cross the boundary without consent. Reading the consumed inputs and the
policy settles most leads; planting a benign marker input and observing it act in the high-integrity context
on an isolated host settles the rest.

## Related

- `hunting-windows-dll-hijacking-and-search-order` - a writable library that an auto-elevating binary loads is
  a consumed input in the sense this skill audits, so the two meet on the elevated load.
- `hunting-windows-service-privilege-escalation` - a service escalation reaches a privileged account by a
  related write-reaches-execution shape, a companion route across the same host.
- `auditing-windows-token-and-privilege-abuse` - reaching a high-integrity context grants a token whose
  privileges that skill then audits as the next lever.
- `detecting-race-conditions` - a writable input consumed by an elevated process is often exploitable through
  a check-to-use window that skill treats generally.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the medium-integrity influence over what an elevated
  process consumes, sink = the auto-elevating or elevated operation, evidence = a benign marker running in the
  high-integrity context without genuine consent on an isolated host.
