---
name: auditing-kubernetes-workload-and-rbac-hardening
description: >-
  Audit Kubernetes manifests for a subject granted more than it needs or a workload that can escape its
  container, after the binding graph and admission policy are resolved. Covers a RoleBinding or
  ClusterRoleBinding to cluster-admin or a wildcard-verb role, a pod running privileged or with host
  namespaces or a sensitive hostPath mount, a container running as root or able to escalate privilege,
  dangerous added capabilities, a service-account token mounted where the workload does not need the API,
  and a workload left flat with no network policy. Use when reviewing the Kubernetes YAML plane (roles,
  bindings, and workload security contexts as declared), not the cloud identity graph or the image build.
  The manifest is the source, a cluster-admin subject or an escaping workload is the sink, and a grant or
  a privilege the binding graph and admission actually allow is the bug.
license: MIT
---

# Auditing Kubernetes workload and RBAC hardening: what the binding graph and admission actually allow

A wildcard ClusterRole that nothing binds grants nobody anything, and a privileged pod that admission
rejects never deploys. That is why this audit is not a scan of role verbs and pod fields; it is a
resolution of the binding graph (which subject is actually granted a role) and the admission layer
(whether the workload can deploy as written). The competitors flag a wildcard role or a root container in
isolation; the finding that survives is the one where a binding grants the power to a workload that can
use it, and admission does not stop it. You audit the manifests by resolving those two layers and asking,
per subject and per workload, whether it holds more than it needs and can act on it. Stay on the
Kubernetes YAML plane, not the cloud IAM graph and not the image build.

## When to use

- You have Kubernetes manifests: roles and bindings, pod and workload specs, service accounts, network policies.
- A binding references cluster-admin or a wildcard role, or a pod sets a privileged or host-level security context.
- You want to know which grants and privileges the binding graph and admission actually allow, not which appear.

## Scope check

Audit only manifests for clusters you own or are authorized to assess, and never apply a manifest or
exercise a binding against a live cluster to test a finding, adjudicate on the resolved graph. If you
can't name the authorization, stop.

## The loop

1. **Resolve the binding graph and admission first.** For each Role and ClusterRole, find the bindings
   that reference it and the subjects they grant it to (a named service account, a broad group, an
   authenticated-users binding), and determine what pod-security admission level or admission policy the
   target namespace enforces. A role is only a grant through a binding, and a workload is only deployable
   if admission allows it; settle both before flagging.

2. **Check over-granted bindings.** Look for a RoleBinding or ClusterRoleBinding to cluster-admin, or to a
   role with wildcard verbs, resources, or API groups, and note how broad the bound subject is. A
   wildcard role bound to a default service account that pods mount is far worse than one bound to nothing.

3. **Check escape-primitive workloads.** Look for a pod set privileged, sharing the host process, IPC, or
   network namespace, or mounting a sensitive host path (the root filesystem, the container runtime
   socket, host config, the kubelet directory). Each is a container-escape primitive if admission permits it.

4. **Check the container security context.** Look for run-as-non-root absent or false, run-as-user zero,
   privilege escalation not disabled, a writable root filesystem where read-only is warranted, and
   dangerous added capabilities or a failure to drop all capabilities. This is the deploy-time context;
   where it overlaps the image default, this skill owns the security context and hands the image to the
   container-image skill.

5. **Check token exposure and network reach.** Look for a service-account token automounted on a pod or
   account that does not call the API, a secret supplied as a literal environment value rather than a
   reference, and a workload selected by no network policy in a cluster without a default-deny. Also flag
   RBAC verbs that are escalation primitives themselves (escalate, bind, impersonate, create pods, read
   secrets) when granted where they enable lateral gain.

6. **Confirm and record.** Confirm by tracing a binding to a subject a reachable workload uses, or a
   security context admission would admit. Kill the lead if a wildcard role has no binding or binds only
   an unused subject, if pod-security admission or an admission policy would reject the privileged or
   hostPath pod as written, if a root context is overridden by a namespace or cluster default or the image
   already drops to non-root, if an automounted token belongs to an account bound to no meaningful RBAC,
   if a missing network policy is covered by a namespace-wide default-deny or the CNI, or if a secret in
   the environment is a reference rather than a plaintext value. Record the subject or workload, the
   resolved grant or privilege, and the admission verdict.

## Where Kubernetes manifests leak

- **The role is not the grant; the binding is.** A wildcard ClusterRole that nothing references grants
  nobody; resolve the binding graph before calling a role a finding.
- **Admission can make a manifest undeployable.** A privileged or hostPath pod that pod-security admission
  rejects never runs as written; the admission level is part of the finding.
- **A root context can be overridden downstream.** A namespace default security context or a non-root
  image user can override a root pod field; check the effective context, not the field alone.
- **A token grants only what its account is bound to.** An automounted service-account token on an account
  with no meaningful RBAC is not an exposure; the binding decides.
- **Some RBAC verbs are escalation primitives.** Escalate, bind, impersonate, create-pods, and read-secrets
  let a subject climb; a narrow-looking role holding one of these can still reach cluster-admin.

## Worked example (a confirm and a kill)

> **Confirm.** A ClusterRoleBinding binds the default service account of a namespace to cluster-admin, and
> pods in that namespace automount that account's token; a workload there is reachable and admission does
> not strip the token. A compromise of that pod is cluster-admin. **Confirmed** cluster-admin granted to a
> mounted default service account, `critical`, remediation = remove the binding, give the workload a
> dedicated least-privilege account, and disable token automount where the API is not used.
>
> **Kill.** A pod spec sets privileged true and mounts a host path, but the target namespace enforces the
> restricted pod-security standard, which rejects both, so the manifest cannot deploy as written.
> **Killed**, `kill_reason` = "the privileged, hostPath pod is rejected by the namespace's restricted
> pod-security admission, so it cannot be admitted."

## Rationalizations to reject

- *"There is a wildcard ClusterRole."* -> Does any binding grant it, and to whom? An unbound wildcard role
  grants nobody; resolve the binding graph first.
- *"The pod runs privileged."* -> Would admission admit it? A restricted or baseline pod-security policy
  rejects it, and the manifest never deploys as written.
- *"The container runs as root."* -> Is the context overridden by a namespace default or the image's
  non-root user? Check the effective security context before flagging.
- *"The service-account token is automounted."* -> Is the account bound to anything meaningful? A mounted
  token that grants nothing is not an exposure.
- *"There is no network policy."* -> Is there a namespace default-deny or a CNI that isolates it? A missing
  policy is only flat if nothing else segments the workload.

## Executing this in practice

You need the roles and bindings, the workload specs and their security contexts, the service accounts and
their token settings, the network policies, and the pod-security admission level or admission policy per
namespace. For each subject, resolve the binding graph; for each workload, resolve the admission verdict;
then decide whether the grant or privilege is more than needed and usable. Reading the manifest tells you
what it declares; resolving the bindings and admission tells you what the cluster would actually allow.

## Related

- `auditing-infrastructure-as-code-exposures` - the sibling for cloud provider resources; this skill owns
  the Kubernetes YAML plane, that one owns buckets, network rules, and identity policies.
- `auditing-container-image-build-hardening` - the mutual FP-killer: a root image is killed by a
  locked-down security context here, and a permissive context here is aggravated by a root image there.
- `hunting-iam-privilege-escalation-paths` - the cloud identity graph that sits beneath the cluster;
  distinct from the in-cluster RBAC binding graph this skill resolves.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the manifest, sink = a cluster-admin subject or an
  escaping workload, evidence = the resolved binding graph and the admission verdict.
