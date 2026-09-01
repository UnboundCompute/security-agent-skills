---
name: hunting-code-interpreter-and-tool-sandbox-escape
description: >-
  Hunt for ways attacker-influenced code or a tool call escapes the sandbox an AI application runs it in: a
  code-interpreter or tool runtime that executes model-generated code with network access, a writable host
  filesystem, or credentials it should never see, a sandbox that shares a kernel, a mount, or an environment
  variable with the host so the guest reaches out, a resource with no CPU, memory, time, or output bound so one
  run starves the host, and a tool whose arguments reach a shell or a path outside the jail. Covers AI features
  that run model-produced code or shell in a sandbox: code interpreters, agent tool runtimes, and notebook or
  eval backends. Use when a model's output becomes code that executes and the sandbox is the boundary. The
  model-generated code or tool argument is the source, the host resource it reaches is the sink, and the missing
  isolation, credential, or resource bound that lets it out is the bug.
license: MIT
---

# Hunting code-interpreter and tool sandbox escape: the model writes code and something runs it

An AI application that runs model-generated code, a code interpreter, an agent tool that shells out, a notebook
backend, is executing attacker-influenceable instructions, because the model's output is shaped by its input
and its input can be attacker-controlled. The sandbox that runs that code is therefore a security boundary
around hostile code, and it is often built as if the code were trusted. The escapes are concrete. The runtime
may have network access, so generated code reaches internal services or exfiltrates. It may see a writable host
filesystem or a shared mount, so code reads or writes outside the jail. It may carry credentials, an API key,
a cloud role, a token, in its environment, so code steals them. The isolation may be thin, a shared kernel, a
container with a host mount, a subprocess with no namespace, so a known primitive escapes it. And with no CPU,
memory, time, or output bound, one run starves the host. The hunt is to run code in the sandbox and see what of
the host it can touch. You hunt this by making the model emit probing code and observing what succeeds.

## When to use

- An AI feature executes model-generated code or shell, or runs agent tools, inside a sandbox.
- The sandbox may have network access, a writable or shared filesystem, or credentials in its environment.
- Isolation may be thin (shared kernel or mount) or resource limits may be missing.

## Scope check

Test sandbox escape only on AI applications and runtimes you own or are authorized to assess, in a
non-production sandbox. Running probing code exercises real execution and can reach real resources, so use an
isolated test deployment and never touch data, credentials, or hosts that are not yours. If you can't name the
authorization, stop.

## The loop

1. **Establish the intended sandbox boundary first.** Name what the runtime is allowed to touch: no network or a
   narrow allowlist, an ephemeral filesystem with nothing host-shared, no ambient credentials, hard CPU/memory/
   time/output limits, and isolation strong enough for hostile code. This is the false-positive killer: a
   runtime with no egress, no host mount, no credentials in scope, enforced resource bounds, and real isolation
   is behaving correctly. Name the intended boundary, then test each edge.

2. **Test network egress from the runtime.** Have the model emit code that opens a connection: to an external
   collector you control, to an internal address, and to a cloud metadata endpoint. Confirm egress is blocked or
   allowlisted. A runtime with open network access lets generated code exfiltrate and pivot inside.

3. **Test filesystem reach.** Run code that lists, reads, and writes outside the working directory: the host
   root, a shared mount, other users' or sessions' data, and the runtime's own configuration. Confirm the
   filesystem is ephemeral and unshared. A writable or shared host filesystem lets code read secrets or persist
   across the boundary.

4. **Hunt for ambient credentials.** Enumerate the environment, instance metadata, mounted service-account
   tokens, and config files reachable from the runtime. Confirm no usable credential is in scope. A cloud role,
   API key, or token the sandbox can read turns code execution into credential theft and lateral movement.

5. **Test isolation strength and resource bounds.** Probe the isolation: whether it is a shared kernel, a
   container with a dangerous mount or capability, or a bare subprocess, and whether a known escape primitive
   works. Separately, run code that spins CPU, allocates memory, sleeps, and emits huge output, and confirm each
   is bounded. Thin isolation escapes to the host; missing bounds let one run starve it.

6. **Confirm and record.** Confirm by escaping the sandbox from model-driven code: reaching the network, reading
   or writing the host filesystem, stealing a credential, or breaking isolation, on a non-production runtime and
   without touching real resources. Kill the lead if the runtime has no egress, no host-shared or writable
   filesystem, no ambient credentials, enforced resource bounds, and isolation that holds against a known
   primitive. Record the model-generated code or tool argument, the host resource it reached, and the missing
   isolation, credential, or resource bound.

## Where the sandbox leaks

- **Network access from the runtime.** Egress lets generated code exfiltrate to the outside and reach internal
  services and metadata endpoints.
- **A writable or shared host filesystem.** A host mount, a shared volume, or a writable root lets code read
  secrets and persist beyond the run.
- **Ambient credentials in scope.** An API key, cloud role, or service-account token the runtime can read turns
  execution into credential theft.
- **Thin isolation.** A shared kernel, a dangerous capability or mount, or a bare subprocess is escapable by a
  known primitive, so the guest becomes the host.
- **Missing resource bounds.** No CPU, memory, time, or output limit lets one model-driven run starve or hang
  the host and neighboring sessions.

## Worked example (a confirm and a kill)

> **Confirm.** A code-interpreter feature runs model-generated Python in a container that has outbound network
> access and a mounted cloud service-account token. A prompt that steers the model into emitting code which
> reads the token file and posts it to an external endpoint succeeds, exfiltrating a credential that grants cloud
> access. **Confirmed** sandbox credential theft via egress and mounted token, `critical`, remediation = remove
> ambient credentials from the runtime, block network egress by default, and mount no host secret into the
> code-execution sandbox.
>
> **Kill.** The runtime runs generated code with no network egress, an ephemeral filesystem with no host mount
> or shared volume, no credentials or metadata reachable in its environment, hard CPU/memory/time/output limits,
> and isolation that resists a known escape primitive. Probing code cannot reach the network, the host
> filesystem, a credential, or the host. **Killed**, `kill_reason` = "no egress, no host-shared filesystem, no
> ambient credentials, enforced resource bounds, and isolation holds; model-driven code touches nothing outside
> the jail."

## Rationalizations to reject

- *"It only runs the user's own code."* → The model's output is shaped by attacker-influenceable input; treat
  everything the runtime executes as hostile and bound it accordingly.
- *"It is in a container."* → A container is not a sandbox by itself; confirm no host mount, no dangerous
  capability, no egress, and no known escape primitive.
- *"There are no secrets in there."* → Enumerate the environment, metadata, and mounted tokens from inside the
  runtime; ambient credentials are the most common one people forget.
- *"We need network for it to be useful."* → Then allowlist specific destinations; open egress lets generated
  code exfiltrate and reach internal services and metadata.
- *"Runs are short."* → Without enforced CPU, memory, time, and output bounds a single run can still starve or
  hang the host; short is not bounded.

## Executing this in practice

You need the runtime's network policy, its filesystem view and any host sharing, the credentials reachable from
inside, the isolation mechanism, and the resource limits. Drive the model to emit code that probes each: an
egress beacon, a host-path read and write, a credential enumeration, and a resource-exhaustion loop. Reading the
sandbox configuration shows the intended boundary; code that reaches the network, the host, or a credential
shows whether it holds.

## Related

- `auditing-ai-agent-permissions` - the agency and approval layer above the runtime; excessive agency plus a
  weak sandbox is how a tool call becomes host access.
- `auditing-mcp-tool-integrations` - tools an agent calls are the other execution surface; their arguments reach
  the same shells and paths this skill jails.
- `hunting-container-escape-surface` - the container-isolation mechanics a code sandbox depends on; the escape
  primitives are shared.
- `auditing-ml-inference-endpoint-abuse` - the model serving that produces the code; abusing the endpoint and
  escaping the runtime are the two halves of the AI-execution surface.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the model-generated code or tool argument, sink = the
  host resource it reaches, evidence = the missing isolation, credential, or resource bound.
