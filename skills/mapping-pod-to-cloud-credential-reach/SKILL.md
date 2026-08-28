---
name: mapping-pod-to-cloud-credential-reach
description: >-
  Map what cloud identity a compromised pod can reach and what that identity can then do: a pod bound to a
  workload identity or role far broader than it needs, a node instance role reachable from any pod through
  the node metadata endpoint, a service-account token mounted into a pod that federates to cloud, and the
  chain from one pod's credential to another cloud resource or identity. Covers Kubernetes on cloud where
  pods obtain cloud credentials through workload identity federation, mounted tokens, or the node role, and
  where a pod compromise becomes cloud access. Use when pods hold or can reach cloud credentials and the
  blast radius of a pod compromise is the question. The compromised pod is the source, the cloud resource its
  credential reaches is the sink, and the over-broad or reachable credential is the bug.
license: MIT
---

# Mapping pod-to-cloud credential reach: the blast radius of one compromised pod

A pod on a cloud-hosted cluster is rarely just a container; it is a holder of cloud credentials. It may bind
to a workload identity, mount a service-account token that federates to a cloud role, or simply sit on a node
whose instance role is reachable through the node metadata endpoint. So the real question about a pod is not
what it does but what its credentials reach: if this pod is compromised, which cloud APIs, data stores, and
other identities does its token open, and can it chain from there to something larger. The dangerous pattern
is a modest workload holding a broad credential, or any pod able to reach the node role, because the blast
radius is the credential, not the application. You map this by identifying every credential a pod holds or can
reach and enumerating what each can do.

## When to use

- Pods obtain cloud credentials through workload identity federation, mounted tokens, or the node role.
- The node metadata endpoint may be reachable from pods, exposing the node instance role.
- You need the blast radius of a pod compromise: which cloud resources and identities its credentials reach.

## Scope check

Map credential reach only in cloud accounts and clusters you own or are authorized to assess, on
non-production identities. Enumerating what a credential can do exercises real cloud APIs, so use read-only
enumeration where possible and never use a reached credential to alter or exfiltrate. If you can't name the
authorization, stop.

## The loop

1. **Establish each pod's intended cloud access first.** Name what cloud resources each workload legitimately
   needs: a specific bucket, a specific queue, nothing. This is the false-positive killer: a pod bound to a
   narrowly scoped identity that reaches only what it needs, with the node role unreachable, is correct. Name
   the intended reach, then enumerate the actual reach.

2. **Identify every credential the pod holds.** List the cloud identity the pod binds to through workload
   identity federation and any service-account token or key mounted into it. Each is a credential a compromised
   process inside the pod can use. Note which identity each maps to in the cloud.

3. **Check reachability of the node role.** Determine whether the node metadata endpoint is reachable from
   pods. If it is, any pod can retrieve the node instance role's credentials regardless of its own binding, so
   the node role is effectively every pod's credential. Confirm whether metadata access is blocked from
   workloads or exposed.

4. **Enumerate what each reached identity can do.** For every credential a pod holds or can reach, enumerate
   the cloud permissions it grants: which resources it reads and writes, which APIs it calls, and whether it
   can assume or impersonate another identity. This is the blast radius, and it is set by the identity's
   policy, not the pod's role in the app.

5. **Chain to further reach.** Follow whether a reached identity can pivot: assume-role or impersonation edges,
   permission to read other secrets or credentials, or write access that grants control of another workload.
   One pod's modest-looking credential can be a first hop to a much larger reach; trace the chain, not just the
   first identity.

6. **Confirm and record.** Confirm by using a pod's credential (or the node role reached from a pod) to
   enumerate a cloud resource it should not reach, in scope and read-only, without altering or exfiltrating.
   Kill the lead if each pod binds only to a narrowly scoped identity reaching what it needs, the node metadata
   endpoint is unreachable from workloads, and no reached identity chains to a broader one. Record the pod
   source, the cloud-resource sink, and the over-broad or reachable credential.

## Where credential reach leaks

- **A modest pod can hold a broad credential.** Blast radius is set by the bound identity's policy, not the
  workload's apparent importance.
- **The node role is every pod's credential.** If the node metadata endpoint is reachable, any pod retrieves
  the node instance role, bypassing its own narrow binding.
- **A mounted token federates outward.** A service-account token that maps to a cloud role gives a pod
  compromise a cloud identity directly.
- **One credential chains to another.** Assume-role, impersonation, or secret-read permissions turn a first
  credential into a path to a larger one.
- **Reach is invisible from inside the app.** The pod's code shows what it uses; only the identity's policy
  shows what a compromise could use.

## Worked example (a confirm and a kill)

> **Confirm.** A logging sidecar pod binds to a workload identity intended only to write to one log bucket, but
> the node metadata endpoint is reachable from pods, and the node instance role can read a secrets store and
> assume an administrative role. A process in the pod retrieves the node role from metadata and enumerates the
> secrets store it was never meant to touch. **Confirmed** pod-to-cloud credential reach via the node role,
> `critical`, remediation = block pod access to the node metadata endpoint, scope the node role to the minimum
> the kubelet needs, and bind each workload to a narrowly scoped identity with no assume-role chain.
>
> **Kill.** Each pod binds to a workload identity scoped to exactly the one resource it needs, the node metadata
> endpoint is unreachable from workloads so the node role cannot be retrieved from a pod, and no bound identity
> can assume or impersonate another or read additional secrets. A compromised pod reaches only its one intended
> resource. **Killed**, `kill_reason` = "each pod bound to a least-privilege identity with the node metadata
> endpoint blocked and no credential chain; the blast radius is the one resource the workload needs."

## Rationalizations to reject

- *"The pod only writes logs."* → Blast radius is the credential's policy, not the app's function; enumerate
  what the bound identity can do, not what the code does.
- *"Each pod has its own scoped identity."* → Confirm the node metadata endpoint is unreachable; if it is not,
  every pod can retrieve the broad node role regardless.
- *"The credential is short-lived."* → Short-lived still reaches whatever it is scoped to for its lifetime; the
  reach, not the duration, sets the blast radius.
- *"It cannot reach anything sensitive."* → Enumerate the reach and the chain; an identity that can assume
  another or read secrets reaches far beyond its first grant.
- *"That role is for the node, not the pod."* → If pods can reach node metadata, the node role is a pod
  credential; the intended boundary is only real if it is enforced.

## Executing this in practice

You need each pod's bound cloud identity and mounted tokens, whether the node metadata endpoint is reachable
from workloads, the permissions of every reachable identity, and the assume-role or impersonation chains from
each. For each pod, enumerate the full reach and the chain, then compare it to the intended access. Reading the
bindings and identity policies shows the granted reach; enumerating a cloud resource read-only with a reached
credential shows whether the boundary holds.

## Related

- `hunting-container-escape-surface` - a node-level escape and node-role reach compound; escape gets to the
  node, this maps what the node's credentials then open.
- `mapping-service-account-impersonation-chains` - the cloud-side impersonation graph a reached identity
  chains through; the two maps meet at the assume-role edge.
- `hunting-non-human-identity-and-secret-reachability` - the general machine-credential reachability discipline
  this applies specifically to pods.
- `auditing-ecs-task-metadata-boundaries` - the same metadata-endpoint and task-versus-instance-role question
  for container tasks outside Kubernetes.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the compromised pod, sink = the cloud resource its
  credential reaches, evidence = the over-broad or reachable credential.
