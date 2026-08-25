---
name: auditing-editor-extension-workspace-trust
description: >-
  Audit an editor or IDE extension for actions it runs on untrusted workspace contents, after the
  workspace-trust capability and activation events are resolved. Covers a task, debug preLaunchTask, or
  command that auto-runs on folder open, a tool path or command template read from workspace settings and
  executed, a workspace-trust bypass where a risky operation runs in restricted mode, a language server
  that loads or executes a workspace-declared binary, repo-controlled data reaching a shell or
  task-execution sink, and an extension webview that renders repo content without a content-security
  policy. Use when reviewing an extension manifest (activation events, the untrusted-workspace
  capability), its workspace config handling, or its language-server tool resolution. A cloned untrusted
  repository is the source, an action the extension auto-runs on open is the sink, and repo-controlled
  code reaching execution without a trust gate is the bug.
license: MIT
---

# Auditing editor extension workspace trust: what an untrusted repository makes the editor do on open

An editor extension turns dangerous the moment merely opening a cloned repository makes it run code the
repository controls. The threat is not a user choosing to run something; it is a drive-by, a hostile
`.vscode/tasks.json`, a `settings.json` that names the compiler binary, an activation event that fires on
folder open and shells out before the user has trusted anything. You audit it by resolving, per risky
action, whether a workspace-trust gate or an explicit user gesture stands between the untrusted repo and
the execution sink. The discipline is separating an action that genuinely fires on open from one that
needs a deliberate click, and a value read from workspace settings from one read from user or global
settings a repo cannot set.

## When to use

- You are reviewing an editor or IDE extension manifest, its activation events, and its trust capability.
- The extension reads workspace configuration, runs tasks, or resolves a tool or binary path from a repo.
- You want to know whether opening a hostile repository executes code before the user trusts it.

## Scope check

Audit only extensions you own or are authorized to assess, and open a crafted hostile workspace only on a
machine and account in scope, an auto-executed task runs real code on your host. Adjudicate on the
manifest and the handler. If you can't name the authorization, stop.

## The loop

1. **Resolve the trust capability and activation events first.** Read the manifest for the
   untrusted-workspace capability (does it declare it unsupported, limited, or fully supported) and for
   the activation events. An event like folder-open, a startup-finished event, or a workspace-contains
   glob makes the extension activate on any repo; a narrow language or command event does not. The gate
   and the activation trigger decide whether a sink is drive-by reachable, so settle both before flagging.

2. **Check auto-run tasks and debug launches.** Look for the extension running a task, a command, or a
   debug preLaunchTask on activation or on open, before or without a trust gate. A task defined in the
   workspace that fires automatically is repo-controlled code executing on open.

3. **Check tool and command paths read from workspace settings.** Look for a compiler, linter, formatter,
   or runner path, or a command template, read from workspace-scoped settings and then executed. A repo
   that ships a `settings.json` naming the binary chooses what runs; the same value read from user or
   global settings is not repo-controlled.

4. **Check the workspace-trust bypass.** Look for a risky operation that still runs when the workspace is
   untrusted or in restricted mode, or that trusts a repo-supplied directive the core mis-validates. An
   extension that declares trust support but performs the dangerous action anyway defeats the gate.

5. **Check language-server execution and webviews.** Look for a language server that loads or executes a
   workspace-declared binary or plugin on open, for repo-controlled data (env, args, cwd) reaching a
   shell or task-execution call, and for an extension webview with scripting enabled that renders
   repo-controlled content without a content-security policy or message-origin validation.

6. **Confirm and record.** Confirm by opening a crafted hostile workspace and showing the extension
   executes repo-controlled code, or resolves and runs a repo-specified binary, with no trust gate and no
   explicit user action. Kill the lead if the extension declares the untrusted-workspace capability
   unsupported (or gates the risky feature when limited) so it refuses to run untrusted, if the action
   requires an explicit user gesture (a palette command, a button) rather than firing on open, if the
   tool path or command is read only from user or global settings a repo cannot set, if the activation
   event is narrow and its body does no execution, if the webview uses a strict content-security policy
   and validates message origins, or if the repo value is used only as display or config data and never
   reaches an execution sink. Record the extension, the trigger, and the sink the repo reaches.

## Where workspace trust leaks

- **The trigger decides drive-by, not the sink alone.** A shell call is only a workspace-trust bug when an
  on-open activation or an auto-run task reaches it; behind an explicit gesture it is user intent.
- **Workspace settings are attacker-writable; user settings are not.** A tool path read from the repo's
  configuration is repo-controlled; the same key read from the global scope is out of the attacker's reach.
- **Declaring trust support and then bypassing it is the bug.** An extension that runs the dangerous action
  in restricted mode gives the gate away; the capability has to actually withhold the feature.
- **A broad activation event is the on-ramp.** Folder-open, startup-finished, and workspace-contains globs
  activate on any repo; a language-specific or command-specific event does not.
- **A scripting webview rendering repo content needs a policy.** Without a strict content-security policy
  and origin-checked messages, repo-controlled markup becomes an extension-context injection into a bridge.

## Worked example (a confirm and a kill)

> **Confirm.** An extension activates on folder open and, on activation, reads a `runner` command from the
> workspace settings and executes it to index the project. A cloned repository ships a `settings.json`
> setting `runner` to an attacker command. Opening the repo runs it, with no trust prompt and no user
> action. **Confirmed** repo-controlled command execution on open with no trust gate, `high` rising to
> `critical`, remediation = gate the feature behind workspace trust, read the runner from user or global
> settings only, and require an explicit gesture to execute.
>
> **Kill.** The same extension declares the untrusted-workspace capability unsupported, so on an untrusted
> folder it activates in restricted mode and does not run the indexer until the user trusts the workspace.
> Opening a hostile repo executes nothing. **Killed**, `kill_reason` = "the extension declares untrusted
> workspaces unsupported and withholds the execution path until the user grants trust; no drive-by sink."

## Rationalizations to reject

- *"The user has to open the folder."* -> Opening a folder is the whole attack; if activation runs code on
  open, cloning and opening a hostile repo is the drive-by. Ask what fires without a further gesture.
- *"It only reads a setting."* -> From which scope? A workspace-scoped setting is repo-controlled; if that
  value becomes a command or a binary path, reading it is reaching a sink.
- *"We support workspace trust."* -> Declaring support is not enforcing it; confirm the risky feature is
  actually withheld in restricted mode, not run anyway.
- *"The activation event is fine."* -> Is it folder-open, startup-finished, or a broad glob? Those activate
  on any repo; only a narrow event with an inert body is safe.
- *"The webview just shows repo files."* -> With scripting enabled and no content-security policy, rendered
  repo content is an injection into the extension context; check the policy and the message origins.

## Executing this in practice

You need the manifest (activation events and the untrusted-workspace capability), the activation handler,
every place a task, command, or debug launch can auto-run, every tool or command path and the settings
scope it is read from, the language-server tool resolution, and any webview and its content-security
policy. For each risky action, decide whether a trust gate or an explicit gesture stands between the
untrusted repo and the sink. Reading the manifest tells you what activates on open; reading the handler
tells you what the repo can make it run.

## Related

- `auditing-browser-extension-trust` - the sibling client-surface audit for a browser extension, where the
  untrusted source is a web page or another extension rather than a cloned repository.
- `auditing-electron-ipc-trust` - the desktop-app sibling, where untrusted renderer content rather than a
  workspace reaches a native capability across the preload bridge.
- `vetting-skills-before-install` - audits the skill or MCP artifact you install into an agent; this skill
  audits an editor extension's handling of an untrusted workspace it opens, a different source and sink.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = a cloned untrusted repository, sink = an action
  the extension auto-runs on open, evidence = the resolved trust gate, the activation trigger, and the
  executed repo-controlled code.
