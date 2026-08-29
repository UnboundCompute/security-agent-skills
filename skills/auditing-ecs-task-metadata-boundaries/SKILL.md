---
name: auditing-ecs-task-metadata-boundaries
description: >-
  Audit container task credential and metadata boundaries in orchestrated compute such as ECS: a workload
  that can reach the container credential endpoint or the host instance metadata service to obtain a role
  broader than the task needs, a task role over-scoped for the workload, a sidecar or co-located container
  sharing the same credentials, and a server-side request path inside the task that reaches the metadata
  endpoint. Covers the task credential relative URI, the instance metadata service reachable from a task,
  and the blast radius when one container in a task is compromised. Use when containerized workloads assume
  a task or instance role and the metadata endpoints are the boundary. The reachable metadata endpoint is
  the source, the credential it returns is the sink, and the role wider than the task's need is the bug.
license: MIT
---

# Auditing ECS task metadata boundaries: when a container reaches a credential it should not

Containerized compute hands each task a role by serving credentials from a metadata endpoint the container
can call. That design puts a powerful boundary inside the task: whatever the workload can reach at the
credential endpoint, it can use. Two endpoints matter. The task credential endpoint returns the task role,
which is fine if that role is scoped to exactly what the workload does. The host instance metadata service,
if reachable from the task, returns the node's instance role, which is usually far broader and was never
meant for the workload. A server-side request flaw inside the task, or a compromised sidecar, turns either
endpoint into a credential-theft primitive. You audit these by checking which endpoints the workload can
reach and how tightly the returned role is scoped.

## When to use

- Containerized workloads assume a task or instance role served from a metadata endpoint.
- A task can reach the host instance metadata service, not only its own task credential endpoint.
- A task runs multiple containers or sidecars that share task credentials, or has a server-side request path.

## Scope check

Test metadata and credential boundaries only against clusters and accounts you own or are authorized to
assess, on non-production infrastructure. A confirming request retrieves live credentials, so stay inside
the authorized account and treat any retrieved credential as sensitive within scope. If you can't name the
authorization, stop.

## The loop

1. **Establish the task's real need first.** Name what this workload actually must do in the cloud account:
   which specific actions on which resources. This is the false-positive killer: a task role scoped to exactly
   that need, reachable only by the workload that needs it, is correct. Name the need, then compare the
   reachable credentials to it.

2. **Determine which metadata endpoints the task can reach.** Confirm whether the workload can reach only its
   task credential endpoint or also the host instance metadata service. A task that can call the instance
   metadata service can obtain the node role, which typically exceeds any single task's need. Check the
   network path and any hop-limit or endpoint restriction that should block it.

3. **Scope the task role against the need.** Read the task role's permissions and compare them to the
   workload's actual actions. A task role granting broad data access, further role assumption, or
   administrative actions the workload never performs is over-scoped, and its whole surface is available to
   anyone who reaches the credential endpoint inside the task.

4. **Check the co-located blast radius.** Containers in the same task, including sidecars, generally share
   the task credentials. If any co-located container is less trusted, third-party, a log shipper, a
   general-purpose sidecar, then compromising it yields the full task role. Determine whether untrusted
   co-located containers share credentials with a powerful task role.

5. **Trace server-side request paths to the endpoints.** A request-forgery flaw in the workload that lets an
   attacker make the task fetch an arbitrary URL can target the metadata endpoints from inside the task,
   retrieving credentials. Confirm whether such a path exists and whether the endpoints are reachable through
   it, since that converts an application bug into cloud credential theft.

6. **Confirm and record.** Confirm by retrieving credentials from a reachable endpoint within scope and
   showing they carry more than the task needs, or by reaching the instance metadata service from the task,
   without using the credentials beyond proof. Kill the lead if the task can reach only its own credential
   endpoint, the task role is scoped to the workload's exact need, no untrusted container shares the
   credentials, and no server-side request path reaches the metadata service. Record the reachable endpoint,
   the returned credential, and the excess over need.

## Where task metadata boundaries leak

- **The instance metadata service is the wrong credential for a task.** If a task can reach it, the workload
  obtains the node role, which almost always exceeds the task's need.
- **An over-scoped task role is fully available inside the task.** Every permission the role holds is reachable
  by any code or container that can call the credential endpoint.
- **Sidecars share the task role.** A compromised or third-party co-located container yields the whole task
  credential, so the least-trusted container sets the blast radius.
- **Server-side request forgery reaches the metadata endpoint.** A fetch-arbitrary-URL flaw in the workload
  becomes cloud credential theft when the endpoints are reachable.
- **A powerful role plus a broad reach is the compounding failure.** Neither alone is as dangerous as a broad
  role reachable through a weak container or a request path.

## Worked example (a confirm and a kill)

> **Confirm.** A task can reach the host instance metadata service, and the node role grants broad access
> across the account. A server-side request path in the workload lets an attacker make the task fetch the
> metadata endpoint, returning the node role's credentials. **Confirmed** metadata boundary breach to node
> role theft, `high`, remediation = block the task's route to the instance metadata service (restrict the
> hop limit or network path), scope the workload to a task role matching its need, and fix the request path
> so it cannot target internal endpoints.
>
> **Kill.** The task can reach only its own credential endpoint, the task role is scoped to the two actions
> the workload performs on its own resources, co-located containers are all first-party and the sidecar has no
> untrusted input, and the workload cannot fetch arbitrary URLs. A request aimed at the instance metadata
> service from the task is unroutable. **Killed**, `kill_reason` = "task reaches only its own least-privilege
> credential endpoint, no untrusted co-located container, and no request path to the instance metadata
> service; no credential beyond the task's need is reachable."

## Rationalizations to reject

- *"The task role is what the workload uses."* → Confirm it matches the workload's exact actions; a role with
  extra permissions exposes all of them to anyone reaching the endpoint.
- *"Only our code runs in the task."* → Sidecars and co-located containers share the credentials; the
  least-trusted one sets the blast radius.
- *"The metadata service is local."* → Local reachability from the task is exactly the problem; a request-
  forgery flaw or a compromised container reaches it.
- *"We do not have SSRF."* → Confirm it; and note a compromised container reaches the endpoints without any
  application request flaw.
- *"The node role is fine, it is our infrastructure."* → The node role is not the task's credential; a task
  reaching it obtains far more than its workload should hold.

## Executing this in practice

You need the task's real need, which metadata endpoints the task can reach, the task and instance role
scopes, the co-located containers and whether they share credentials, and any server-side request path. For
each task, compare the reachable credentials to the need and confirm the instance metadata service is
unreachable. Reading the role and the network path shows the intended boundary; retrieving a credential from a
reachable endpoint shows whether it holds.

## Related

- `exploiting-ssrf-to-cloud-metadata` - the request-forgery-to-metadata path in detail; this skill scopes the
  credentials that path would steal inside a task.
- `hunting-non-human-identity-and-secret-reachability` - the task and instance roles are machine identities;
  that skill governs what each reachable credential can then access.
- `auditing-kubernetes-workload-and-rbac-hardening` - the Kubernetes analog, where the projected service-
  account token and node identity form the same boundary.
- `adjudicating-taint-paths` - use it to confirm a server-side request path in the workload reaches a metadata
  endpoint.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the reachable metadata endpoint, sink = the
  credential it returns, evidence = the role wider than the task's need.
