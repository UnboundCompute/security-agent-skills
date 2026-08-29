---
name: auditing-ansible-become-and-vault-trust
description: >-
  Audit configuration-management privilege escalation and secret handling for trust that runs as root on
  every managed host: a task that escalates with become across a whole play when only one step needs it, a
  role or variable sourced from an untrusted place that runs under that escalation, a vault-encrypted secret
  whose decryption key is exposed to the runner or logged, and a templated value or module argument that
  takes attacker-influenceable input while privileged. Covers Ansible and similar agentless tools where a
  control node runs plays that escalate privilege and decrypt secrets across a fleet. Use when playbooks
  escalate with become or handle vault secrets across managed hosts. The untrusted role, variable, or input
  running under escalation is the source, the privileged task or decrypted secret is the sink, and the
  over-broad escalation or exposed key is the bug.
license: MIT
---

# Auditing Ansible become and vault trust: when the control node runs as root everywhere

An agentless configuration tool is a fleet-wide root shell with a scheduler. The control node connects to
every managed host and runs tasks that escalate privilege, and it decrypts the secrets those tasks need. Two
trust questions decide the blast radius. First, escalation scope: become is often set at the play or role
level so every task runs privileged, and any role, variable, or templated input that flows into those tasks
runs as root on every host it touches. Second, secret handling: vault-encrypted values are only as protected
as the decryption key, and a key exposed to the runner, passed on a command line, or logged is a fleet-wide
secret leak. When a play escalates and decrypts across many hosts, an untrusted role or an exposed key is not
one host's problem. You audit this by scoping where escalation applies and tracing what runs under it, and by
checking how vault secrets are decrypted and whether the key or the plaintext leaks.

## When to use

- Playbooks escalate privilege with become across managed hosts, often at the play or role level.
- Roles, collections, or variables come from shared or external sources and run under that escalation.
- Vault-encrypted secrets are decrypted on the control node, and the key or plaintext handling may leak.

## Scope check

Audit playbooks and vault handling only for fleets and control nodes you own or are authorized to assess, on
non-production hosts. Running a play escalates on real hosts and decrypts real secrets, so use a non-
production inventory and never exfiltrate a decrypted secret. If you can't name the authorization, stop.

## The loop

1. **Establish where escalation legitimately applies first.** Name which tasks genuinely need privilege and on
   which hosts, versus where become is set broadly out of convenience. This is the false-positive killer:
   escalation scoped to the specific tasks that need it, running only first-party reviewed roles, is correct.
   Name the legitimate escalation, then find where it is broader.

2. **Scope where become actually applies.** Determine whether escalation is set at the task level for the few
   steps that need it, or at the play or role level so every task runs privileged. Play-level become means every
   role, include, and templated task in that play runs as root, widening what an untrusted input can do. Map the
   actual escalation scope against the legitimate one.

3. **Trace untrusted roles, collections, and variables into escalated tasks.** Follow every role, collection,
   include, and variable source that runs under escalation. A role pulled from a shared or external source, or a
   variable file an attacker can influence, executes with root on every host when it runs in an escalated play.
   Distinguish first-party reviewed content from external content running privileged.

4. **Check templated input and module arguments under escalation.** A task that templates an attacker-
   influenceable value into a command, a shell module, a file path, or a module argument runs that input as
   root. Find where escalated tasks build commands or arguments from variables, facts, or inventory an attacker
   could shape, since that is privileged injection on the managed host.

5. **Audit vault key exposure and secret handling.** Determine how vault secrets are decrypted: where the vault
   password or key lives, whether it is passed on a command line or in an environment the runner exposes, and
   whether decrypted values are logged, displayed by a task, or written to a readable file. The key protects
   every vault secret, so an exposed key or a logged plaintext is a fleet-wide disclosure.

6. **Confirm and record.** Confirm by showing an untrusted role or templated input runs under escalation on a
   non-production host, or that a vault secret's key is exposed or its plaintext logged, within scope and without
   exfiltrating secrets or altering real hosts. Kill the lead if escalation is task-scoped to what needs it, only
   reviewed first-party content runs privileged, no attacker input is templated into a privileged task, and vault
   keys and plaintext are never exposed or logged. Record the untrusted source, the privileged task or decrypted
   secret sink, and the over-broad escalation or exposed key.

## Where become and vault trust leak

- **Play-level become escalates everything.** Setting escalation once at the play or role level runs every task
  in it as root, not just the step that needed it.
- **An untrusted role runs privileged fleet-wide.** A shared or external role executing under escalation is
  root-level code on every host the play targets.
- **Templated input becomes a privileged command.** An attacker-influenceable variable rendered into a shell or
  command module under become is injection running as root.
- **The vault key protects every secret.** A decryption key exposed to the runner, passed on a command line, or
  stored beside the vault is a fleet-wide secret leak.
- **Decrypted plaintext leaks through logs.** A task that displays or logs a vault value, or writes it to a
  world-readable file, exposes the secret the vault was meant to protect.

## Worked example (a confirm and a kill)

> **Confirm.** A play sets become at the play level and includes a role pulled from a shared external source. A
> task in that role templates an inventory-supplied variable into a shell command. On a non-production host, a
> crafted variable runs a benign marker command as root, and the play's vault password is passed via an
> environment variable visible in the process list. **Confirmed** over-broad escalation running untrusted input
> as root plus vault-key exposure, `high`, remediation = scope become to the specific tasks that need it, run
> only reviewed first-party roles under escalation, never template attacker-influenceable input into a shell
> module, and supply the vault key through a protected prompt or secrets integration rather than a visible
> environment or command line.
>
> **Kill.** Escalation is set per-task only on the steps that require it, every role and collection is
> first-party and reviewed, no escalated task templates inventory or external input into a command or module
> argument, and vault secrets are decrypted with a key supplied through a protected channel and never logged or
> displayed. A crafted inventory value reaches no privileged command and the vault key is not exposed. **Killed**,
> `kill_reason` = "become task-scoped to what needs it with only reviewed first-party roles privileged, no
> attacker input templated into a privileged task, and vault key and plaintext never exposed; no fleet-wide
> escalation or secret leak."

## Rationalizations to reject

- *"We set become once for the whole play, it is simpler."* → Every task then runs as root; scope escalation to
  the steps that need it so an untrusted input cannot ride it fleet-wide.
- *"The role is from the community."* → A shared role under escalation is root-level code on every host; review
  and pin it, or do not run it privileged.
- *"The variable comes from inventory, not a user."* → Inventory and facts can be attacker-influenceable; a value
  templated into a privileged command is injection as root.
- *"The secret is vault-encrypted."* → Vault protects data at rest only; if the key is exposed or the plaintext
  is logged, the encryption bought nothing.
- *"We pass the vault password in the environment."* → A runner environment or command line is often visible;
  supply the key through a protected prompt or secrets integration.

## Executing this in practice

You need the legitimate escalation scope, where become actually applies, every role and variable source running
under it, the templated inputs and module arguments in escalated tasks, and the vault key and plaintext handling.
For each play, compare the escalation scope to what needs privilege and trace untrusted content and input into
privileged tasks; for vault, confirm the key is protected and no plaintext is logged. Reading the playbooks and
vault setup shows the intended trust; running a play on a non-production host shows whether escalation and secret
handling hold.

## Related

- `auditing-cicd-oidc-trust` - the control node's own identity and the pipeline that runs plays; escalation on
  hosts and federated identity in CI are two halves of the same privileged automation.
- `hunting-non-human-identity-and-secret-reachability` - the vault key and the control node's credentials are
  machine secrets; that skill traces what an exposed one reaches.
- `hunting-cicd-workflow-injection` - the same attacker-input-into-a-privileged-run threat from the pipeline
  side, where here the privileged run is a fleet-wide play.
- `auditing-iac-module-and-provider-supply-chain` - the declarative-provisioning companion; roles and
  collections are the configuration-management analog of modules and providers.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the untrusted role, variable, or input under
  escalation, sink = the privileged task or decrypted secret, evidence = the over-broad escalation or exposed key.
