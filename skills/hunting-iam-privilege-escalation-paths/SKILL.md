---
name: hunting-iam-privilege-escalation-paths
description: >-
  Hunt privilege-escalation paths in cloud identity and access management: a
  low-privileged principal that chains role assumptions, policy rewrites,
  role-passing, and over-broad trust relationships to reach an administrative or
  data-access principal. Covers the identity-to-permission-to-resource graph, the
  known escalation primitives (passing a more privileged role to a service,
  rewriting a policy to a permissive version, assuming a role whose trust condition
  is too loose), and the boundary controls that should stop the chain. Use when
  reviewing cloud IAM, role and policy configuration, or an identity graph. A
  reachable path from an untrusted principal to admin is the finding.
license: MIT
---

# Hunting IAM privilege-escalation paths: a reachable chain to admin is the bug

Cloud access is a graph. Principals hold policies, policies grant permissions,
and some of those permissions let a principal change the graph itself: pass a
role, rewrite a policy, assume another identity. Privilege escalation is a path
through that graph from where an attacker starts to a principal that can read the
data or run the workloads that matter. No single permission looks alarming. The
bug is the reachable chain, and you only see it when you follow the edges.

## When to use

- You are reviewing cloud IAM: roles, policies, groups, and trust relationships.
- A principal is meant to be low-privileged and you want to prove it cannot reach admin.
- You have the policy documents and want to know what they compose into, not just what each says.

## Scope check

Audit identity configuration in accounts you own or are authorized to test, with
credentials provisioned for the review. If you can't name the authorization, stop.

## The loop

1. **Build the identity graph.** Inventory every principal, the policies attached
   to it, the permissions those policies grant, and the resources they touch.
   Represent it as edges: principal grants permission on resource. The whole method
   is reachability over this graph, so the graph is the first artifact.

2. **Enumerate the escalation primitives.** Mark every permission that lets a
   principal change the graph rather than just use it: passing a more privileged
   role to a compute or automation service, creating or rewriting a policy version,
   attaching a policy to yourself, updating a function or task that already runs as
   a privileged role, or minting credentials for another identity. These edges are
   how one node reaches another's privilege.

3. **Map trust relationships and their conditions.** For each assumable identity,
   read who is allowed to assume it and under what condition. A trust that names a
   broad set of principals, a wildcard, or a condition that an attacker can satisfy
   is an edge into that identity. A precise trust condition is not.

4. **Search for a path from an untrusted start to a privileged target.** Starting
   from a low-privileged or externally reachable principal, follow escalation and
   trust edges toward any principal that can read sensitive data or administer the
   account. Record the exact sequence of permissions the path uses.

5. **Check the boundary controls.** A path is only real if nothing stops it.
   Confirm whether an organization guardrail, a permission boundary, or a session
   policy denies a step on the chain. A boundary that caps the effective permission
   breaks the path; its absence leaves the path live.

6. **Confirm and record.** Confirm a path by walking it with the starting
   credentials in an authorized account and reaching the target privilege. Kill the
   lead if a boundary denies a step, the trust condition cannot be met, or no
   escalation edge connects the start to any privileged node. Record the full
   witness path: each principal, each permission used, and the target reached.

## Where IAM privilege escalation hides

- **The dangerous permissions are the ones that change identity.** Passing a role,
  rewriting a policy, and assuming an identity matter far more than any single read.
- **Trust conditions are edges.** A loose trust relationship is a door into a more
  privileged identity, and it is easy to write one wider than intended.
- **A chain of harmless steps is not harmless.** Each permission on the path can be
  defensible alone; the composition is the escalation.
- **Boundaries decide whether a path is real.** The same permissions escalate in an
  account with no guardrail and dead-end in one with a boundary that caps them.

## Worked example (a confirm and a kill)

> **Confirm.** A build principal may pass any role to the automation service and
> may update the automation job definition. It passes a role that has administrative
> policies, points a job at it, and runs the job, which now executes as admin.
> Walking it from the build credentials reaches account administration.
> **Confirmed** role-pass to admin, `critical`, remediation = restrict which roles
> the principal may pass to the exact roles it needs, and separate job-definition
> rights from role-passing rights.
>
> **Kill.** A principal can rewrite policy versions, which looks like an escalation
> primitive, but a permission boundary on that principal caps its effective
> permissions to a read-only set, and every privileged role's trust condition names
> a specific service under a condition the principal cannot satisfy. Walking every
> candidate path dead-ends at the boundary. **Killed**, `kill_reason` = "permission
> boundary caps effective permissions; privileged trust conditions unsatisfiable
> from this principal; no path reaches a privileged node."

## Rationalizations to reject

- *"Each permission is scoped and reviewed."* → Escalation is the composition, not
  the single grant. Follow the edges the permissions create.
- *"That role is only for the CI service."* → Then the trust condition must pin it to
  that service under a condition an attacker cannot forge. Read the condition.
- *"There is a guardrail on the account."* → Confirm it denies a step on this
  specific path. A guardrail that does not cover the primitive does not stop it.
- *"You would need several permissions to pull that off."* → An attacker who lands
  on the starting principal has them. That is the path.

## Executing this in practice

You need the full set of policy and trust documents, a way to compose them into a
reachability graph rather than reading each in isolation, and authorized credentials
to walk a candidate path to the target. A graph that treats principals as nodes and
identity-changing permissions and trust conditions as edges is ideal, because the
finding is a path and a path is a graph query; the walk from the starting credential
is the confirmation.

## Related

- `auditing-cicd-oidc-trust` - the pipeline identity that often is the untrusted
  starting principal for one of these paths.
- `hunting-non-human-identity-and-secret-reachability` - the leaked machine
  credential that lands an attacker on the starting node.
- `auditing-declarative-authorization` - the application-layer counterpart, where
  authorization is configuration rather than identity-graph edges.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the low-privileged starting
  principal, sink = the administrative or data-access principal the path reaches.
