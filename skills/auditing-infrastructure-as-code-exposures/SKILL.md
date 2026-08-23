---
name: auditing-infrastructure-as-code-exposures
description: >-
  Audit existing infrastructure-as-code definitions (Terraform, OpenTofu, CloudFormation, Bicep,
  Pulumi) for resource state that would provision an insecure resource, after variables, modules, and
  account defaults are resolved. Covers storage exposed to the public, a security-group or firewall rule
  open to the whole internet on a sensitive port, an identity or resource policy with wildcard actions or
  principals, encryption left off or a snapshot or image shared publicly, logging or audit trails
  disabled, and a plaintext secret in a variable default or connection string. Use when reviewing the
  static definition files, not authoring or refactoring them, and not walking the runtime identity graph.
  The declared resource block is the source, the insecure provisioned resource it would create is the
  sink, and effective config that violates the baseline is the bug.
license: MIT
---

# Auditing infrastructure-as-code exposures: what the definition would actually provision

Infrastructure-as-code is desired state, so the bug is not what a line says but what the resource
becomes once variables resolve, modules compose, and account defaults apply. A bucket ACL that reads
public may be overridden by an account block, and a security group open to the internet may sit in a
private subnet nothing routes to. This audit reads the definition and asks one question per resource:
once applied, does this provision a resource that violates its security baseline, least exposure,
encryption on, least privilege, logging on, no plaintext secret. Everyone has the checklist; the work
that matters is adjudicating the effective config and killing the finding that a different layer already
neutralizes. Anchor on auditing what exists, not generating or refactoring it.

## When to use

- You have Terraform, OpenTofu, CloudFormation, Bicep, or Pulumi definitions for infrastructure in scope.
- A resource block declares storage, a network rule, an identity or resource policy, encryption, or logging.
- You want to know which blocks would provision an insecure resource once applied, not just which lines look wrong.

## Scope check

Audit only definitions for infrastructure you own or are authorized to assess, and never apply a
definition or mutate a live account to test a finding, adjudicate on the resolved config, not by
provisioning. If you can't name the authorization, stop.

## The loop

1. **Resolve the effective config first.** For each resource, follow the variables, locals, modules, and
   any tfvars, parameter files, or workspace overrides that feed its arguments, and factor in account or
   region defaults (default encryption, a block-public-access setting, an org policy). The literal in the
   module is not what deploys; the resolved state is. Settle it before judging any field.

2. **Check storage exposure.** Look for a bucket or object ACL of public-read or public-read-write, a
   bucket or resource policy with a wildcard principal, or block-public-access disabled, then confirm no
   account-level block or explicit deny overrides it. Public storage is the effective policy, not the ACL.

3. **Check network ingress.** Look for a security-group or firewall rule allowing the whole internet
   (an all-addresses source) on a sensitive port (remote admin, database, cache, search), then check
   whether the resource is actually reachable: a public subnet with an internet route and a public
   address, versus a private subnet a deny rule or absent route isolates.

4. **Check identity and resource policies.** Look for a policy granting wildcard actions or resources, an
   admin-equivalent managed policy, or a wildcard principal on a resource policy, then check for a
   condition, a permissions boundary, or an organization control that caps the effective grant. A
   wildcard narrowed by a real condition is not the same finding as an open one.

5. **Check encryption, logging, and plaintext secrets.** Look for encryption switched off or a snapshot
   or machine image shared publicly, access or audit logging or versioning disabled where the resource
   needs it, and a hardcoded password, key, or token in a variable default, a sensitive value's default,
   or a connection string. For a secret, report only that it is present in the definition; hand its
   blast-radius to the secret-reachability skill.

6. **Confirm and record.** Confirm by showing the resolved config provisions the insecure resource and no
   other layer neutralizes it. Kill the lead if an account block or explicit deny overrides a public ACL,
   if the open rule attaches only to a resource with no internet path, if the absent encryption field
   still provisions encrypted under an account default, if a wildcard policy is capped by a condition or
   boundary or service-control policy, if the value comes from an apply-time override rather than the
   literal, if the resource is never instantiated (count zero, empty iteration, a disabled flag, an
   uncalled module), or if a hardcoded secret is a documented placeholder or resolved from a secret
   manager at apply. Record the block, the resolved state, and the baseline it violates.

## Where infrastructure-as-code leaks

- **The bug is the effective config, not the line.** Variables, modules, and account defaults change what
  provisions; adjudicate the resolved state, or you report the source and miss the deploy.
- **A public ACL under an account block is inert.** Storage exposure is the effective policy after
  overrides and denies, not the permissive ACL read alone.
- **An open rule with no route is not reachable.** A whole-internet ingress on a private-subnet resource
  with no public address and no route exposes nothing; reachability is part of the finding.
- **A wildcard with a real condition is a different finding.** A source-address, org, tag, or boundary
  condition caps the grant; report the uncapped wildcard, not the constrained one.
- **A never-instantiated resource provisions nothing.** Count zero, an empty iteration, a disabled flag,
  or an uncalled module means the insecure block never deploys.

## Worked example (a confirm and a kill)

> **Confirm.** A security-group rule allows ingress from the whole internet on the database port; the
> group attaches to an instance with a public address in a public subnet routed to an internet gateway,
> and no deny rule intervenes. The resolved config exposes the database to the internet. **Confirmed**
> internet-open database ingress on a reachable instance, `high` rising to `critical` for a
> credential-store port, remediation = restrict the source to the application tier and remove the public
> address.
>
> **Kill.** A bucket declares a public-read ACL, but the account enforces block-public-access with a
> deny-all-public policy that overrides the ACL, so the applied bucket is private. **Killed**,
> `kill_reason` = "the public ACL is overridden by an account block-public-access setting that denies all
> public access, so the effective bucket policy is private."

## Rationalizations to reject

- *"The ACL says public, so it is public."* -> Not if an account block or a deny policy overrides it.
  Adjudicate the effective policy, not the ACL in isolation.
- *"The rule is open to the whole internet."* -> Is the resource reachable? An open rule on a private,
  unrouted resource with no public address exposes nothing.
- *"The IAM policy has a wildcard action."* -> Is it capped by a condition, boundary, or org policy? A
  constrained wildcard is a different, weaker finding than an open one.
- *"Encryption is not set."* -> Does the account default to encrypted? An absent field can still provision
  an encrypted resource; check the default before flagging.
- *"There is a password in the variable default."* -> Is it a documented placeholder, or resolved from a
  secret manager at apply? Only a real literal baked into the deploy is the finding.

## Executing this in practice

You need the definition files, the variables, modules, and apply-time overrides that resolve each
argument, the account and region defaults, and any org or boundary policy that caps a grant. For each
resource, resolve the effective config and decide whether it provisions an insecure resource no other
layer neutralizes. Reading the block tells you what it asks for; resolving the variables, defaults, and
overrides tells you what it actually provisions.

## Related

- `auditing-kubernetes-workload-and-rbac-hardening` - the sibling for the Kubernetes YAML plane; this
  skill owns cloud provider resources, that one owns RBAC subjects and workload security contexts.
- `auditing-container-image-build-hardening` - the sibling for the image build plane; a hardcoded secret
  or root default in a Dockerfile belongs there, cloud resource state belongs here.
- `hunting-non-human-identity-and-secret-reachability` - takes the blast-radius of a secret this skill
  finds present in a definition, and the runtime reach of an over-granted identity.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the declared resource block, sink = the insecure
  provisioned resource, evidence = the resolved effective config that violates the baseline.
