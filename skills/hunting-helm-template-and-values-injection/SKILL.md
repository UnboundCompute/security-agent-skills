---
name: hunting-helm-template-and-values-injection
description: >-
  Hunt injection through Kubernetes packaging templates and their values: an untrusted value rendered into a
  manifest without quoting so it injects YAML structure, a value that flows into a container command, an
  annotation, or an RBAC rule and grants more than intended, and a chart that renders privileged security
  context or host access from a caller-supplied value. Covers Helm-style templating where a values file or a
  user-supplied override is rendered into Kubernetes manifests, and where an unescaped or unconstrained value
  becomes structure, a command, or a permission. Use when charts render manifests from values that a tenant,
  a pipeline, or a user can influence. The untrusted value rendered into the manifest is the source, the
  template render is the sink, and the injected YAML structure or widened permission is the bug.
license: MIT
---

# Hunting Helm template and values injection: when a value becomes a manifest

Kubernetes packaging templates turn a values file into manifests, and that rendering is a text
transformation with the same injection hazards as any templating: a value spliced in without quoting can
break out of its intended field and inject new YAML structure, and a value that flows into a sensitive field
can grant more than the author intended. When the values come from a place a tenant, a pipeline, or a user
can influence, an override that self-service platforms and multi-tenant chart deployments commonly allow, the
rendered manifest can gain a privileged security context, a host mount, an extra RBAC rule, or an altered
container command. The manifest looks authored by the chart, but part of it was supplied by the caller. You
hunt this by tracing untrusted values into the template and finding where they become structure or reach a
sensitive field.

## When to use

- Charts render Kubernetes manifests from values that a tenant, pipeline, or user can influence or override.
- A values field is rendered into a manifest without quoting, or into a command, annotation, or RBAC rule.
- A self-service platform lets callers supply overrides that reach a chart's templates.

## Scope check

Test chart rendering only against charts and clusters you own or are authorized to assess, on non-production
namespaces. A confirming override can create a privileged workload, so render and inspect manifests rather
than applying them where possible, and stay inside the authorized cluster. If you can't name the
authorization, stop.

## The loop

1. **Establish which values are untrusted first.** Determine which values a caller (tenant, pipeline, user)
   can set or override versus which are fixed by the chart author. This is the false-positive killer: a value
   hardcoded by the author and never overridable is not an injection source no matter how it is rendered. Name
   the untrusted values, then trace them.

2. **Trace untrusted values into the template render.** Follow each caller-influenceable value into the
   templates. The key distinction is whether the value is rendered as a quoted scalar into one field, or
   spliced in unquoted where it can introduce YAML structure, new keys, list items, or nested objects. An
   unquoted untrusted value is the structural-injection vector.

3. **Find sensitive fields the value can reach.** Identify where a value renders into a security-relevant
   field: the security context (privileged, run-as-root, capabilities), host mounts and host networking,
   container command and args, service-account binding, RBAC rules, and annotations that drive controllers.
   A value reaching any of these can widen permission or alter execution even without structural injection.

4. **Check quoting and constraint at each render.** A value rendered through proper quoting into a fixed field
   is safe; a value rendered raw, or into a field whose whole content it controls, is not. Confirm the chart
   quotes untrusted scalars, constrains which fields a value can populate, and does not let a value select or
   append privileged settings. Schema validation on the values, if present, is part of this.

5. **Assess the rendered manifest, not just the template.** Render the chart with a crafted override and read
   the resulting manifest: did the value stay in its field, or did it inject a new key, flip a security
   context, or add an RBAC rule? The rendered output is the ground truth for whether the boundary held.

6. **Confirm and record.** Confirm by supplying an in-scope override that injects YAML structure or renders a
   privileged setting into the manifest, inspecting the rendered output without applying a privileged workload
   to a real cluster. Kill the lead if every untrusted value is quoted into a fixed field, no value reaches a
   security-sensitive field it can widen, and values are schema-validated. Record the value, the template
   sink, and the injected structure or widened permission.

## Where Helm template and values injection leaks

- **An unquoted value injects YAML structure.** A caller value spliced in raw can add keys, list items, or
  nested objects the author never wrote.
- **A value into the security context widens privilege.** Rendering a caller-supplied setting into privileged,
  run-as-root, or capabilities turns an override into a privileged workload.
- **Host mounts and host networking are one field away.** A value that reaches a volume or hostNetwork setting
  breaks the container boundary.
- **RBAC rules rendered from values grant permissions.** A value that appends a rule or binds a service account
  grants cluster access from a chart override.
- **The template hides the injection.** The chart looks authored and safe; the crafted part appears only in the
  rendered manifest, so review the output, not just the source.

## Worked example (a confirm and a kill)

> **Confirm.** A multi-tenant platform lets each tenant supply values to a shared chart. A pod annotation field
> renders a tenant value unquoted, and a crafted value injects an additional key that sets a privileged
> security context on the container. Rendering the chart with the override shows the privileged context in the
> manifest. **Confirmed** values injection to privileged workload, `high`, remediation = quote every
> tenant-supplied scalar, constrain tenant values to a validated schema that cannot reach the security context
> or host fields, and render security-sensitive fields only from author-fixed values.
>
> **Kill.** Tenant values are validated against a schema that permits only non-privileged fields, every value
> is rendered quoted into a fixed field, and the security context, host settings, and RBAC rules are set
> entirely from author-controlled values a tenant cannot override. A crafted override renders inertly inside
> its field. **Killed**, `kill_reason` = "untrusted values schema-constrained and quoted into fixed fields with
> security-sensitive settings author-controlled; no override injects structure or widens permission."

## Rationalizations to reject

- *"Values are just configuration."* → Values are rendered into manifests; an unquoted or unconstrained value
  becomes structure or a permission, which is injection.
- *"The chart is ours."* → The chart being trusted does not make the caller's override trusted; trace the
  overridable values, not the author's fields.
- *"We quote most values."* → One unquoted untrusted value is enough to inject structure; confirm every
  caller-influenceable value is quoted and constrained.
- *"Tenants only set replica counts and names."* → Confirm the schema enforces that; without validation a tenant
  can often reach security-context and host fields.
- *"It only renders, it does not apply."* → In a self-service platform the render is applied; and even a render
  that widens permission is a finding once applied downstream.

## Executing this in practice

You need which values are caller-influenceable, how each is rendered (quoted scalar versus raw), the
security-sensitive fields a value can reach, and any values schema. For each untrusted value, render the chart
with a crafted override and inspect the manifest for injected structure or widened permission. Reading the
template shows the intent; the rendered manifest shows whether the boundary held.

## Related

- `auditing-kubernetes-workload-and-rbac-hardening` - the manifest-level companion; this skill finds how a
  value injects a privileged setting, that one audits the setting itself.
- `hunting-orm-and-query-builder-injection` - the same value-versus-structure distinction applied to queries;
  here the structure is YAML, not SQL.
- `auditing-iac-module-and-provider-supply-chain` - the chart's own supply chain: where the chart and its
  subcharts come from, alongside the values that drive them.
- `adjudicating-taint-paths` - use it to confirm a caller-supplied value reaches a security-sensitive template
  field through subchart and values indirection.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the untrusted value rendered into the manifest, sink =
  the template render, evidence = the injected YAML structure or widened permission.
