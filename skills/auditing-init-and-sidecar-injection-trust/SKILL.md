---
name: auditing-init-and-sidecar-injection-trust
description: >-
  Audit the init and sidecar containers a workload runs, including ones injected by a mutating admission
  webhook, for trust the main container never granted: an injected sidecar that runs with broader privileges,
  host access, or credentials than the workload, an init container that fetches and executes remote content
  before the app starts, a shared volume or process namespace that lets a sidecar read the main container's
  secrets, and an injection whose image and configuration come from a source the workload owner does not
  control. Covers Kubernetes pods where init and sidecar containers, declared or webhook-injected, share the
  pod with the application. Use when pods run init or sidecar containers, especially injected ones. The
  injected or auxiliary container is the source, the pod resource or credential it reaches is the sink, and
  the trust it holds beyond the main container is the bug.
license: MIT
---

# Auditing init and sidecar injection trust: the containers you did not write share your pod

A pod is not just its application container. Init containers run first, sidecars run alongside, and many are
injected automatically by a mutating admission webhook that the workload owner never sees in their manifest.
All of them share the pod: the same network namespace, often shared volumes, sometimes the process namespace,
and they can be granted their own security context and credentials. That makes every init and sidecar
container a trust question the main container did not ask. An injected sidecar may run privileged or with host
access the app never needed; an init container may fetch and execute remote content before the app starts; a
shared secret volume or process namespace lets a sidecar read the application's secrets; and the injected
image and its configuration may come from a source outside the workload owner's control. The pod looks like
one workload but is a collection of containers with different privileges and origins. You audit this by
enumerating every init and sidecar container, declared and injected, and checking what trust each holds.

## When to use

- Pods run init or sidecar containers, especially ones injected by a mutating admission webhook.
- An injected or auxiliary container may run with broader privilege, host access, or credentials than the app.
- Init containers fetch or execute content, or sidecars share volumes or namespaces with the main container.

## Scope check

Audit init and sidecar trust only on clusters and workloads you own or are authorized to assess, on
non-production pods. Confirming a sidecar can read the app's secrets or that an init container executes remote
content touches real workloads, so use non-production pods and do not exfiltrate anything you surface. If you
can't name the authorization, stop.

## The loop

1. **Establish the workload's intended trust first.** Name what the application container is supposed to run
   with: its privilege level, its credentials, its host access. This is the false-positive killer: a pod whose
   init and sidecar containers run at or below the app's privilege, from controlled images, sharing nothing
   sensitive, is fine. Name the intended trust, then check each auxiliary container against it.

2. **Enumerate every init and sidecar container, including injected ones.** List the containers actually in the
   pod at runtime, not just those in the authored manifest, since a mutating webhook may add sidecars. For each,
   note its image source, security context, credentials, and mounts. An injected container the owner did not
   declare is the one most likely to hold unexpected trust.

3. **Compare each auxiliary container's privilege to the app.** For each init and sidecar container, check
   whether it runs privileged, adds capabilities, shares a host namespace, or mounts host paths that the main
   container does not. A sidecar with broader privilege than the app raises the pod's whole blast radius to that
   sidecar's level.

4. **Check init containers that fetch or execute content.** For each init container, determine whether it
   downloads and runs remote content, applies configuration from an external source, or executes a script
   before the app starts. An init container that pulls and executes untrusted content runs attacker-influencable
   code in the pod before the application even begins.

5. **Check shared volumes and namespaces for secret reach.** Determine which volumes and namespaces the
   auxiliary containers share with the main container. A sidecar that mounts the app's secret volume or shares
   its process namespace can read the application's secrets and memory. Confirm sensitive mounts and namespaces
   are not shared with a less-trusted sidecar.

6. **Confirm and record.** Confirm by showing an init or sidecar container holds trust the app did not grant: a
   sidecar reading the app's secret, an injected container running privileged, or an init container executing
   remote content, on non-production pods and without exfiltrating. Kill the lead if every init and sidecar
   container, declared and injected, runs at or below the app's privilege from a controlled image, shares no
   sensitive volume or namespace, and executes no untrusted content. Record the auxiliary container source, the
   pod resource or credential sink, and the trust it held beyond the main container.

## Where init and sidecar trust leaks

- **Injected sidecars are invisible in the manifest.** A mutating webhook adds containers the owner never
  declared, so the authored spec understates the pod's trust.
- **A sidecar can out-privilege the app.** A sidecar running privileged or with host access raises the pod's
  blast radius above the application's own security context.
- **Init containers can fetch and execute.** An init container that pulls and runs remote content executes
  attacker-influencable code before the app starts.
- **Shared volumes and namespaces expose secrets.** A sidecar that mounts the app's secret volume or shares its
  process namespace reads the application's secrets and memory.
- **Injected images come from elsewhere.** The injected image and its configuration may originate from a source
  the workload owner does not control or verify.

## Worked example (a confirm and a kill)

> **Confirm.** A mutating webhook injects a sidecar into every pod in a namespace. The injected sidecar runs
> privileged and mounts the application's secret volume, and its image comes from a registry the workload owner
> does not control. From the sidecar, a process reads the application's database credential the app never shared
> with it. **Confirmed** injected-sidecar trust beyond the workload, `high`, remediation = scope the injected
> sidecar to the minimum privilege it needs, stop it from mounting the app's secret volume, and pin and verify
> the injected image from a controlled source.
>
> **Kill.** Every init and sidecar container in the pod, including the webhook-injected ones, runs unprivileged
> at or below the app's security context from pinned, verified images the owner controls, none mounts the
> application's secret volume or shares its process namespace, and no init container fetches or executes remote
> content. No auxiliary container holds trust the app did not grant. **Killed**, `kill_reason` = "all init and
> sidecar containers run at or below app privilege from controlled images with no shared secret volume or
> namespace and no remote execution; no injected container exceeds the workload's trust."

## Rationalizations to reject

- *"The manifest only has one container."* → A mutating webhook may inject more at runtime; enumerate the
  containers actually in the pod, not just the authored spec.
- *"The sidecar is from the platform team."* → Confirm its privilege, image source, and mounts; a trusted team's
  sidecar can still out-privilege the app or read its secrets.
- *"Init containers just set things up."* → If an init container fetches and runs remote content, it executes
  code in the pod before the app; check what it pulls and runs.
- *"They are in the same pod, sharing is fine."* → Sharing a secret volume or process namespace lets a
  less-trusted sidecar read the app's secrets; scope what is shared.
- *"The injected image is standard."* → Confirm it is pinned and from a source you verify; an injected image
  from elsewhere runs in your pod with whatever trust it is granted.

## Executing this in practice

You need the full runtime list of init and sidecar containers including injected ones, each container's image
source, security context, credentials, and mounts, whether any init container fetches or executes content, and
the shared volumes and namespaces. For each auxiliary container, compare its trust to the application's. Reading
the runtime pod spec and the mutating webhook configuration shows the granted trust; showing a sidecar reach the
app's secret or an init container execute remote content shows whether the boundary holds.

## Related

- `hunting-container-escape-surface` - the privilege and host-access settings this skill checks on auxiliary
  containers are the same escape surface, viewed per injected container.
- `auditing-admission-control-policy-gaps` - the mutating webhook that injects sidecars is admission machinery;
  a gap there is how an untrusted injection gets in.
- `auditing-workload-secret-exposure-surface` - the shared-volume secret reach a sidecar exploits is that
  skill's subject, here caused by an injected neighbor.
- `auditing-container-image-provenance` - the injected sidecar's image provenance question; an unverified
  injected image runs in your pod.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the injected or auxiliary container, sink = the pod
  resource or credential it reaches, evidence = the trust it held beyond the main container.
