---
name: auditing-datastore-exposure-and-abuse
description: >-
  Audit in-memory and cache datastores such as Redis and memcached for exposure and command abuse: an
  instance reachable without authentication, a request that composes datastore commands from untrusted
  input, or server-side scripting and module or config commands that reach code execution or a file write.
  Covers unauthenticated network exposure, command injection where input becomes a command rather than a
  value, Lua or scripting evaluation on untrusted input, and dangerous administrative commands that rewrite
  the on-disk file or load a module. Use when an application talks to a cache or key-value store and either
  the instance is network-reachable or untrusted input reaches the command layer. The untrusted input or the
  open port is the source, the datastore command interface is the sink, and the unauthenticated access or
  the composed dangerous command is the bug.
license: MIT
---

# Auditing datastore exposure and abuse: when the cache is an unauthenticated shell

Cache and key-value stores such as Redis and memcached are built for speed inside a trusted network, and
that assumption is exactly what makes them dangerous when it does not hold. Two failures dominate. First,
exposure: an instance bound to a reachable interface with no authentication is an open command interface,
and on some stores the command set reaches configuration changes, on-disk file rewrites, module loading, and
scripting, so an open port is a path to code execution, not just data theft. Second, command abuse: when an
application composes datastore commands from untrusted input, the attacker can inject extra commands or reach
a scripting or administrative command the application never intended. You audit both by checking how the
instance is reached and how commands are composed.

## When to use

- An application talks to a cache or key-value store such as Redis or memcached.
- A datastore instance may be bound to a reachable interface or lack authentication.
- Untrusted input reaches the command layer, a scripting evaluation, or a key or command name.

## Scope check

Test datastore exposure and abuse only against instances and applications you own or are authorized to
assess, on non-production data. A confirming command can change configuration, write files, or run a script
on the host, so treat every proof as a live intrusion inside the authorized scope. If you can't name the
authorization, stop.

## The loop

1. **Establish the reachability and the command sinks first.** Determine how each datastore instance is
   reached (bind address, network exposure, authentication requirement) and inventory where the application
   composes commands, including any scripting evaluation and any use of administrative or config commands.
   This is the false-positive killer: an instance bound to loopback or an authenticated private network with
   commands built from fixed templates and bound values is not the bug. Name the reachability and the sinks
   first.

2. **Check exposure and authentication.** Confirm whether the instance requires authentication and whether it
   is reachable from outside the trusted boundary. An unauthenticated, reachable instance is exploitable
   directly: an attacker issues commands, and on a scriptable store reaches config changes, file rewrites, or
   module loads. Treat protected-mode defaults and network policy as part of this answer, not as a substitute
   for authentication.

3. **Trace untrusted input into command composition.** Follow request values into the commands the
   application issues. Separate input used as a value (a key or a stored value, generally safe with a proper
   client) from input that becomes a command name, a command argument that selects behavior, or a
   concatenated command in a protocol the store parses. Input that becomes a command rather than a value is
   the injection.

4. **Check scripting and administrative reach.** Server-side scripting (a Lua `EVAL` or a stored function),
   module loading, and config commands that set the working directory or file name are the code-execution and
   file-write paths. Determine whether untrusted input can reach a scripting evaluation or whether the
   application (or an exposed instance) permits the dangerous administrative commands at all.

5. **Check the defenses that actually stop it.** Requiring authentication and binding to a private interface,
   using a client that sends commands and arguments as discrete typed fields rather than a concatenated
   string, disabling or renaming dangerous administrative and scripting commands, and running the store as an
   unprivileged user with a read-only config each remove a vector. Determine which stands for this instance
   and this command path.

6. **Confirm and record.** Confirm by connecting within scope to show the instance answers commands without
   authentication, or by supplying an in-scope input that issues an unintended command (a benign
   configuration read or a marker key set via injection), without rewriting files or running a destructive
   script. Kill the lead if the instance requires authentication and is not reachable outside the boundary,
   commands are composed as discrete typed fields with input used only as values, and dangerous scripting and
   administrative commands are disabled. Record the reachability or the input, the command sink, and the
   unintended command reached.

## Where datastore exposure and abuse leak

- **An open instance is a command interface, not just data.** On a scriptable store, an unauthenticated port
  reaches config changes, file rewrites, and module loads, which is code execution.
- **Input that becomes a command is the injection.** A value is generally safe through a proper client; input
  that becomes a command name or a behavior-selecting argument is not.
- **Scripting is the code-execution path.** A Lua or stored-function evaluation reachable from untrusted input
  runs code inside the store.
- **Config and file commands rewrite the host.** Commands that set the directory and file name, then persist,
  can drop an attacker-controlled file where it will be executed.
- **Protected mode and network policy are not authentication.** They reduce exposure but a reachable instance
  without a required credential is still open; treat authentication as mandatory.

## Worked example (a confirm and a kill)

> **Confirm.** A cache instance is bound to a reachable interface with no authentication required. Connecting
> within scope, the instance answers administrative commands, and a benign configuration read confirms full
> command access; on this store the reachable command set includes config and scripting commands that would
> reach a file write. **Confirmed** unauthenticated datastore exposure to command execution, `critical`,
> remediation = require authentication, bind to a private interface unreachable from untrusted networks,
> disable or rename the dangerous administrative and scripting commands, and run the store as an unprivileged
> user.
>
> **Kill.** The instance requires authentication, is bound to a private interface reachable only from the
> application subnet, and has scripting and the dangerous config and module commands disabled; the application
> composes every command through a client that sends the command and arguments as discrete typed fields with
> untrusted input used only as key and value contents. No unauthenticated access and no input becomes a
> command. **Killed**, `kill_reason` = "instance authenticated and privately bound with dangerous commands
> disabled, and commands composed as typed fields with input used only as values; no open access and no
> command injection."

## Rationalizations to reject

- *"It is only a cache, there is nothing sensitive in it."* → On a scriptable store an open port reaches
  config, file, and scripting commands, so exposure is code execution, not just cache disclosure.
- *"It is on a private network."* → A private network is not authentication; confirm a credential is required
  and the bind address is not reachable from untrusted segments.
- *"We only put user data in keys and values."* → Values are generally safe through a proper client; the risk
  is input that becomes a command name or a scripting argument, which you must confirm separately.
- *"Protected mode is on."* → Protected mode reduces accidental exposure but is not a credential; require
  authentication explicitly.
- *"We do not use Lua or modules."* → Confirm they are disabled, not merely unused; an exposed instance
  permits them regardless of whether the application calls them.

## Executing this in practice

You need each datastore instance's bind address, exposure, and authentication requirement, every command
composition in the application, any scripting or administrative command reach, and whether dangerous commands
are disabled. For each instance, ask whether it is reachable without authentication; for each command sink,
whether input becomes a command or a script. A scoped connection shows whether the instance answers without a
credential; a benign injected command shows whether input reaches the command layer.

## Related

- `mapping-attack-surface` - use it to locate reachable datastore ports and services before auditing any one
  instance's command layer.
- `hunting-non-human-identity-and-secret-reachability` - an exposed cache often holds tokens and session data;
  that skill governs what a reachable instance leaks.
- `exploiting-ssrf-to-cloud-metadata` - a server-side request primitive can reach an internal datastore port;
  the two skills meet at the instance's reachability.
- `auditing-multi-tenant-isolation` - a shared cache across tenants is an isolation boundary; a command that
  reads another tenant's keys is the same class as an open instance.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the open port or the untrusted input, sink = the
  datastore command interface, evidence = the unauthenticated access or the composed dangerous command.
