---
name: auditing-iac-module-and-provider-supply-chain
description: >-
  Audit the supply chain of infrastructure-as-code modules and providers for trust that runs at plan or
  apply time: a module sourced from an unpinned or attacker-influenceable location, a provider or plugin
  pulled from a registry without integrity pinning, a module that executes local commands or fetches remote
  content during planning, and a lockfile that is missing, ignored, or not enforced in CI. Covers Terraform
  and similar declarative tools where a module or provider runs with the credentials of whoever applies it.
  Use when infrastructure is built from third-party or shared modules and providers and the apply identity is
  privileged. The untrusted module or provider source is the source, the plan or apply execution is the sink,
  and the code running under the applier's credentials without integrity pinning is the bug.
license: MIT
---

# Auditing IaC module and provider supply chain: when planning runs someone else's code

Infrastructure-as-code feels declarative, but modules and providers are code, and they run with the
credentials of whoever plans or applies them, which in a pipeline is usually a highly privileged identity.
That makes the module and provider supply chain a direct path to those credentials. A module sourced from an
unpinned reference can change under you between runs; a provider or plugin pulled without integrity
verification can be swapped at the registry; a module that shells out or fetches remote content during
planning executes on the runner before anyone reviews an apply; and a lockfile that is absent or not enforced
lets the resolved versions drift. The declarative surface hides that a plan is an execution. You audit this by
finding where module and provider code enters and whether its source and integrity are pinned.

## When to use

- Infrastructure is built from third-party or shared modules and from providers pulled at init time.
- Module sources or provider versions are unpinned, or a dependency lockfile is missing or unenforced.
- Plan or apply runs in CI with a privileged identity, so executing module code reaches real credentials.

## Scope check

Audit IaC supply chain only for infrastructure and pipelines you own or are authorized to assess, on
non-production state. Confirming that a module executes at plan time runs code on the runner, so keep any
proof benign and inside the authorized pipeline. If you can't name the authorization, stop.

## The loop

1. **Establish the apply identity and its power first.** Name the credentials that plan and apply run under
   and what they can do in the environment. This is the false-positive killer that sets severity: an unpinned
   module matters far more when apply holds broad cloud permissions than when it is tightly scoped. Name the
   identity, then trace what code runs under it.

2. **Inventory every module and provider source.** List where each module comes from (a registry, a version-
   control reference, a local path) and every provider or plugin the configuration pulls. Each external source
   is code that will run under the apply identity. Distinguish first-party internal modules from third-party
   ones, since the trust question differs.

3. **Check source and version pinning.** For each module, confirm the source is pinned to an immutable
   reference (a specific version or a commit digest), not a mutable branch or a bare registry name that can
   resolve to a new release. For each provider, confirm the version is pinned and the dependency lockfile
   records and enforces the exact provider hashes. An unpinned source can change between runs without review.

4. **Find code that executes during plan.** Some module constructs run at planning time: local command
   execution, data sources that shell out, and provisioners or external programs that fetch or run content.
   These execute before an apply is even approved, so a malicious or compromised module reaches the runner's
   credentials during a plan. Locate every such construct and whether it takes untrusted input.

5. **Check lockfile enforcement in CI.** A lockfile only helps if CI initializes against it and fails when the
   resolved hashes do not match, rather than regenerating it silently. Confirm the pipeline enforces the
   lockfile, verifies provider integrity, and does not auto-upgrade. An ignored lockfile is no lockfile.

6. **Confirm and record.** Confirm by showing an unpinned module reference resolves to changeable content, or
   that a plan executes a benign marker command from a module under the apply identity, within the authorized
   pipeline and without a destructive payload. Kill the lead if every module and provider source is pinned to
   an immutable reference, the lockfile is enforced with integrity verification in CI, and no plan-time
   construct runs untrusted code. Record the source, the plan or apply sink, and the unpinned or executing
   dependency.

## Where IaC supply chain leaks

- **A plan is an execution.** Modules and providers run with the applier's credentials, so planning is not a
  safe read-only step when the source is untrusted.
- **An unpinned source changes under you.** A mutable branch or a bare registry reference can resolve to new
  content between runs with no review.
- **Plan-time command execution reaches the runner early.** A module that shells out during planning runs
  before any apply approval gate.
- **A provider is code with credentials.** An unpinned or unverified provider plugin executes at init and
  apply with the same power as the configuration.
- **An unenforced lockfile is decoration.** If CI regenerates or ignores it, the pinned versions drift and the
  integrity guarantee is lost.

## Worked example (a confirm and a kill)

> **Confirm.** A pipeline consumes a shared module referenced by a mutable branch, and the module includes a
> data source that executes a local command during plan. The plan runs in CI under a privileged apply role.
> Pointing the branch at updated content runs a benign marker command under the apply role at plan time,
> before any approval. **Confirmed** IaC supply-chain execution under the apply identity, `high`, remediation
> = pin the module to an immutable commit digest, remove or sandbox the plan-time command execution, and gate
> the apply role behind approval that the plan cannot bypass.
>
> **Kill.** Every module is pinned to a commit digest and every provider to a version recorded in an enforced
> lockfile with hash verification in CI, no module runs commands or fetches remote content at plan time, and
> the apply identity is scoped to the resources the configuration manages. A changed upstream cannot alter a
> run without a reviewed pin bump. **Killed**, `kill_reason` = "modules and providers pinned to immutable
> references with an enforced lockfile, no plan-time code execution, and a scoped apply identity; no untrusted
> code runs under the applier's credentials."

## Rationalizations to reject

- *"Terraform is declarative, plan is read-only."* → Plan runs module code, data sources, and providers; a
  plan-time command executes on the runner under the apply identity.
- *"The module is from our registry."* → Confirm it is pinned to an immutable version; a bare registry name
  can resolve to a new, unreviewed release.
- *"We have a lockfile."* → Confirm CI enforces it with integrity verification and does not regenerate it; an
  ignored lockfile pins nothing.
- *"Providers come from the official registry."* → Official does not mean integrity-pinned for your run; pin
  the version and verify the hash in the lockfile.
- *"Apply is gated by approval."* → Check that plan-time execution cannot reach credentials before the gate;
  the gate protects apply, not planning.

## Executing this in practice

You need the plan and apply identity and its power, every module and provider source, the pinning of each,
the plan-time execution constructs, and the lockfile enforcement in CI. For each dependency, ask whether its
source and integrity are pinned and whether it runs code before an apply gate. Reading the sources and the
lockfile shows the intended trust; showing an unpinned reference resolve to changed content, or a plan-time
marker command run, shows whether the boundary holds.

## Related

- `hunting-supply-chain-risks` - the general dependency-trust discipline; IaC modules and providers are a
  specific, credential-bearing case of it.
- `auditing-cicd-oidc-trust` - the pipeline identity that plan and apply run under is often a federated cloud
  role; scope it there and here together.
- `hunting-cicd-workflow-injection` - attacker-controlled repository data reaching a privileged run step is the
  same runner-credential threat from the workflow side.
- `auditing-infrastructure-as-code-exposures` - the effective-config side: what the resources declare, once
  the module and provider supply chain is trusted.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the untrusted module or provider source, sink = the
  plan or apply execution, evidence = the unpinned or plan-time-executing dependency under the apply identity.
