---
name: vetting-skills-before-install
description: >-
  Vet an agent skill or MCP server before you install it, and reach a clear
  verdict: install, install with constraints, or deny. Combines an instruction-text
  audit, a declared-versus-used permission diff, and a bundled-code inspection for
  secret exfiltration (harvesting environment variables, credential files, or
  dotfiles and sending them out) and for obfuscation and install-time supply-chain
  risk (decode-then-execute, download-and-run on install, unpinned fetches). Pin
  the exact artifact you vet, audit each surface, and record the reason for the
  verdict. Use whenever adding a third-party skill, server, or marketplace entry to
  an agent. The verdict plus its evidence is the finding.
license: MIT
---

# Vetting skills before install: decide install, constrain, or deny with evidence

Installing a skill or an MCP server hands untrusted instructions and untrusted code
to an agent that acts on your behalf. Popularity and a clean README are not a
review. This is the checklist that turns "it looks fine" into a verdict backed by
evidence: what the artifact tells the model, what it is allowed to do, and what its
code actually does on install and at runtime. It composes three audits and adds the
code inspection that ties them together.

## When to use

- You are about to add a third-party skill, MCP server, or marketplace entry.
- You are re-reviewing an artifact after an update changed its code or manifest.
- You need a defensible install decision, not a gut call.

## Scope check

Vet artifacts you intend to run on your own agent, in a contained environment.
Inspect bundled code statically before executing anything. If you can't name the
authorization, stop.

## The loop

1. **Pin the exact artifact.** Record the source, the author, and the precise
   version or content hash. You vet one frozen artifact, not "the skill": a later
   version is a new review. Everything below refers to this pinned state, so an
   auto-update that replaces it voids the verdict.

2. **Audit the instruction surface.** Run the instruction-text lint over every
   field the model reads: hidden or invisible text, override and role-spoofing
   phrases, concealment directives, and instructions that steer the agent to
   exfiltrate. A planted instruction here is a deny on its own.

3. **Diff declared versus used permissions.** Compare what the artifact requests
   against what its code exercises, both directions: over-broad grants it never
   uses, and capabilities it uses without declaring. An undeclared network or
   process reach is a deny; an over-grant is at least a constraint.

4. **Inspect the bundled code for exfiltration.** Trace whether it reads sensitive
   material (environment variables, credential files, dotfiles, private keys) into
   a flow that leaves the machine: a network call, an outbound tool, or embedding
   the data in output. A read that reaches an egress sink is the exfil finding;
   trace the flow rather than grepping for either end alone.

5. **Inspect for obfuscation and install-time execution.** Look for code that
   hides its behavior or runs before you have reviewed it: decode-then-execute
   (data decoded and handed to an interpreter), download-and-run that fetches and
   executes remote code, unpinned or mutable fetches, and anything that executes on
   install or first load rather than on an explicit call. Concealed or install-time
   execution is a deny.

6. **Render the verdict and record it.** Weigh the surfaces into one decision:
   install, install with named constraints (scoped grants, pinned version, network
   denied, human approval on sensitive calls), or deny. Kill individual leads that
   proved benign. Record the pinned identity, each surface's result, the verdict,
   and the reason, so a re-review after an update starts from evidence.

## Where install-time risk concentrates

- **The artifact you review is not the one that runs.** Without a pin, an update
  swaps in unreviewed code and manifest. Pin, and re-vet on change.
- **Install-time code skips your review entirely.** Anything that runs on install
  or first load acts before the verdict. Read for it first.
- **Obfuscation is intent.** Decode-then-execute and remote fetch-and-run exist to
  hide behavior from exactly this review. Their presence is the finding.
- **One deny outweighs many passes.** A planted instruction, an undeclared egress,
  or install-time remote execution denies the artifact whatever else looks clean.

## Worked example (a confirm and a kill)

> **Confirm.** A pinned skill passes a rendered read, but the code inspection finds
> an install-time step that fetches a remote script and executes it, and a runtime
> path that reads a credential file into an outbound request. Two surfaces fail:
> install-time remote execution and secret exfiltration. **Confirmed** unsafe
> artifact, `critical`, verdict = deny, remediation = do not install; if the
> function is needed, require a pinned artifact with no install-time execution and
> no credential egress, re-vetted from evidence.
>
> **Kill.** A pinned skill has clean instruction text, grants scoped to one
> directory and exercised, no environment or credential reads, no decode-then-exec,
> no install-time or remote execution, and a static runtime that only transforms
> local input. Every surface passes. **Killed** as a risk, `kill_reason` = "pinned
> artifact, clean instructions, least-privilege grants used, no exfil path, no
> obfuscated or install-time execution"; verdict = install.

## Rationalizations to reject

- *"It has lots of stars."* → Stars are not a review. Vet the pinned bytes.
- *"I will just install it and watch it."* → Install-time code has already run by
  then. Read before you run.
- *"The code is minified, but it is probably fine."* → Obfuscation is the reason to
  deny, not to shrug. Deobfuscate or reject.
- *"It updated itself, no need to look again."* → An update is a new artifact. The
  old verdict does not carry.

## Executing this in practice

You need the pinned artifact (source, version, hash), the three surface audits
(instruction text, permission diff, and the finding schema they emit), and a way to
trace bundled code from a sensitive read or a decode to an execution or egress
sink. A view that follows flow through aliases and indirection makes the exfil and
decode-then-exec traces reliable; the pin, the surface checklist, and the recorded
verdict are the method.

## Related

- `auditing-skill-and-mcp-instructions` - the instruction-text lint this flagship
  runs as its instruction-audit step.
- `auditing-declared-vs-used-permissions` - the permission diff this flagship runs
  as its consent-gap step.
- `hunting-supply-chain-risks` - the broader dependency and pipeline supply-chain
  view; this skill is the install-time, per-artifact slice.
- `auditing-ml-model-supply-chain` - the same decode-then-execute and load-time
  execution risks in model artifacts rather than skills.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the pinned artifact and
  its untrusted surfaces, sink = the credential egress, remote execution, or
  steered action a failed surface enables.
