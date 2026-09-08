---
name: auditing-windows-named-pipe-and-rpc-exposure
description: >-
  Audit local privilege escalation through Windows inter-process interfaces, where a privileged service
  exposes a named pipe, remote procedure call endpoint, or local COM server that a lower-privileged caller can
  reach and that performs a privileged action, impersonates the caller unsafely, or trusts caller-supplied
  input without authorization, letting a standard user drive privileged behavior. Use when a service listens
  on a named pipe, an RPC interface, or a COM server reachable by non-administrators. Covers weak interface
  permissions, unsafe impersonation of a connecting client, missing caller authorization, and privileged
  actions exposed to low-privileged callers. The lower-privileged call into the interface is the source, the
  privileged action the service performs is the sink, and driving privileged behavior as a standard user is
  the bug.
license: MIT
---

# Auditing Windows named pipe and RPC exposure: when a local endpoint hands out privilege

A privileged Windows service often exposes an interface for others to talk to it: a named pipe, a remote
procedure call endpoint, or a local COM server. Those interfaces are safe only when the service restricts who
may connect and checks what each caller is authorized to ask for, and only when it does not blindly assume the
identity of whoever connects. The exposure is an interface reachable by a lower-privileged caller that then
performs a privileged action on the caller's behalf, or impersonates the connecting client in a way that lets
a crafted caller flip the trust, or trusts caller-supplied input, a path, a command, a target, without
checking the caller's authorization. In each case a standard user drives the privileged service into doing
something the user could not do directly. The bug is a privileged action reachable across an inter-process
boundary without the authorization that boundary should enforce. You audit these by listing each service's
interfaces, who can reach them, and what each exposed operation does with the caller's request.

## When to use

- A privileged service exposes a named pipe, RPC endpoint, or local COM server reachable by non-administrators.
- An exposed operation performs a privileged action or acts on caller-supplied input.
- A service impersonates connecting clients and a crafted caller may exploit that impersonation.

## Scope check

Audit inter-process interfaces only on hosts you own or are authorized to assess, on non-production or a
snapshot, using a benign proof that the privileged action was driven rather than performing a damaging one. A
confirmed exposure lets a standard user drive privileged behavior, so keep every probe within scope. If you
can't name the authorization, stop.

## The loop

1. **Establish whether the interface restricts callers and authorizes each operation first.** For each
   exposed interface, determine whether its permissions limit who may connect and whether the service checks
   the caller's authorization for each operation, or whether a lower-privileged caller can both reach it and
   invoke a privileged action. This is the false-positive killer: an interface restricted to administrators,
   or one that authorizes each operation against the caller, is not driven by a standard user however
   powerful its actions. Name the reachable operation and the missing check before crafting a call.

2. **Enumerate the service's interfaces and their permissions.** List the named pipes, RPC endpoints, and
   local COM servers each privileged service exposes, and read the access control on each: who may connect,
   who may invoke. Interfaces reachable by standard users or broad groups are the candidates.

3. **Map each reachable operation to what it does.** For each operation a low-privileged caller can invoke,
   determine whether it performs a privileged action, writes a protected location, runs or configures
   something, or returns protected data, and whether it acts on caller-supplied input such as a path, a
   target, or a command.

4. **Check the caller authorization.** Determine whether the service verifies the caller is entitled to the
   operation and its arguments, or whether reaching the interface is treated as sufficient authority. An
   operation that trusts any connected caller, or trusts caller-supplied input without checking authorization,
   is the exposure.

5. **Check the impersonation behavior.** Determine whether the service impersonates the connecting client and,
   if so, whether a crafted caller can cause the service to act with more privilege than the caller holds, or
   whether the service performs a privileged action without dropping to the caller's context where it should.
   Unsafe impersonation flips the trust in either direction.

6. **Confirm and record.** Confirm by connecting as a lower-privileged caller and driving the exposed
   operation to perform a benign privileged action on an isolated host. Kill the lead if the interface
   restricts callers to administrators, if each operation authorizes the caller and its input, and if
   impersonation neither over-grants nor under-drops privilege. Record the interface, the reachable operation,
   the missing check, and the benign proof, or set a `kill_reason`.

## Where interface exposure leaks

- **The reachable privileged operation is the finding.** An interface a standard user can reach that performs
  a privileged action without authorizing the caller is the escalation. Name the operation and the missing
  check.
- **Interface permissions drift open.** A named pipe, RPC endpoint, or COM server whose access control was
  loosened for an application lets non-administrators connect to a privileged service.
- **Reaching is treated as authorization.** An operation that assumes any connected caller is entitled runs
  privileged actions for whoever gets in; connectivity is not authorization.
- **Caller-supplied input steers the action.** An operation that acts on a caller-supplied path, target, or
  command without checking authorization lets the caller direct the privileged action.
- **Unsafe impersonation flips trust.** A service that impersonates a crafted caller may act with the wrong
  privilege, either over-granting to the caller or performing a privileged action it should have dropped.

## Worked example (a confirm and a kill)

> **Confirm.** A privileged service exposes a named pipe reachable by standard users, and one operation writes
> a file to a caller-supplied path without checking the caller's authorization. A lower-privileged caller
> drives it to write a benign marker into a protected location the caller could not write directly, on an
> isolated host. **Confirmed** local privilege escalation through an unauthorized privileged operation on a
> reachable interface, `high`, remediation = restrict the interface permissions to the callers that need it,
> authorize each operation and its arguments against the caller, and validate caller-supplied paths and targets
> before acting.
>
> **Kill.** The same service restricts the pipe to administrators, authorizes each operation against the
> caller's token, validates caller-supplied paths, and impersonates the caller so privileged actions run only
> in the caller's own context. A standard user can neither reach the interface nor drive a privileged action.
> **Killed**, `kill_reason` = "the interface is restricted to administrators and every operation authorizes the
> caller and its input; a standard user reaches no privileged action."

## Rationalizations to reject

- *"It is only a local interface."* -> Local is exactly the escalation surface; a standard user on the host
  reaches the interface, and the question is whether it authorizes them, not whether it is remote.
- *"The service checks that a client connected."* -> Connecting is not authorization; confirm the service
  checks the caller is entitled to the specific operation and its arguments.
- *"Impersonation makes it safe."* -> Only if the impersonation actually confines the action to the caller's
  privilege; a crafted caller or a missed drop can flip the trust the other way.
- *"Only our application calls this pipe."* -> If the interface permissions allow standard users, anyone on
  the host can call it; confirm the access control, not the intended client.
- *"The operation just takes a path."* -> A caller-supplied path into a privileged write is precisely the
  lever; validate the path and authorize the caller rather than trusting the argument.

## Executing this in practice

You need every named pipe, RPC endpoint, and local COM server a privileged service exposes, the access
control on each, the actions each reachable operation performs, whether it authorizes the caller and its
input, and how it impersonates connecting clients. For each reachable operation, decide whether a
lower-privileged caller can drive a privileged action without the authorization the boundary should enforce.
Reading the permissions and the operation's handling settles most leads; connecting as a lower-privileged
caller and driving a benign privileged action on an isolated host settles the rest.

## Related

- `hunting-ntlm-coercion-and-relay` - the interfaces that coercion triggers abuse to induce outbound
  authentication are the local endpoints this skill audits, meeting on the exposed interface.
- `hunting-windows-service-privilege-escalation` - the privileged service behind the interface is the same
  service whose configuration that skill audits, a companion route to the same account.
- `auditing-windows-token-and-privilege-abuse` - unsafe impersonation on an interface captures a caller's
  token, the impersonation lever that skill audits from the privilege side.
- `auditing-electron-ipc-trust` - a renderer-to-main IPC boundary that trusts a caller without authorization
  is the application-layer cousin of the operating-system interfaces this skill audits.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the lower-privileged call into the interface, sink =
  the privileged action the service performs, evidence = a standard user driving a benign privileged action
  through the interface on an isolated host.
