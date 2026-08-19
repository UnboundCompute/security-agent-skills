---
name: hunting-scheduled-job-and-search-path-hijacks
description: >-
  Hunt local privilege escalation through scheduled jobs and the paths privileged
  processes trust: periodic and timer jobs whose script, or a file or directory they read,
  is writable by a lower-privileged user; commands invoked by an unqualified name resolved
  through a writable search-path entry; and argument injection where a command expands a
  shell wildcard over a directory an attacker can write to, so a file named like an option
  (a leading-dash filename) becomes a command-line flag. Covers writable job scripts,
  writable directories on an effective path, relative command execution, and the
  filename-as-flag wildcard trick. Use when auditing a host or image for local escalation
  through automation. The writable input is the source, execution as the job's identity is
  the sink.
license: MIT
---

# Hunting scheduled-job and search-path hijacks: what runs, as whom, over what you can write

Automation escalates privilege quietly. A job runs on a timer as a privileged identity, and
if anything it touches is writable by a lesser user, or if it resolves a command through a
path or a wildcard that a lesser user can influence, that user's content runs as the job's
identity. The bug is never the schedule; it is the trust the job places in a writable script,
a writable directory on its search path, an unqualified command name, or a filename that a
wildcard hands to a command as an option. You find it by listing every job, the identity it
runs as, and every input it trusts, then checking which of those inputs a lesser user controls.

## When to use

- You are auditing a host or image for local privilege escalation through automation.
- Periodic jobs, timers, or service-triggered scripts run as a privileged identity.
- Those jobs execute scripts, resolve commands by name, or expand wildcards over directories.

## Scope check

Audit scheduled-job and path escalation only on hosts or images you own or are authorized to
test, with an unprivileged account you may escalate from. If you can't name the authorization,
stop.

## The loop

1. **Enumerate every job and the identity it runs as.** Inventory periodic jobs, timers, and
   service-triggered scripts across every location they can be defined, system-wide and
   per-user, and record which identity each runs as. A job running as an unprivileged service
   account is a smaller prize than one running as root, but map them all; the identity sets
   the stakes.

2. **List every input each job trusts.** For each job, follow what it executes and reads: the
   job script itself, any script or config or data file it sources or parses, the directories
   it operates over, and the commands it launches. Each of these is an input the job trusts to
   be benign. The audit is which of those inputs a lesser user can change.

3. **Check writability of scripts, configs, and directories.** For every trusted input, check
   whether a lower-privileged user can write it: a job script that is group- or world-writable,
   a config sourced from a writable path, a directory the job writes into or globs over that a
   lesser user owns. A writable trusted input is a direct hijack; the lesser user plants
   content that runs as the job's identity.

4. **Check command resolution against a writable path.** Where the job invokes a command by an
   unqualified name, determine the effective search path at run time and whether any entry is
   writable by, or ahead of the real binary for, a lesser user. A privileged job that runs a
   command by short name with a writable directory early on its path executes the planted
   binary as its identity.

5. **Check wildcard and argument injection.** Where the job runs a command that expands a shell
   wildcard over a directory a lesser user can write to, check whether a crafted filename is
   handed to the command as an option rather than data. A file whose name begins like a flag
   turns an innocent operate-on-all-files command into one carrying attacker-chosen options,
   several of which reach command execution or file overwrite as the job's identity. The
   writable directory plus the unquoted wildcard is the bug.

6. **Confirm and record.** Confirm by escalating on a test host: plant a writable-script or
   writable-path payload, or a flag-shaped filename in a globbed directory, and observe it run
   as the job's identity on the next trigger. Kill the lead if every job runs a root-owned,
   non-writable script, resolves commands by absolute path with a controlled environment, and
   globs only over directories no lesser user can write. Record with the job, the writable input
   or wildcard, and the execution it produced.

## Where scheduled jobs and paths leak

- **The schedule is never the bug; the trusted writable input is.** A job is a hijack wherever
  a lesser user can change a script, config, directory, path entry, or globbed filename it trusts.
- **Unqualified command names inherit the caller's path.** A short name plus one writable early
  path entry is arbitrary code as the job's identity.
- **An unquoted wildcard hands filenames to the command as options.** A leading-dash filename in a
  globbed writable directory is argument injection, and several common commands turn that into
  execution or overwrite.
- **Writability is per component, not just the file.** A non-writable script in a writable
  directory can be replaced; a config sourced from a writable path is as good as writable.
- **Per-user and system jobs are one surface.** An escalation through a privileged user's job is
  still an escalation; enumerate every definition location, not just the system ones.

## Worked example (a confirm and a kill)

> **Confirm.** A privileged periodic job archives everything in a spool directory by expanding a
> wildcard over it, and the spool directory is writable by an unprivileged service user. That user
> drops two files: one named to look like the archiver's option that runs a command, and a script
> for it to run. On the next trigger the archiver treats the filename as an option and executes the
> script as root. **Confirmed** wildcard argument injection to root, `high`, remediation = pass
> `--` before the wildcard or an explicit file list, never glob over a writable directory, and run
> the job over a root-owned staging path.
>
> **Kill.** Every job is a root-owned, non-writable script; each invokes commands by absolute path
> with a fixed, minimal environment; the only directories they write or glob are root-owned and not
> writable by any lesser user; wildcards are guarded with an end-of-options marker. Planted scripts,
> path entries, and flag-shaped filenames never execute. **Killed**, `kill_reason` = "root-owned
> non-writable scripts, absolute-path commands with controlled environment, no lesser-writable
> globbed or sourced input, wildcards option-guarded."

## Rationalizations to reject

- *"It only runs as a service account, not root."* → Then it escalates to the service account,
  which often owns the data you care about and is a step toward root. Rank it, do not dismiss it.
- *"The job script is owned by root."* → Owned, but in a writable directory, or sourcing a config
  from a writable path? Ownership of the file is not ownership of everything it trusts.
- *"It just archives or copies files."* → Over a writable directory with a wildcard? That is where
  a flag-shaped filename becomes code execution. The mundane job is the classic vector.
- *"The path is set correctly in the job."* → At definition, or at run time after the environment is
  built? Verify the effective path and every entry's writability.
- *"Nobody can write there."* → Check it, per component, for every lesser identity. One
  group-writable directory is the whole escalation.

## Executing this in practice

You need every job definition and its run-as identity across all system and per-user locations, the
set of scripts, configs, directories, and commands each job trusts, the writability of each of those
for every lesser identity, and the effective run-time path. An unprivileged account confirms. A view
that maps each scheduled job to the files and commands it touches and flags which are writable by a
lower-privileged user turns this into a coverage pass over the automation surface; the what-runs,
as-whom, over-what-you-can-write question is the method by hand.

## Related

- `hunting-setuid-and-capability-escalation` - the other local-escalation surface: binaries that
  carry elevated identity directly.
- `hunting-dynamic-linker-hijacks` - when the writable input a job trusts is a library the loader
  finds, not a script or command.
- `detecting-race-conditions` - a job that checks then acts on a path a lesser user can swap is a
  TOCTOU on top of the trust gap.
- `finding-fail-open-flaws` - a job that proceeds when an input is missing or malformed is the
  fail-open shape in automation.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the writable script, path entry, or globbed
  filename, sink = execution as the job's identity, evidence = the payload that ran.
