---
name: auditing-network-policy-segmentation-gaps
description: >-
  Audit cluster network segmentation for the reachability a workload should not have: a namespace with no
  default-deny so every pod can reach every other pod, a missing egress policy that lets a compromised pod
  call out to the internet or the cloud metadata endpoint, an overly broad selector that admits more sources
  than intended, and a policy that governs one direction while the other stays open. Covers Kubernetes
  network policies and equivalent segmentation where pod-to-pod, pod-to-service, and pod-to-external
  reachability is meant to be constrained. Use when network policy is the control limiting lateral movement
  and egress in a cluster. The reachable source pod is the source, the pod, service, or external endpoint it
  can reach is the sink, and the segmentation gap that permits the reach is the bug.
license: MIT
---

# Auditing network segmentation gaps: when every pod can reach every pod

Kubernetes networking is open by default: without a policy, every pod can talk to every other pod, every
service, and often the internet and the cloud metadata endpoint. Network policy is what closes that down, so
segmentation is not a property the cluster has, it is a property each namespace earns by having policies that
actually constrain reachability in both directions. The gaps are predictable: a namespace with no
default-deny where the whole open default still applies, a missing egress rule that lets a compromised pod
exfiltrate or reach the node metadata endpoint, a selector broad enough to admit sources it never meant to,
and a policy that locks ingress while egress stays wide open. Lateral movement and egress are exactly what an
attacker does after a foothold, and segmentation is what limits both. You audit this by checking, per
namespace and per workload, what can actually reach what.

## When to use

- Network policy is the control meant to limit pod-to-pod, pod-to-service, or pod-to-external reachability.
- Namespaces may lack a default-deny, so the open-by-default reachability still applies.
- Egress may be unconstrained, letting a compromised pod reach the internet or the cloud metadata endpoint.

## Scope check

Test segmentation only on clusters you own or are authorized to assess, on non-production namespaces. Probing
reachability sends traffic between real pods and possibly outbound, so keep probes benign and inside the
authorized namespaces, and do not exfiltrate through any egress gap you find. If you can't name the
authorization, stop.

## The loop

1. **Establish the intended reachability first.** Name, per workload, what it should be able to reach and what
   should be able to reach it: which services, which namespaces, what egress. This is the false-positive
   killer: a namespace with a default-deny and explicit allow rules that match exactly the intended
   reachability is correctly segmented. Name the intended graph, then compare actual reachability to it.

2. **Check for a default-deny per namespace.** Determine whether each namespace has a default-deny for ingress
   and egress, or whether the open default applies. A namespace without default-deny gives every pod full
   reachability regardless of any specific allow rules, so the absence of default-deny is the first and biggest
   gap.

3. **Audit egress explicitly.** For each workload, check whether egress is constrained. Unconstrained egress
   lets a compromised pod call out to the internet to exfiltrate or fetch a payload, and reach the cloud
   metadata endpoint to grab node credentials. Egress is frequently forgotten while ingress is locked; confirm
   both directions.

4. **Check selector breadth on allow rules.** For each allow policy, read the pod, namespace, and IP selectors
   and confirm they admit only the intended sources or destinations. A selector broader than needed, an empty
   selector that matches everything, or a namespace label an attacker workload can wear, admits more than
   intended.

5. **Reconcile both directions and cross-namespace paths.** Confirm that ingress and egress policies agree and
   that cross-namespace reachability matches intent: a service meant to be internal is not reachable from a
   namespace that should be walled off, and a policy governing one direction is not undone by the other staying
   open. Map the effective reachability graph, not each rule in isolation.

6. **Confirm and record.** Confirm by reaching, from a source pod, a pod, service, or external endpoint it
   should not be able to reach, on non-production namespaces and without exfiltrating. Kill the lead if every
   namespace has a default-deny, egress is constrained (including to the metadata endpoint), selectors admit
   only intended sources, and both directions agree with the intended graph. Record the source pod, the reached
   sink, and the segmentation gap that permitted it.

## Where segmentation gaps leak

- **No default-deny means fully open.** A namespace without default-deny keeps the open-by-default
  reachability, so specific allow rules add nothing.
- **Unconstrained egress enables exfiltration and metadata theft.** A pod with open egress can reach the
  internet and the cloud metadata endpoint after a compromise.
- **A broad selector admits extra sources.** An overly wide or empty selector, or one keyed on an
  attacker-wearable label, lets in more than the intended source set.
- **One-directional policy leaves the other open.** Locking ingress while egress stays open, or vice versa,
  leaves half the reachability unconstrained.
- **Cross-namespace reachability is the lateral path.** An internal service reachable from a namespace that
  should be isolated is the move an attacker makes after a foothold.

## Worked example (a confirm and a kill)

> **Confirm.** An application namespace has ingress policies on its front-end service but no default-deny and no
> egress policy. A pod in that namespace, standing in for a compromised workload, reaches the cloud metadata
> endpoint and an internal database in another namespace that the front end never needed. **Confirmed**
> segmentation gap enabling lateral reach and metadata access, `high`, remediation = apply a default-deny for
> ingress and egress in the namespace, add explicit egress rules that exclude the metadata endpoint and the
> internet, and scope cross-namespace allow rules to the exact services required.
>
> **Kill.** Every namespace has a default-deny for both directions, egress is restricted to the specific
> destinations each workload needs with the metadata endpoint blocked, allow selectors match only the intended
> pods and namespaces on labels the cluster controls, and ingress and egress agree with the intended graph. A
> source pod reaches only its permitted destinations. **Killed**, `kill_reason` = "default-deny in every
> namespace with constrained egress, metadata blocked, and tight selectors in both directions; no pod reaches
> anything outside the intended reachability graph."

## Rationalizations to reject

- *"We have network policies."* → Confirm a default-deny exists; without it the open default applies and your
  allow rules only add to a fully open baseline.
- *"Ingress is locked down."* → Check egress too; an open egress lets a compromised pod exfiltrate and reach
  the metadata endpoint regardless of ingress.
- *"The cluster is internal."* → Internal still has lateral movement and a metadata endpoint; segmentation
  limits an attacker who already has one foothold.
- *"The selector targets our app."* → Confirm the selector cannot be matched by another workload's labels; a
  broad or wearable selector admits more than your app.
- *"Nothing sensitive is in that namespace."* → Cross-namespace reachability is the lateral path; a reachable
  internal service is a target even if the source namespace looks unimportant.

## Executing this in practice

You need, per namespace, whether a default-deny exists; per workload, the ingress and egress allow rules and
their selectors; the egress posture toward the internet and the metadata endpoint; and the intended
reachability graph. For each workload, compute what it can actually reach in both directions and compare to
intent. Reading the policies shows the intended segmentation; reaching a should-be-unreachable endpoint from a
source pod shows whether it holds.

## Related

- `mapping-pod-to-cloud-credential-reach` - open egress to the metadata endpoint is how a pod reaches the node
  role; segmentation and credential reach compound here.
- `auditing-namespace-as-tenant-boundary` - network policy is one of the controls a namespace-as-tenant
  boundary depends on; audit segmentation as part of that boundary.
- `auditing-service-mesh-mtls-and-authz-trust` - mesh authorization is the layer-7 companion to layer-3/4
  network policy; the two together define who can reach whom.
- `mapping-attack-surface` - use it to enumerate the services and external endpoints reachable from a namespace
  before auditing the policies that should constrain them.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the reachable source pod, sink = the pod, service, or
  external endpoint it reaches, evidence = the segmentation gap that permitted the reach.
