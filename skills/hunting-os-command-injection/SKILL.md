---
name: hunting-os-command-injection
description: >-
  Hunt OS command injection where untrusted input reaches a process-spawning API through a shell that
  interprets metacharacters. Covers a command string passed to a system or shell-exec call, a spawn that
  requests a shell, and indirect shells reached through wildcards, subshells, environment values, or an
  attacker-influenced PATH or IFS. The distinguishing fact is the shell: a direct exec with a fixed program
  and an argument vector does not interpret metacharacters and is not this bug, while any shell-composed
  string is. Allowlisting the program name does not help when arguments are interpolated into a shell.
  Use when input flows into a call that runs a command or spawns a process. The untrusted input is the
  source, the shell-interpreted command is the sink, and arbitrary command execution is the bug.
license: MIT
---

# Hunting OS command injection: when input crosses into a shell

Command injection happens when untrusted input reaches a process-spawning API in a way that lets a shell
interpret it. A shell treats characters like the semicolon, the pipe, the ampersand, backticks, and dollar
parentheses as syntax, so an input that carries them can chain, substitute, or replace the intended command.
The single fact that decides whether the class exists is whether a shell is in the loop: a call that composes
a string and hands it to a system shell is injectable, while a direct exec that passes a fixed program and a
separate argument vector, with no shell, is not, because the arguments are delivered as data. You find command
injection by locating every spawn, deciding whether it goes through a shell, and tracing whether untrusted
input reaches the command portion of a shell-interpreted call.

## When to use

- Code runs an external command built from, or including, request or stored input.
- A spawn API is called in shell mode, or with a single composed string rather than an argument vector.
- Input influences the environment, the working directory, or the PATH of a spawned process.

## Scope check

Test command injection only against systems you own or are authorized to assess, on non-production hosts,
because a confirmed case runs commands on the server. Prove the class with an inert, side-effect-free command
(an echo or a fixed delay) rather than anything destructive, and stay within the authorized host. If you
can't name the authorization, stop.

## The loop

1. **Establish whether a shell is in the loop first.** For each spawn, determine whether it invokes a shell
   (a system-style call, a shell-mode flag, or a single composed command string) or is a direct exec with a
   fixed program and a separate argument vector. This is the false-positive killer: a shell-less exec does
   not interpret metacharacters, so untrusted arguments there are not command injection, even though they
   may be argument injection, a separate concern. Name the spawn mode before judging the input.

2. **Trace untrusted input to the command portion.** Follow request parameters, headers, filenames, and
   stored fields into the spawn, and identify whether the input lands in the command itself or only in an
   argument. Input concatenated into a shell command string is the vector; confirm it is attacker-controlled
   and not fully fixed by the code.

3. **Check for indirect shells and interpretation.** A spawn can reach a shell indirectly: a program that
   itself invokes a shell, a wildcard the shell expands, a value substituted from the environment, or a
   nested command. Determine whether any layer between the input and execution interprets the string, not
   just the outermost call.

4. **Examine the environment, PATH, and IFS.** Even a shell-less exec can be steered if the attacker controls
   the environment or the working directory: a relative program name resolved through an attacker-influenced
   PATH, or field splitting driven by IFS, can change what runs. Check whether the spawn pins an absolute
   program path and a controlled environment.

5. **Read the neutralization that would hold.** Determine whether the code avoids the shell entirely by using
   an argument-vector exec with a fixed program, whether any user value in a shell context is quoted and
   escaped by a real shell-quoting routine, and whether an allowlist constrains the command to a fixed set.
   Note that allowlisting the program while interpolating arguments into a shell does not close the class.

6. **Confirm and record.** Confirm by injecting an inert marker (an echoed token or a bounded delay) through
   the input and observing the shell interpret it on an authorized host. Kill the lead if the spawn is a
   shell-less exec with a fixed program and the input is only a data argument, if a real shell-quoting or
   allowlist neutralizes every user value in a shell context, or if the input never reaches the command.
   Record the input, the spawn call and its mode, the interpreting layer, and the neutralization state.

## Where command injection leaks

- **The shell is the whole bug.** A composed string handed to a system shell interprets metacharacters; the
  same input as a data argument to a shell-less exec does not.
- **Indirect shells hide the sink.** A program that internally calls a shell, or a wildcard the shell
  expands, reintroduces interpretation even when the top-level call looks argument-based.
- **PATH and the environment steer shell-less spawns.** A relative program name plus an attacker-influenced
  PATH runs an attacker binary without any metacharacter.
- **Allowlisting the binary is not enough.** If the arguments are interpolated into a shell string, a fixed
  program name does not stop chaining or substitution in the arguments.
- **Quoting must be real and complete.** A hand-rolled quote that misses one metacharacter or one context is
  an opening; only a shell-safe quoting routine, or avoiding the shell, holds.

## Worked example (a confirm and a kill)

> **Confirm.** An export feature builds a command string that includes a user-supplied filename and passes it
> to a system shell call. A filename containing a command separator and an inert echoed token is interpreted
> by the shell, and the token appears in the output, proving execution. **Confirmed** OS command injection,
> `critical`, remediation = replace the shell call with an argument-vector exec of a fixed program, pass the
> filename as a separate argument, and validate it against an allowlist of permitted names.
>
> **Kill.** The same feature spawns a fixed absolute program with an argument vector and no shell, passes the
> filename as one element of that vector, pins the environment and PATH, and validates the name. A filename
> with separators is delivered verbatim as a single argument and never interpreted. **Killed**, `kill_reason`
> = "spawn is a shell-less exec of a fixed absolute program with the input as a data argument, environment
> and PATH pinned; no shell interprets the value."

## Rationalizations to reject

- *"We allowlist the command."* -> Allowlisting the program does not stop injection in the arguments if they
  are interpolated into a shell string; the separators still chain.
- *"We use the safe spawn function."* -> Many spawn functions run a shell when given a single string or a
  shell flag; the mode, not the function name, decides.
- *"We escape quotes."* -> Escaping one character or one context is not shell-safe quoting; a missed
  metacharacter or a nested shell reopens it. Prefer no shell.
- *"The program does not call a shell."* -> Some programs invoke a shell internally, and wildcards or the
  environment can reintroduce one; verify the whole chain.
- *"The input is just a filename."* -> A filename is attacker-controlled input; in a shell context it carries
  separators and substitutions like any other string.

## Executing this in practice

You need every spawn in the code, its mode (shell versus argument-vector exec), whether untrusted input
reaches the command portion, and the environment and PATH the spawn runs with. For each shell-mode spawn,
decide whether input reaches the command and whether quoting or an allowlist neutralizes it; for shell-less
spawns, check the program path and environment. Reading the spawn call shows the mode; an inert echoed marker
or a bounded delay through the input shows whether a shell interprets it.

## Related

- `hunting-command-argument-and-flag-injection` - the distinct sibling for shell-less spawns, where an
  untrusted value becomes an option or flag rather than a chained command.
- `hunting-scheduled-job-and-search-path-hijacks` - the environment and PATH angle in depth, where a
  writable path element changes what a spawned program resolves to.
- `hunting-dynamic-linker-hijacks` - a related environment-driven hijack where loader variables change what a
  binary executes.
- `adjudicating-taint-paths` - use it to prove input reaches the command portion of a shell-interpreted call.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the untrusted input, sink = the shell-interpreted
  command, evidence = an inert marker interpreted by the shell rather than passed as data.
