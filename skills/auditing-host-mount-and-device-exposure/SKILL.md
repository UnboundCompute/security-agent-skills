---
name: auditing-host-mount-and-device-exposure
description: >-
  Audit the host paths and devices a workload mounts for reach across the container boundary onto the node: a
  writable hostPath into a sensitive node directory, a mount of the host root or a system path that exposes
  other pods' data and node configuration, a raw device or block volume that grants low-level host access, and
  a hostPath whose subpath or symlink handling lets a workload escape the intended directory. Covers
  Kubernetes and container hosts where hostPath volumes and device mounts connect a container to the node
  filesystem and hardware. Use when workloads mount host paths or devices and the question is what node state
  that mount exposes. The workload holding the host mount is the source, the node file or device it reaches is
  the sink, and the host-path or device exposure beyond the workload's need is the bug.
license: MIT
---

# Auditing host mount and device exposure: a path into the node filesystem

A hostPath volume or a device mount is a hole cut in the container boundary: it connects the workload directly
to the node's filesystem or hardware. Sometimes that is genuinely needed, a node agent reading a specific
system directory, but the exposure is defined by what the mount reaches, not by why it was added. A writable
hostPath into a sensitive node directory lets the workload alter node state; a mount of the host root or a
broad system path exposes other pods' data, node configuration, and credentials; a raw device or block volume
grants low-level access that bypasses filesystem controls; and a hostPath with loose subpath or symlink
handling can be walked out of its intended directory into the rest of the node. The audit treats each host
mount as node reach and asks how far it goes. You audit this by inventorying every host-path and device mount
and determining exactly what node state each exposes and whether it is writable.

## When to use

- Workloads mount hostPath volumes or devices that connect the container to the node filesystem or hardware.
- A mount may reach a sensitive node directory, the host root, or a raw device broader than the workload needs.
- A hostPath's subpath or symlink handling may let a workload escape the intended directory into the node.

## Scope check

Audit host mounts only on clusters and nodes you own or are authorized to assess, on non-production nodes.
Confirming a mount's reach reads or writes real node state and can affect other pods, so use a non-production
node and do not alter node state or read other pods' real data. If you can't name the authorization, stop.

## The loop

1. **Establish the workload's legitimate host need first.** Name exactly which node path or device each
   workload genuinely needs and whether it needs to write. This is the false-positive killer: a workload with
   no host mount, or one with a narrow read-only mount of exactly the directory it needs, is correctly bounded.
   Name the legitimate need, then compare each mount to it.

2. **Inventory every host-path and device mount.** List all hostPath volumes and device or block-volume mounts
   across workloads, with the exact host path or device, the mount point, and the read-write mode. Each is a
   connection to the node; the ones that are writable or reach broad paths are the candidates.

3. **Determine what each mount reaches.** For each host-path mount, establish what node state the path exposes:
   a narrow application directory, a sensitive system directory, other pods' data directories, node
   configuration and credentials, or the host root. The breadth of the path sets the exposure, and a mount of
   the root or a system path exposes far more than the workload's own data.

4. **Check write access and device breadth.** For each mount, confirm whether it is writable, since a writable
   mount of a sensitive path lets the workload alter node or other-pod state, and check whether any device
   mount grants raw or block-level access that bypasses filesystem permissions. Writable and raw-device mounts
   are the higher-severity cases.

5. **Check subpath and symlink escape.** For hostPath mounts, determine whether subpath and symlink handling
   confine the workload to the intended directory or whether a symlink or a crafted path lets it traverse into
   the rest of the node filesystem. A mount meant to expose one directory that can be walked out of exposes the
   whole node.

6. **Confirm and record.** Confirm by reading node state or another pod's data through a mount, or writing to a
   node path a workload should not, or escaping a hostPath's intended directory, on a non-production node and
   without altering real node state or reading real pod data. Kill the lead if every host mount is a narrow,
   read-only mount of exactly the path the workload needs, with no raw-device access and no subpath or symlink
   escape. Record the workload source, the node file or device sink, and the host-path or device exposure
   beyond the workload's need.

## Where host mount exposure leaks

- **A broad host path exposes more than the workload's data.** A mount of the host root or a system directory
  reaches other pods' data, node configuration, and credentials.
- **A writable mount alters node state.** A writable hostPath into a sensitive directory lets the workload
  change node or other-pod state, not just read it.
- **A raw device bypasses filesystem controls.** A device or block-volume mount grants low-level access beneath
  the file-permission layer.
- **Subpath and symlink handling can escape the directory.** A hostPath meant to expose one directory can be
  walked out of into the rest of the node if traversal is not confined.
- **The need is narrow but the mount is broad.** A workload that needs one file is often given a whole
  directory or the host root, exposing far beyond the requirement.

## Worked example (a confirm and a kill)

> **Confirm.** A monitoring workload mounts a broad host system directory read-write to read one metrics file.
> Through the writable mount, a process reads node configuration and another pod's data directory under the same
> path, and can write to a node file outside its own scope. **Confirmed** host-mount exposure of node and
> cross-pod state, `high`, remediation = replace the broad mount with a read-only mount of exactly the one
> metrics file or directory the workload needs, remove write access, and confine the mount so it cannot traverse
> outside the intended path.
>
> **Kill.** The workload mounts only the single directory it needs, read-only, with no access to the host root,
> system directories, other pods' data, or node credentials, no raw-device mount, and subpath handling confines
> it so no symlink or crafted path escapes the directory. The mount exposes exactly the workload's own need and
> nothing more. **Killed**, `kill_reason` = "narrow read-only mount of exactly the needed path with no broad
> host access, no raw device, and no subpath or symlink escape; the mount reaches nothing beyond the workload's
> need."

## Rationalizations to reject

- *"It only mounts one directory."* → Check what that directory contains; a system directory or a parent of
  other pods' data exposes far more than the workload's own files.
- *"It needs to read a node file."* → Mount that one file or directory read-only; a broad or writable mount for
  a narrow read is exposure, not a requirement.
- *"The mount is read-only."* → Read-only still exposes node configuration, credentials, and other pods' data if
  the path is broad; scope the path, not just the mode.
- *"It is just a device for hardware access."* → A raw device mount bypasses filesystem permissions and can
  grant low-level host access; confirm it is the specific device needed and nothing more.
- *"The mount is confined to its directory."* → Confirm subpath and symlink handling actually prevent traversal;
  a mount that can be walked out of exposes the whole node.

## Executing this in practice

You need every host-path and device mount across workloads with the exact path or device, mount point, and
read-write mode, what node state each path exposes, whether any device grants raw access, and whether subpath
and symlink handling confine each hostPath. For each mount, compare its reach to the workload's legitimate need.
Reading the pod specs shows the granted mounts; reading node or cross-pod state through a mount on a
non-production node shows the exposure it actually confers.

## Related

- `hunting-container-escape-surface` - the broader escape audit; host mounts and devices are the filesystem and
  hardware half of that surface, examined in depth here.
- `auditing-container-runtime-and-socket-exposure` - the most severe single host mount, the runtime socket,
  whose blast radius is the whole node.
- `auditing-workload-secret-exposure-surface` - a broad host mount can reach other pods' mounted secrets;
  the two exposures meet at the node filesystem.
- `auditing-kubernetes-workload-and-rbac-hardening` - the workload-hardening context in which host-path and
  device mounts should be constrained or forbidden.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the workload holding the host mount, sink = the node
  file or device it reaches, evidence = the host-path or device exposure beyond the workload's need.
