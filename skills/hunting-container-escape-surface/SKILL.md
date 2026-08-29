---
name: hunting-container-escape-surface
description: >-
  Hunt for the configuration that lets a workload break out of its container onto the node: a pod that runs
  privileged or adds dangerous capabilities, a host namespace shared into the container (host PID, network,
  or IPC), a writable host path or device mounted in, and a security context that disables the defenses that
  would otherwise keep a process inside its container. Covers Kubernetes pods and standalone containers where
  a compromised or hostile process inside the container tries to reach the node, the kubelet, or other pods.
  Use when workloads run with elevated security contexts, host mounts, or shared host namespaces. The
  container process is the source, the node or host resource it reaches is the sink, and the escape the
  security context permits is the bug.
license: MIT
---

# Hunting container escape surface: when the container boundary is the security boundary

A container is only an isolation boundary while its security context keeps it one. Treat the container as a
box a process might try to climb out of, and the question is which settings hand it a ladder. A privileged
pod, an added capability that grants kernel-level power, a host namespace shared in, a writable host path,
or a mounted device each turns a container-local compromise into node access, and node access is every other
pod on that node. The hunt is not about a bug in the application; it is about the runtime configuration that
decides whether a foothold stays contained. You hunt this by inventorying the security context of each
workload and finding where a setting lets the process reach the node.

## When to use

- Workloads run with elevated security contexts: privileged mode, added capabilities, or host access.
- Pods share host namespaces (host PID, host network, host IPC) or mount host paths and devices.
- A compromise inside a container would be far worse if it could reach the node or neighboring pods.

## Scope check

Test escape surface only on clusters and nodes you own or are authorized to assess, on non-production
workloads. Demonstrating an escape reaches the real node and can affect other pods, so use a non-production
node and stop at proof rather than pivoting. If you can't name the authorization, stop.

## The loop

1. **Establish the intended isolation first.** Name what each workload is supposed to be able to touch: its
   own filesystem and network, nothing on the node. This is the false-positive killer: a pod that runs
   unprivileged, drops capabilities, shares no host namespace, and mounts no host path is contained by design,
   and an elevated setting it genuinely needs and constrains is not automatically a finding. Name the intended
   boundary, then find where the security context exceeds it.

2. **Inventory privileged mode and capabilities.** For each workload, read whether it runs privileged and
   which Linux capabilities it adds. Privileged mode removes most isolation at once; individual capabilities
   like the ones granting raw device, module, or admin access are nearly as powerful. A privileged or
   capability-broad container is a candidate escape regardless of the application.

3. **Find shared host namespaces.** Check whether the pod shares the host PID, network, or IPC namespace.
   Host PID exposes and can signal every process on the node; host network removes network isolation and
   exposes node-local services; host IPC shares memory segments. Each shared namespace is a direct bridge from
   the container to the node.

4. **Trace host mounts and devices.** Follow every host-path volume and mounted device into the container. A
   writable mount of a sensitive host directory, the container runtime socket, or a raw device lets the process
   read or write node state, and a mount of the host root or a runtime socket is a direct escape. Note which
   mounts are writable and what they expose.

5. **Check the defenses that keep a process inside.** Confirm whether the workload keeps the isolation
   defenses on: a non-root user, a read-only root filesystem where feasible, seccomp and mandatory-access-
   control profiles enforced rather than unconfined, and no-new-privileges set. A security context that
   disables these widens whatever the other settings expose.

6. **Confirm and record.** Confirm by showing a process in the container reaches a node resource it should
   not, reads host state through a mount, or acts through a shared namespace, on a non-production node and
   without pivoting further. Kill the lead if the workload is unprivileged, drops capabilities, shares no host
   namespace, mounts no sensitive host path, and keeps its isolation profiles enforced. Record the container
   source, the node resource sink, and the security-context setting that permitted the escape.

## Where escape surface leaks

- **Privileged mode removes isolation wholesale.** A privileged container has near-node power; the escape is
  the setting, not a separate bug.
- **A single capability can be enough.** Capabilities granting raw device, module load, or admin operations
  give kernel-level reach without full privileged mode.
- **A shared host namespace is a bridge.** Host PID, network, or IPC each removes one wall between the
  container and every process or service on the node.
- **A host mount exposes node state.** A writable host path, the runtime socket, or a device mount lets the
  process read or alter the node directly.
- **Disabled profiles widen everything.** Running as root, unconfined seccomp or mandatory-access-control, or
  a writable root filesystem amplifies whatever the other settings already expose.

## Worked example (a confirm and a kill)

> **Confirm.** A batch workload runs with a host-path volume mounting the node root filesystem read-write, and
> its security context runs as root with an unconfined seccomp profile. A process in the container writes to a
> node-level path outside its own filesystem and reads another pod's mounted secret from the host. **Confirmed**
> container escape to node via writable host mount, `critical`, remediation = remove the host-root mount, run
> the workload as a non-root user with a read-only root filesystem, enforce a restrictive seccomp profile, and
> drop all unneeded capabilities.
>
> **Kill.** The workload runs unprivileged as a non-root user, drops all capabilities and adds none, shares no
> host namespace, mounts only its own ephemeral volumes with no host path or device, and enforces seccomp and a
> mandatory-access-control profile with no-new-privileges set. A process inside cannot reach the node or another
> pod. **Killed**, `kill_reason` = "unprivileged non-root workload with capabilities dropped, no host namespace
> or host mount, and isolation profiles enforced; the container boundary holds and no node resource is
> reachable."

## Rationalizations to reject

- *"It needs privileged mode to work."* → Confirm exactly which capability or device it needs and grant only
  that; blanket privileged mode is an escape, not a requirement.
- *"It only mounts one host directory."* → A single writable host mount can expose node state or another pod's
  data; check what it grants, not how many there are.
- *"Host network is just for performance."* → Host network removes network isolation and exposes node-local
  services; treat it as an escape surface, not a tuning knob.
- *"The image is trusted."* → Escape surface is about the runtime configuration, not the image; a trusted image
  compromised at runtime still uses whatever the security context grants.
- *"There is nothing else on the node."* → Nodes are shared and rescheduled; a node-level foothold reaches
  whatever lands there next.

## Executing this in practice

You need each workload's security context: privileged flag, added and dropped capabilities, shared host
namespaces, host-path and device mounts with their read-write mode, the run-as user, and the seccomp and
mandatory-access-control profiles. For each workload, compare that to the intended isolation and find the
setting that lets a process reach the node. Reading the pod spec shows the granted surface; demonstrating a
node-resource reach on a non-production node shows whether the boundary holds.

## Related

- `auditing-kubernetes-workload-and-rbac-hardening` - the broader workload-hardening audit; this skill focuses
  specifically on the settings that permit a node escape.
- `auditing-host-mount-and-device-exposure` - the deep dive on the host-path and device half of the escape
  surface this skill inventories.
- `auditing-container-runtime-and-socket-exposure` - the specific and severe case of the container runtime
  socket mounted into a pod.
- `mapping-pod-to-cloud-credential-reach` - what a foothold on the node then reaches in the cloud; escape and
  credential reach compound.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the container process, sink = the node or host
  resource, evidence = the security-context setting that permitted the escape.
