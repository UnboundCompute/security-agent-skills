---
name: hunting-command-argument-and-flag-injection
description: >-
  Hunt argument and flag injection where untrusted input occupies a slot in a subprocess argument vector,
  with no shell involved, and the target program parses it as an option rather than data. Covers a value
  that starts with a dash and becomes a flag, and a value inserted before the program separates options
  from operands, turning a data argument into one that changes behavior: writing or reading a file, running
  an embedded command, changing a config or protocol, or reaching a network target. This is not command
  injection, there is no shell, so metacharacter defenses miss it; the fix is an argument terminator and a
  fixed positional layout. Use when input is placed into an argv slot of a spawned tool. The untrusted argv
  value is the source, the target program option parser is the sink, and an argument reinterpreted as a
  dangerous option is the bug.
license: MIT
---

# Hunting command argument and flag injection: when a value becomes an option

Argument injection lives in the safe-looking case: a program is spawned without a shell, as a fixed binary
with a separate argument vector, so there is no metacharacter interpretation and command injection does not
apply. The bug is that the target program parses its own arguments, and an attacker-controlled value placed
in the vector can be read as an option rather than as data. A value that begins with a dash becomes a flag; a
value inserted where the program has not yet seen the options-terminator can turn on a behavior the caller
never intended, writing to a file, reading one, executing an embedded command, changing an output or a
protocol, or contacting a network host. Because there is no shell, defenses aimed at metacharacters do
nothing. You find it by locating shell-less spawns whose argument vector includes untrusted values and asking
whether the target can parse those values as options.

## When to use

- Code spawns a fixed program with an argument vector (no shell) that includes untrusted input.
- The target program accepts options that read or write files, run commands, or change its destination.
- Input can start with a dash, or is inserted before a fixed set of trailing operands.

## Scope check

Test argument injection only against systems you own or are authorized to assess, on non-production hosts,
because a confirmed case makes a real tool take an unintended action. Prove the class with an inert option
whose effect is observable but harmless, and stay within the authorized host. If you can't name the
authorization, stop.

## The loop

1. **Establish that input can occupy an option slot first.** For each shell-less spawn, determine whether an
   untrusted value can start with a dash, or be positioned in the vector before the program has seen an
   options terminator, so the target parses it as an option rather than an operand. This is the
   false-positive killer: a value pinned after a `--` terminator, or in a fixed operand position the program
   treats as data, cannot become a flag. Confirm the value can reach an option-parsed slot before going on.

2. **Identify the target program and its dangerous options.** Name the spawned tool and read, generically,
   which of its options change behavior in a security-relevant way: writing output to an arbitrary path,
   reading or including a file, executing a subcommand or hook, changing the destination or protocol, or
   loading a config. The impact is defined by what options the specific tool exposes.

3. **Trace the untrusted value into the vector.** Follow the request or stored input into the exact argv
   element it becomes, and confirm the code does not sanitize a leading dash, reorder operands, or insert a
   terminator. A value that the caller believes is a filename or a search term, placed directly into the
   vector, is the vector.

4. **Check the options-terminator and positional layout.** Determine whether the spawn inserts a `--`
   terminator before untrusted operands and whether the tool honors it, and whether the layout fixes
   untrusted values in positions the tool treats as data. A missing terminator, or a tool that keeps parsing
   options after operands, leaves the slot exploitable.

5. **Assess reachable impact for this tool.** From a value that can be an option, determine what it actually
   enables for the target: an arbitrary-write flag, a file-read or include flag, a command-execution or hook
   option, or a network or protocol switch. This is where severity is decided; assess it from the tool's
   option surface, not by launching destructive flags.

6. **Confirm and record.** Confirm by supplying a value that a benign, observable option would trigger (an
   output written to an allowed location, a version banner, a bounded delay) through the input on an
   authorized host, showing the tool parsed it as an option. Kill the lead if the spawn inserts a `--`
   terminator the tool honors before untrusted operands, if the value is pinned to a data-only position, if
   a leading dash is rejected or neutralized, or if the tool exposes no behavior-changing option. Record the
   input, the spawn and target tool, the option the value became, and the reachable effect.

## Where argument injection leaks

- **There is no shell, so shell defenses miss it.** Quoting and metacharacter filtering aim at a shell that
  is not present; the bug is in the target program's own option parser.
- **A leading dash is the trigger.** A value that starts with a dash is read as a flag by most tools unless a
  terminator or a fixed operand position prevents it.
- **The missing terminator is the root cause.** Without a `--` before untrusted operands, the tool cannot
  tell the caller's data from options.
- **Impact is the tool's option surface.** A tool with a write-to-file, read-file, exec-hook, or
  change-destination option turns a data argument into that action.
- **Callers assume the value is data.** The code treats the value as a filename or a query, but the target
  treats a dash-prefixed value as configuration.

## Worked example (a confirm and a kill)

> **Confirm.** A repository feature spawns a fixed version-control binary with a user-supplied ref placed
> directly in the argument vector and no options terminator. A ref that begins with a dash is parsed by the
> tool as an option that runs an external helper, and an inert helper that writes an observable marker to an
> allowed path executes. **Confirmed** argument injection reaching command execution through a tool option,
> `high`, remediation = insert a `--` terminator before untrusted operands, reject values beginning with a
> dash, and fix untrusted values to positions the tool treats as data.
>
> **Kill.** The same spawn places the ref after a `--` terminator that the tool honors, validates the ref
> against an allowlist pattern, and rejects a leading dash. A dash-prefixed ref is delivered as a literal
> operand and the tool reads it as data. **Killed**, `kill_reason` = "spawn inserts an honored options
> terminator before the untrusted operand and rejects leading dashes; the value cannot be parsed as an
> option."

## Rationalizations to reject

- *"There is no shell, so it is safe."* -> The absence of a shell rules out command injection, not argument
  injection; the target program parses its own options from the vector.
- *"We escape shell metacharacters."* -> Metacharacter escaping is irrelevant with no shell; a plain
  dash-prefixed value is the whole problem.
- *"It is only a filename."* -> A filename that starts with a dash is an option to most tools; the caller's
  intent does not bind the target's parser.
- *"The tool is harmless."* -> Assess the tool's actual option surface; many common tools have a write, read,
  include, hook, or destination option that changes behavior.
- *"We put it at the end of the arguments."* -> Position alone does not help unless a terminator precedes it
  and the tool stops parsing options after operands.

## Executing this in practice

You need every shell-less spawn whose argument vector includes untrusted input, the target program and its
behavior-changing options, whether a `--` terminator is inserted and honored, and whether leading dashes are
rejected. For each, decide whether the value can reach an option-parsed slot and what option it could become.
Reading the spawn shows the vector layout and terminator; supplying a value that a benign observable option
would trigger shows whether the target parsed it as an option.

## Related

- `hunting-os-command-injection` - the sibling for shell-mode spawns, where the bug is metacharacter
  interpretation rather than an option parser; this skill covers the shell-less case.
- `hunting-scheduled-job-and-search-path-hijacks` - a related spawn-behavior class where the environment and
  path, rather than an argument, change what runs.
- `hunting-path-traversal-and-file-access` - a frequent impact when the injected option is a read or write
  flag pointed at an arbitrary path.
- `adjudicating-taint-paths` - use it to prove an untrusted value reaches an option-parsed argv slot.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the untrusted argv value, sink = the target
  program option parser, evidence = a benign observable option triggered by the injected value.
