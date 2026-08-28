---
name: auditing-admission-control-policy-gaps
description: >-
  Audit cluster admission control for gaps that let a non-compliant or hostile workload through: a validating
  webhook that fails open when its backend is unreachable, a policy that scopes by namespace or label and
  misses the namespaces that matter, a mutating webhook whose changes are trusted downstream, and an ordering
  or exemption that lets a privileged pod bypass the checks meant to stop it. Covers Kubernetes admission
  webhooks and policy engines that are supposed to enforce workload security at create and update time. Use
  when admission policies are the control that keeps privileged, unsigned, or over-permissioned workloads out
  of the cluster. The workload submitted for admission is the source, the admitted resource is the sink, and
  the policy gap that let it through is the bug.
license: MIT
---

# Auditing admission control policy gaps: when the gatekeeper waves it through

Admission control is the cluster's gate: every create and update passes through validating and mutating
webhooks that are supposed to reject workloads that break policy, privileged pods, unsigned images,
over-broad service accounts, host mounts. The gate only protects the cluster if it actually fires on the
requests that matter and cannot be skipped. In practice it is riddled with quiet gaps: a webhook configured
to fail open so an unreachable backend admits everything, a policy scoped to some namespaces but not the one
an attacker targets, an exemption for system workloads that a hostile pod can wear, and a mutating webhook
whose output later stages trust without re-validating. A gate with a gap is worse than no gate, because teams
believe it is enforcing. You audit this by checking, for each policy, whether it fires on every relevant
request and whether anything can bypass it.

## When to use

- Admission webhooks or a policy engine are the control keeping non-compliant workloads out of the cluster.
- Webhooks may fail open, be scoped narrowly, carry exemptions, or run in an order that allows a bypass.
- A mutating webhook changes resources that later stages or other policies trust without re-checking.

## Scope check

Test admission control only on clusters you own or are authorized to assess, on non-production namespaces.
Probing a policy means submitting workloads to a real cluster, so use a non-production namespace and remove
any test workload you create. If you can't name the authorization, stop.

## The loop

1. **Establish what each policy must enforce and where first.** Name the rules admission control is supposed
   to guarantee (no privileged pods, only signed images, no host mounts) and the full set of namespaces they
   must cover. This is the false-positive killer: a policy that fires closed on every relevant request across
   every namespace, with no usable exemption, is doing its job. Name the intended coverage, then test against
   it.

2. **Check the failure mode of each webhook.** Read whether each validating webhook fails open or closed when
   its backend is unavailable or times out. A fail-open webhook admits everything the moment its backend is
   unreachable, so an attacker who can stress or block the backend disables the policy. Confirm security-
   critical webhooks fail closed.

3. **Map policy scope against the namespaces that matter.** Compare each policy's namespace and label selectors
   to the namespaces where a hostile workload could land. A policy scoped to a subset, or keyed on a label the
   submitter controls, misses the target namespace or is dodged by omitting the label. Find the gap between
   where the policy fires and where it needs to.

4. **Hunt exemptions and bypasses.** Identify every exemption: system namespaces, service-account allowlists,
   and objects excluded by name or label. Check whether a hostile workload can place itself in an exempt
   namespace, assume an exempt identity, or set an exempt label. An exemption an attacker can wear is a full
   bypass of the policy.

5. **Check mutating-webhook trust and ordering.** Where a mutating webhook injects or alters fields (sidecars,
   defaults, security context), confirm the result is still validated afterward and that the mutation cannot be
   used to satisfy or skip a later check. A mutation trusted downstream, or a webhook order that lets a resource
   avoid validation, is a structural gap.

6. **Confirm and record.** Confirm by admitting a workload that the policy should reject through a real gap,
   fail-open under backend pressure, an uncovered namespace, an exemption worn by the test workload, in a
   non-production namespace and removing it after. Kill the lead if every security-critical webhook fails
   closed, policies cover all relevant namespaces without submitter-controlled scoping, no exemption is
   attacker-assumable, and mutations are re-validated. Record the submitted workload, the admitted-resource
   sink, and the policy gap that let it through.

## Where admission gaps leak

- **Fail-open webhooks disable under pressure.** A webhook that admits on backend error is off the moment its
  backend is unreachable, which an attacker can arrange.
- **Narrow scope misses the target namespace.** A policy that fires in some namespaces but not the one a
  workload lands in never sees the request.
- **Submitter-controlled selectors are dodgeable.** Scoping on a label the submitter sets lets a workload omit
  the label and escape the policy.
- **An attacker-wearable exemption is a bypass.** An exempt namespace, identity, or label that a hostile
  workload can adopt skips the check entirely.
- **Trusted mutations skip validation.** A mutating webhook whose output is not re-validated lets an injected
  change satisfy or evade a later policy.

## Worked example (a confirm and a kill)

> **Confirm.** A policy engine forbids privileged pods, but its validating webhook is configured to fail open,
> and the same policy exempts a system namespace by name. Submitting a privileged pod while the webhook backend
> is briefly unreachable admits it, and submitting one directly into the exempt namespace admits it even when
> the backend is healthy. **Confirmed** admission bypass of the privileged-pod policy, `high`, remediation =
> set the security-critical webhook to fail closed, remove or tightly gate the namespace exemption, and scope
> policies on cluster-controlled attributes rather than submitter-set labels.
>
> **Kill.** Every security-critical webhook fails closed, the no-privileged-pod policy applies to all
> namespaces with no exemption a workload can place itself into, scoping uses attributes the submitter cannot
> forge, and mutating webhooks are followed by a validating pass over the final object. A privileged pod is
> rejected from every namespace and under backend failure. **Killed**, `kill_reason` = "webhooks fail closed,
> policy covers all namespaces with no attacker-wearable exemption, and mutations are re-validated; no submitted
> workload evades the gate."

## Rationalizations to reject

- *"The policy engine forbids that."* → Confirm the webhook fails closed and covers the target namespace; a
  fail-open or unscoped policy forbids nothing when it does not fire.
- *"Only system namespaces are exempt."* → Check whether a hostile workload can land in or assume a system
  namespace or identity; an exemption is only safe if it is unreachable.
- *"We scope by label."* → If the submitter sets the label, they can omit it; scope on attributes the cluster
  controls, not the request.
- *"The mutating webhook adds our security defaults."* → Confirm the mutated object is validated afterward; a
  trusted mutation is a place to smuggle a non-compliant field.
- *"Admission control is enabled."* → Enabled is not enforcing; a gate with a fail-open, a scope gap, or an
  exemption admits exactly what it was meant to stop.

## Executing this in practice

You need each webhook's failure policy, the namespace and label scope of every policy, the full exemption
list, the mutating-then-validating ordering, and the set of namespaces a hostile workload could target. For
each policy, ask whether it fires on every relevant request and whether anything can skip it. Reading the
webhook and policy configuration shows the intended gate; admitting a should-be-rejected workload through a
gap in a non-production namespace shows whether it holds.

## Related

- `auditing-container-image-provenance` - the image-signing policy is one of the rules admission control
  enforces; this skill checks whether that enforcement can be bypassed.
- `hunting-container-escape-surface` - the privileged and host-mount settings admission control is meant to
  block; a gap here lets exactly those workloads in.
- `auditing-kubernetes-workload-and-rbac-hardening` - the workload posture admission control enforces at
  create time; audit the policy and the posture together.
- `adjudicating-taint-paths` - use it to reason about whether a submitter-controlled attribute reaches the
  policy's scoping decision.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the workload submitted for admission, sink = the
  admitted resource, evidence = the policy gap that let it through.
