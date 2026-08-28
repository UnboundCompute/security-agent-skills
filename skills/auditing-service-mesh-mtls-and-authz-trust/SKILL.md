---
name: auditing-service-mesh-mtls-and-authz-trust
description: >-
  Audit a service mesh for the trust it claims but does not enforce: a mesh in permissive mode that accepts
  plaintext alongside mutual TLS so an unauthenticated caller still gets through, an authorization policy that
  is absent, scoped too narrowly, or defaults to allow, a workload reachable outside the mesh that bypasses
  the sidecar entirely, and an identity check that trusts a header a caller can set. Covers service meshes
  where sidecars are supposed to enforce mutual TLS and per-service authorization between workloads. Use when
  a mesh is the control asserting who may call whom and that calls are authenticated. The unauthenticated or
  unauthorized caller is the source, the called service is the sink, and the permissive mode or missing authz
  policy that admits it is the bug.
license: MIT
---

# Auditing service mesh mTLS and authz trust: when the mesh asserts a trust it does not enforce

A service mesh is sold as identity and encryption between workloads: sidecars establish mutual TLS so every
call is authenticated, and authorization policies decide who may call whom. Teams then reason about the
cluster as if those guarantees hold everywhere. The gaps are where the guarantee is asserted but not enforced.
A mesh in permissive mode accepts plaintext next to mutual TLS, so an unauthenticated caller is still served
while the dashboards show encryption. Authorization frequently defaults to allow, or a policy covers some
services and not the one that matters. A workload reachable outside the mesh, directly by pod IP or through
an ingress that skips the sidecar, bypasses every mesh control. And an identity check that trusts a caller-set
header authenticates nothing. You audit this by testing whether the mesh actually refuses an unauthenticated
and an unauthorized call, rather than trusting that it does.

## When to use

- A service mesh is the control enforcing mutual TLS and per-service authorization between workloads.
- The mesh may run in permissive mode, accepting plaintext alongside mutual TLS.
- Authorization policies may be missing, default-allow, narrowly scoped, or bypassable outside the sidecar.

## Scope check

Test a mesh only on clusters you own or are authorized to assess, on non-production services. Probing means
sending calls between real workloads, including unauthenticated ones, so keep them benign and inside the
authorized namespaces. If you can't name the authorization, stop.

## The loop

1. **Establish the intended call graph first.** Name which services may call which, and that every call must be
   mutually authenticated. This is the false-positive killer: a mesh in strict mutual-TLS mode with
   authorization policies that allow exactly the intended edges and deny by default is enforcing its claim.
   Name the intended graph and the authentication requirement, then test against them.

2. **Check the mutual-TLS mode.** Determine whether the mesh enforces strict mutual TLS or runs in permissive
   mode. Permissive mode accepts plaintext alongside mutual TLS, so an unauthenticated caller reaching the
   sidecar is still served. Confirm strict mode on the workloads that matter, cluster-wide and per-namespace,
   since the setting can be overridden locally.

3. **Audit authorization policy presence and default.** For each service, check whether an authorization policy
   exists and whether the effective default is deny or allow. A mesh with mutual TLS but no authorization
   authenticates callers without restricting them, so any authenticated workload can call any service. Confirm
   policies exist and default to deny.

4. **Check policy scope and identity basis.** For each authorization policy, confirm it covers the target
   service and that it decides on the verified peer identity from the mutual-TLS certificate, not on a
   caller-set header, source IP, or other spoofable attribute. A policy scoped to the wrong workloads, or keyed
   on a forgeable attribute, does not constrain the caller it should.

5. **Find paths that bypass the sidecar.** Determine whether any workload is reachable outside the mesh:
   directly by pod IP, through an ingress that terminates before the sidecar, or via a port the sidecar does
   not capture. A service reachable off-mesh is subject to none of the mesh's mutual-TLS or authorization
   controls, so the mesh's guarantees do not apply to it.

6. **Confirm and record.** Confirm by making an unauthenticated call that permissive mode accepts, an
   authenticated-but-unauthorized call that a missing or default-allow policy admits, or a direct off-mesh call
   that skips the sidecar, in a non-production namespace and benignly. Kill the lead if the mesh enforces strict
   mutual TLS, authorization policies exist and default to deny on verified peer identity, and no workload is
   reachable outside the sidecar. Record the caller source, the called-service sink, and the permissive mode or
   missing authz policy that admitted it.

## Where mesh trust leaks

- **Permissive mode serves plaintext.** A mesh that accepts unauthenticated calls alongside mutual TLS
  authenticates nothing for those calls while appearing encrypted.
- **Mutual TLS without authorization restricts no one.** Authenticated callers with no authz policy can call
  any service; identity without authorization is only half the control.
- **Default-allow or wrong-scope policies miss the target.** A policy that defaults to allow, or does not cover
  the service that matters, leaves it open.
- **Header- or IP-based identity is spoofable.** An authorization decision on a caller-set header or source IP
  is not the verified mesh identity and can be forged.
- **An off-mesh path skips every control.** A workload reachable by pod IP or a sidecar-bypassing ingress is
  subject to none of the mesh's guarantees.

## Worked example (a confirm and a kill)

> **Confirm.** A namespace runs the mesh in permissive mode, and a sensitive internal service has no
> authorization policy. An unauthenticated plaintext call from a test pod reaches the service, and an
> authenticated call from a workload that should never call it also succeeds. **Confirmed** mesh trust bypass
> via permissive mode and missing authorization, `high`, remediation = enforce strict mutual TLS on the
> namespace, add an authorization policy that defaults to deny and allows only the intended caller identities,
> and ensure the service is reachable only through the sidecar.
>
> **Kill.** The mesh enforces strict mutual TLS cluster-wide with no permissive override on the target
> namespace, every service has an authorization policy that defaults to deny and allows only the intended peer
> identities from their verified certificates, and no workload is reachable outside its sidecar by pod IP or a
> bypassing ingress. An unauthenticated or unauthorized call is refused. **Killed**, `kill_reason` = "strict
> mutual TLS with default-deny authorization on verified identity and no off-mesh path; the mesh refuses every
> unauthenticated and unauthorized call."

## Rationalizations to reject

- *"The mesh gives us mutual TLS."* → Confirm strict mode; in permissive mode an unauthenticated caller is
  still served, so the guarantee is not enforced.
- *"Everything in the mesh is encrypted."* → Encryption is not authorization; without a default-deny policy any
  authenticated workload can call any service.
- *"We have authorization policies."* → Confirm they cover this service, default to deny, and key on verified
  identity, not a header or IP a caller can set.
- *"Only mesh workloads can reach it."* → Check for a pod-IP or sidecar-bypassing path; a service reachable
  off-mesh is subject to none of the mesh's controls.
- *"The sidecar handles it."* → Only for traffic the sidecar captures; a port or ingress that skips the sidecar
  skips the mesh entirely.

## Executing this in practice

You need the mutual-TLS mode cluster-wide and per-namespace, the authorization policy for each service with its
default and identity basis, and every path by which a workload can be reached outside its sidecar. For each
service, test an unauthenticated call, an unauthorized authenticated call, and an off-mesh call. Reading the
mesh configuration shows the claimed trust; a call the mesh should refuse but serves shows whether it holds.

## Related

- `auditing-network-policy-segmentation-gaps` - the layer-3/4 companion; network policy and mesh authorization
  together define who can reach whom, and a gap in either undermines the other.
- `auditing-tls-and-certificate-validation` - the certificate-verification layer the mesh's mutual-TLS identity
  rests on; weak validation there undermines every authorization decision above it.
- `auditing-namespace-as-tenant-boundary` - mesh authorization is one control a namespace tenant boundary
  leans on; audit them together.
- `mapping-attack-surface` - use it to find services reachable outside the mesh before testing the mesh's
  own controls.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the unauthenticated or unauthorized caller, sink =
  the called service, evidence = the permissive mode or missing authorization policy that admitted it.
