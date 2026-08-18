---
name: auditing-declared-vs-used-permissions
description: >-
  Find the consent gap in an agent skill or MCP server: the distance between the
  permissions and capabilities it declares and what its bundled code and
  instructions actually exercise. Covers over-broad grants a skill requests but
  never uses, capabilities it exercises without declaring, and grants that are used
  but still wider than the task needs. Read the declared surface in frontmatter or
  manifest, inventory the real behavior, and diff the two in both directions. Use
  when reviewing a skill or server before install, or auditing least privilege
  across an agent's installed set. An over-broad or undeclared grant is the
  finding.
license: MIT
---

# Auditing declared versus used permissions: the consent gap is the bug

A user grants a skill the permissions it asks for. The question no one checks is
whether it needs them. A skill that requests shell access and never shells out, or
network access and never makes a call, has widened the user's blast radius for
nothing, and a compromise or a later update inherits that headroom. Worse is the
other direction: a skill that acts beyond what it declared. This skill measures
the gap between the grant and the use, in both directions.

## When to use

- You are reviewing a skill, an MCP server, or a marketplace entry before install.
- You are auditing least privilege across an agent's installed skills and tools.
- You want to right-size a grant, not just confirm the artifact runs.

## Scope check

Audit skills and servers you own or are authorized to review. Inspect the bundled
code statically; run it only in a contained environment. If you can't name the
authorization, stop.

## The loop

1. **Read the declared permission surface.** Collect exactly what the artifact
   asks for: the allowed-tools or capability list in frontmatter or manifest, the
   scopes it requests, the network, filesystem, and process access it declares.
   This is the consent the user is asked to give. Record it as the grant set.

2. **Inventory what the code and instructions actually exercise.** Walk the bundled
   scripts and the instruction text and record the real capabilities used: which
   tools are invoked, whether it spawns a process, opens a socket, reads outside
   its own directory, or reaches the network. Prefer tracing the flow over a string
   grep, so an aliased or indirect call is not missed. This is the use set.

3. **Diff for over-grants: declared but never used.** Subtract use from grant.
   Every capability requested and never exercised is an over-grant: pure headroom
   the user consented to for no delivered function. This is the common consent gap
   and the easiest least-privilege win.

4. **Diff the other direction: used but not declared.** Subtract grant from use.
   A capability the code exercises without declaring is worse than an over-grant:
   the artifact acts beyond the consent it requested. Treat undeclared reach,
   especially network or process, as a high-signal finding.

5. **Weigh the used grants that remain.** A capability that is both declared and
   used can still be excessive. Ask whether the task needs the full scope or a
   narrower one: write to one directory rather than the filesystem, one host rather
   than the open network. Name the least-privilege version.

6. **Confirm and record.** Confirm each over-grant by showing the capability is
   never exercised, and each undeclared use by tracing the call. Kill the lead when
   a grant is genuinely exercised at the scope requested. Record the declared set,
   the used set, the direction of the gap, and the least-privilege grant.

## Where the consent gap opens

- **Requested is not needed.** The grant list is a request, not a requirement.
  Measure it against real use before you accept it.
- **Undeclared reach is the worse half.** A skill acting past its declaration broke
  consent, not just tidiness. Trace network and process capabilities specifically.
- **Copy-paste inflates grants.** Manifests cloned from a template carry
  capabilities the author never used. The unused ones are the tell.
- **Headroom is what a compromise inherits.** An over-grant is dormant until the
  skill is subverted or updated, then it is the attacker's budget. Right-size now.

## Worked example (a confirm and a kill)

> **Confirm.** A skill's manifest requests process execution and open network
> access. Tracing the bundled code, it only reads a local file and formats it: it
> never spawns a process and never opens a socket. Both grants are pure headroom.
> Separately, the code reads an environment secret it never declared needing.
> **Confirmed** over-broad grant plus undeclared read, `high`, remediation = drop
> the process and network grants, declare and justify the secret read or remove it,
> and scope every remaining grant to the task.
>
> **Kill.** A skill declares read access to one directory and network access to one
> named host. Tracing the code, it reads only that directory and calls only that
> host, both needed for its stated function, nothing wider. Declared, used, and
> already scoped. **Killed**, `kill_reason` = "each grant is exercised at the
> declared scope and no narrower scope covers the task."

## Rationalizations to reject

- *"It might need those permissions later."* → Then it declares them later. Grant
  what the shipped code uses, not what a future version might.
- *"The extra scope is harmless."* → It is dormant blast radius a compromise
  inherits. Harmless until it is not.
- *"It only reads one secret it did not declare."* → Undeclared reach is a broken
  consent boundary regardless of size. Declare it or remove it.
- *"The manifest came from a template."* → Then it was never sized to this skill.
  That is the reason to audit it, not to trust it.

## Executing this in practice

You need the declared grant surface from the frontmatter or manifest, a trace of
what the bundled code and instructions actually exercise (calls, processes,
sockets, file reads), and a way to diff the two sets in both directions. A view
that follows calls through aliases and indirection beats a string grep, because an
undeclared capability is easiest to hide behind one. The two-way diff and the
least-privilege rewrite are the method.

## Related

- `vetting-skills-before-install` - the flagship that runs this permission audit
  as one step before an install verdict.
- `auditing-ai-agent-permissions` - the runtime least-privilege view of an agent's
  own tools; this skill is the install-time view of a packaged skill.
- `auditing-skill-and-mcp-instructions` - the instruction-text lint that pairs with
  this permission diff to vet an artifact.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the declared grant, sink =
  the capability the code exercises or fails to declare.
