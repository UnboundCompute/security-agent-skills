---
name: auditing-container-runtime-and-socket-exposure
description: >-
  Audit whether a workload can reach the container runtime and thereby control the host: the container runtime
  socket mounted into a pod or bound into a build or CI container, a privileged sidecar that talks to the
  runtime to launch or inspect containers, a runtime API exposed over a reachable port, and tooling that
  needs the runtime and is given it far more broadly than the one operation requires. Covers container hosts
  where reaching the runtime socket or API grants the ability to start privileged containers, mount the host
  filesystem, and take over the node. Use when a workload, build, or agent is given access to the container
  runtime. The workload with runtime access is the source, the container runtime is the sink, and the socket
  or API exposure that grants host control is the bug.
license: MIT
---

# Auditing container runtime and socket exposure: a handle on the runtime is a handle on the host

Access to the container runtime is access to the host, full stop. Whoever can talk to the runtime socket or
API can start a new container that is privileged, mounts the host root filesystem, and shares the host
namespaces, and from there they own the node. So mounting the runtime socket into a pod, or binding it into a
build or CI container, is not a convenience; it is handing that workload the ability to escape to the host by
design, no kernel bug required. The pattern shows up wherever tooling genuinely needs the runtime, image
builds, node agents, CI that builds containers, and the mistake is granting broad runtime access for a narrow
need. The audit is direct: find every workload that can reach the runtime and treat that reach as host
control unless it is tightly brokered. You audit this by locating runtime-socket mounts and runtime-API
reachability and confirming nothing untrusted holds them.

## When to use

- A workload, build container, or agent is given the container runtime socket or access to the runtime API.
- A sidecar or tool talks to the runtime to build, launch, or inspect containers.
- The runtime API may be exposed over a reachable port rather than a local, access-controlled socket.

## Scope check

Audit runtime exposure only on hosts and clusters you own or are authorized to assess, on non-production
nodes. Demonstrating runtime access can launch a container that controls the host, so use a non-production
node and stop at proof rather than taking over the node. If you can't name the authorization, stop.

## The loop

1. **Establish which workloads legitimately need the runtime first.** Name the workloads that genuinely must
   talk to the runtime and exactly what operation each needs. This is the false-positive killer: a workload
   that needs no runtime access and is given none is fine, and one that needs a narrow operation brokered
   through a constrained intermediary is far better than a raw socket. Name the legitimate need, then find
   raw or broad runtime access.

2. **Find runtime-socket mounts.** Identify every pod or container that mounts the container runtime socket,
   including build containers, CI runners, node agents, and sidecars. A mounted runtime socket lets the
   workload issue any runtime command, so each mount is a candidate host-control grant regardless of what the
   workload appears to do.

3. **Check runtime-API reachability.** Determine whether the runtime API is reachable over a network port
   rather than only a local socket, and whether that port is authenticated and access-controlled. A runtime
   API reachable from a pod or the network without strong authentication is host control for anyone who reaches
   it.

4. **Assess what the access grants.** For each workload with runtime reach, confirm what it can do: start a
   privileged container, mount the host filesystem, share host namespaces, or inspect and control other
   containers. Runtime access is near-total host control by default, so the severity is high unless the access
   is brokered down to a single safe operation.

5. **Check whether the access is brokered and constrained.** Where runtime access is genuinely needed, confirm
   it is mediated: a constrained builder or a brokering service that permits only the specific, safe operations
   rather than raw socket access, running with the least privilege the task needs. Raw socket access handed to
   a workload is unbrokered host control; a narrow, mediated interface is not.

6. **Confirm and record.** Confirm by using a workload's runtime access to launch a benign marker container
   that demonstrates host reach (for example a container that could mount the host filesystem), on a
   non-production node and without taking over the node. Kill the lead if no untrusted workload mounts the
   runtime socket or reaches an unauthenticated runtime API, and any legitimate runtime need is brokered to a
   single constrained operation. Record the workload source, the container-runtime sink, and the socket or API
   exposure that granted host control.

## Where runtime exposure leaks

- **A mounted socket is host control.** A container with the runtime socket can start a privileged, host-
  mounting container and own the node, no exploit required.
- **CI and build containers are common holders.** Container builds and CI runners are often handed the runtime
  socket, making the build environment a host-takeover path.
- **A network-reachable runtime API is worse than a socket.** An exposed, weakly authenticated runtime API
  hands host control to anyone who can reach the port.
- **The narrow need becomes broad access.** Tooling that needs one runtime operation is frequently given raw
  socket access to the whole runtime.
- **Runtime access ignores the pod's security context.** Even an otherwise unprivileged pod owns the host if it
  can tell the runtime to launch a privileged one.

## Worked example (a confirm and a kill)

> **Confirm.** A CI runner pod mounts the container runtime socket so it can build images. A job on that runner,
> standing in for a compromised build, uses the socket to launch a new privileged container that mounts the host
> root filesystem, reaching node state and other pods' data. **Confirmed** host control via mounted runtime
> socket, `critical`, remediation = remove the runtime-socket mount, build images with a rootless, daemonless
> builder that needs no runtime access, or broker the build through a constrained service that permits only the
> build operation.
>
> **Kill.** No workload mounts the container runtime socket, the runtime API is bound to a local, access-
> controlled socket unreachable from pods and the network, and the one build workload that needs to produce
> images uses a rootless builder that never talks to the host runtime. Nothing untrusted can issue a runtime
> command. **Killed**, `kill_reason` = "no runtime-socket mount and no reachable runtime API, with image builds
> done by a rootless builder; no workload can reach the runtime to control the host."

## Rationalizations to reject

- *"The build needs the socket."* → Modern rootless, daemonless builders build images without the runtime
  socket; a mounted socket is host control, not a build requirement.
- *"It is only a sidecar."* → A sidecar with runtime access can launch a privileged container and own the node
  regardless of the main container's constraints.
- *"The runtime API is on localhost."* → Confirm it is unreachable from pods and the network and is
  authenticated; a reachable runtime API is host control for whoever reaches it.
- *"The pod is unprivileged."* → Runtime access bypasses the pod's own security context; it can tell the runtime
  to start a privileged container.
- *"Only our tooling uses it."* → If the tooling is compromised or the socket is reachable, the access is host
  control; broker it down to the one operation instead.

## Executing this in practice

You need every workload that mounts the runtime socket or reaches the runtime API, whether that API is network-
reachable and authenticated, what each access grants, and whether any legitimate need is brokered to a narrow
operation. For each, treat runtime reach as host control unless it is mediated. Reading the pod specs and
runtime configuration shows the granted access; launching a benign marker container through it shows the host
reach it confers.

## Related

- `hunting-container-escape-surface` - the broader escape audit; the runtime socket is the most direct and
  severe single item in it.
- `auditing-host-mount-and-device-exposure` - the sibling host-access vector; a runtime socket is one host mount
  whose blast radius is the whole node.
- `hunting-cicd-workflow-injection` - CI runners with runtime access are a prime target; attacker-controlled
  build input plus a mounted socket is host takeover.
- `mapping-pod-to-cloud-credential-reach` - host control via the runtime then reaches the node role and cloud
  credentials; the two compound.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the workload with runtime access, sink = the container
  runtime, evidence = the socket or API exposure that granted host control.
