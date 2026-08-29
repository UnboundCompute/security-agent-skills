---
name: auditing-namespace-as-tenant-boundary
description: >-
  Audit a Kubernetes namespace that is treated as a tenant isolation boundary for the isolation it does not
  actually provide: cluster-scoped resources and nodes shared across namespaces, RBAC that grants a tenant
  reach beyond its own namespace, missing network policy so pods cross namespaces freely, and shared cluster
  services (DNS, ingress, admission, storage classes) that see or serve every tenant. Covers multi-tenant
  clusters where each tenant is given a namespace and the namespace is assumed to contain them. Use when a
  namespace is the unit of tenant separation and the assumption is that a tenant cannot affect or observe
  another. The tenant confined to a namespace is the source, the cross-tenant resource or namespace it
  reaches is the sink, and the isolation the namespace does not enforce is the bug.
license: MIT
---

# Auditing the namespace as a tenant boundary: what a namespace does not isolate

A namespace is a name-scoping and policy-attachment unit, not a security sandbox, yet multi-tenant clusters
routinely hand each tenant a namespace and assume it contains them. It does not, on its own. Nodes are shared,
so a tenant that can influence scheduling or escape a container reaches another tenant's pods on the same node.
Cluster-scoped resources, custom resource definitions, node objects, persistent volumes, and cluster roles
sit outside every namespace and are visible or reachable across them. RBAC that grants a verb at cluster scope,
or on another namespace, punches straight through the boundary. Without network policy, pods cross namespaces
freely. And the shared cluster services every tenant depends on, DNS, ingress, admission, storage classes, see
or serve all tenants at once. The audit question is not whether tenants have separate namespaces but whether
the namespace actually stops one from affecting or observing another. You audit this by testing each isolation
the boundary is assumed to provide.

## When to use

- A namespace is the unit of tenant separation and is assumed to isolate one tenant from another.
- Tenants share nodes, cluster-scoped resources, and cluster services (DNS, ingress, admission, storage).
- RBAC, network policy, or resource quotas may not fully confine a tenant to its own namespace.

## Scope check

Audit tenant isolation only on clusters you own or are authorized to assess, on non-production tenants. Testing
cross-tenant reach touches other namespaces' resources, so use non-production tenants and do not read or alter
another tenant's real data. If you can't name the authorization, stop.

## The loop

1. **Establish what the boundary must isolate first.** Name what one tenant must not be able to do to another:
   read its resources, reach its pods, exhaust shared capacity, observe its traffic. This is the false-positive
   killer: a namespace backed by tenant-scoped RBAC, a default-deny network policy, resource quotas, and no
   cross-namespace or cluster-scoped grants is enforcing the boundary. Name the required isolations, then test
   each.

2. **Audit RBAC reach beyond the namespace.** For each tenant's identities, enumerate whether any binding
   grants a verb at cluster scope or on another namespace: cluster roles, cross-namespace role bindings, or
   access to cluster-scoped resources. A single cluster-scoped grant lets a tenant read or act outside its
   namespace, breaking the boundary at the API layer.

3. **Check network isolation between namespaces.** Determine whether pods in one tenant namespace can reach pods
   and services in another. Without a default-deny network policy, namespaces do not isolate traffic, so one
   tenant reaches another's services directly. Confirm segmentation exists between tenant namespaces, not just
   within one.

4. **Find shared-resource and node exposure.** Identify the cluster-scoped resources and nodes tenants share:
   node objects, persistent volumes and storage classes, custom resource definitions, and node co-tenancy. A
   tenant that can view or influence these, or that shares a node with another tenant's pods, reaches across the
   boundary through the shared layer rather than the API.

5. **Check shared cluster services and quotas.** Confirm the shared services every tenant uses, DNS, ingress,
   admission webhooks, do not leak one tenant's data to another or let one tenant's configuration affect
   another, and that resource quotas prevent one tenant from exhausting capacity and denying service to the
   rest. A shared service that serves all tenants without isolation, or missing quotas, is a cross-tenant
   effect.

6. **Confirm and record.** Confirm by acting as one tenant to read another tenant's resource, reach another's
   pod, or exhaust a shared resource, across non-production tenants and without touching real data. Kill the
   lead if tenant RBAC is namespace-scoped with no cluster-scoped or cross-namespace grants, default-deny
   network policy separates tenants, quotas bound each tenant, and shared services do not leak across tenants.
   Record the tenant source, the cross-tenant resource or namespace sink, and the isolation the namespace did
   not enforce.

## Where the namespace boundary leaks

- **Cluster-scoped grants punch through.** A cluster role or cross-namespace binding lets a tenant read or act
  outside its namespace at the API layer.
- **No network policy means namespaces are not isolated.** Without a default-deny, pods in one tenant namespace
  reach another's services directly.
- **Shared nodes host multiple tenants.** Node co-tenancy plus a container escape reaches another tenant's pods
  on the same node.
- **Cluster-scoped resources are visible across tenants.** Node objects, volumes, and custom resource
  definitions sit outside every namespace and are reachable across them.
- **Shared services and missing quotas create cross-tenant effects.** A shared service that serves all tenants,
  or an unquota'd tenant, lets one tenant observe or starve another.

## Worked example (a confirm and a kill)

> **Confirm.** Each tenant has a namespace, but a convenience cluster role grants tenants read on a
> cluster-scoped resource, and there is no network policy between namespaces. Acting as one tenant, a request
> lists a cluster-scoped object revealing another tenant's configuration, and a pod reaches another tenant's
> internal service directly. **Confirmed** namespace boundary failure across RBAC and network, `high`,
> remediation = remove cluster-scoped and cross-namespace grants from tenant roles, apply a default-deny
> network policy between tenant namespaces, and add resource quotas per tenant.
>
> **Kill.** Every tenant identity is bound only within its own namespace with no cluster-scoped or
> cross-namespace grant, a default-deny network policy separates tenant namespaces, resource quotas bound each
> tenant, and shared services do not expose one tenant's data to another. A tenant cannot read another's
> resources, reach its pods, or exhaust shared capacity. **Killed**, `kill_reason` = "tenant RBAC
> namespace-scoped with no cluster-scoped grants, default-deny network policy between tenants, quotas enforced,
> and shared services isolated; the namespace confines the tenant."

## Rationalizations to reject

- *"Each tenant has its own namespace."* → A namespace scopes names, not security; test RBAC reach, network
  reach, and shared resources before assuming it isolates.
- *"RBAC is set per namespace."* → Check for cluster-scoped roles and cross-namespace bindings; one cluster-
  scoped grant reaches every namespace.
- *"Pods are separated by namespace."* → Without a default-deny network policy, namespaces do not separate
  traffic; confirm segmentation between tenants.
- *"They share the cluster, that is expected."* → Sharing nodes and cluster-scoped resources is exactly where a
  namespace fails to isolate; audit the shared layer, not just the namespaces.
- *"One tenant cannot affect another."* → Confirm quotas exist; without them one tenant can exhaust shared
  capacity and deny service to the rest.

## Executing this in practice

You need each tenant's RBAC bindings and their scope, the network policy between tenant namespaces, the
cluster-scoped resources and nodes tenants share, the shared cluster services, and the resource quotas. For
each isolation the boundary is assumed to provide, test whether one tenant can breach it. Reading the RBAC,
policies, and quotas shows the intended boundary; acting as one tenant to reach another shows whether it holds.

## Related

- `auditing-multi-tenant-isolation` - the general multi-tenant discipline; this skill applies it specifically to
  the namespace-as-boundary assumption in Kubernetes.
- `auditing-network-policy-segmentation-gaps` - the network half of the boundary; a namespace without inter-
  tenant network policy is not isolated.
- `hunting-container-escape-surface` - the node co-tenancy risk; a shared node plus an escape reaches another
  tenant's pods regardless of namespace.
- `auditing-kubernetes-workload-and-rbac-hardening` - the RBAC-scoping discipline this boundary depends on;
  cluster-scoped grants are where it breaks.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the tenant confined to a namespace, sink = the
  cross-tenant resource or namespace it reaches, evidence = the isolation the namespace did not enforce.
