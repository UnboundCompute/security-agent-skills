---
name: hunting-supply-chain-risks
description: >-
  Hunt for the ways an attacker gets code into your build without touching your
  repo: dependency confusion (a public package shadowing an internal name),
  typosquatting and slopsquatting (a package named after a model's hallucination),
  poisoned pipeline execution (untrusted input running as a build step), and
  over-privileged or injectable CI. Use when reviewing a build pipeline, a
  dependency manifest, an internal package registry, or a CI/CD configuration. The
  app code can be clean while the artifact you ship is not.
license: MIT
---

# Hunting supply-chain risks: the code you didn't write but still ship

Your application code can pass every review while the thing you actually ship is
compromised, because the supply chain is a second, softer surface: what you pull in,
what your build runs, and what your CI is trusted to do. These attacks execute in
your build or your users' installs, usually before any code review looks at them.
Finding them means auditing resolution, execution, and privilege, not the source.

## When to use

- You are reviewing a build pipeline, a dependency manifest, or a lockfile.
- You run or depend on an internal package registry alongside public ones.
- You are auditing a CI/CD configuration, its triggers, and its secrets.

## Scope check

Audit builds, registries, and pipelines you own or are authorized to test. Do not
publish packages or trigger jobs against systems you do not control; probe name
collisions passively. If you can't name the authorization, stop.

## The loop

1. **Map what the build resolves and runs.** List every dependency source
   (registries, internal and public), how names resolve when both exist, and every
   step the pipeline executes that is not your reviewed code: install scripts,
   generated config, third-party actions or plugins, and anything triggered by
   external input.

2. **Check for dependency confusion.** For each internal or private package name,
   can a public registry serve a package of the same name, and would your resolver
   prefer or fall back to it? An internal name that is unclaimed publicly and
   resolvable from the public registry is a hijack waiting for a version bump.
   Confirm scoping, registry pinning, and namespace ownership.

3. **Check for typo- and slopsquatting.** Are dependency names verified against a
   known-good source, or copied from documentation, an assistant's suggestion, or
   memory? A package name a model hallucinated and an attacker then registered
   (slopsquatting) installs attacker code under a plausible name. Verify every name
   resolves to the intended, established package, not a lookalike or a freshly
   published near-name.

4. **Check for poisoned pipeline execution.** Does any CI step run on untrusted input
   (a pull request from a fork, an issue title, a branch name, a dependency's install
   script) while holding secrets or the ability to alter the build? Untrusted input
   that reaches a build step with credentials is remote code execution in your
   pipeline. Trace external triggers to privileged steps.

5. **Check CI privilege and secret blast radius.** Which secrets, tokens, and deploy
   keys does a job hold, and does a low-trust trigger (a fork PR) run with the same
   power as a trusted one? Scope tokens per job and separate untrusted-trigger jobs
   from secret-holding ones.

6. **Check the integrity of what's pulled.** Are dependencies pinned by hash or
   lockfile, are actions and plugins pinned by commit rather than a mutable tag, and
   is what you install verified? A mutable reference is a rug-pull channel: benign
   today, swapped tomorrow.

7. **Confirm or kill each and record.** For each risk, the fix: claim and scope
   internal names and pin the registry, verify package names, isolate
   untrusted-trigger jobs from secrets, pin by hash or commit, minimize token scope.
   Record confirmed exposures and closed paths in the schema.

## Where the chain breaks

- **Resolution order is a security decision.** If public can shadow private, naming
  is authentication, and it is weak.
- **A package name from a model or a doc is unverified input.** Slopsquatting turns a
  confident hallucination into installed code.
- **A fork PR is untrusted content that runs code.** Treat pipeline triggers as an
  ingestion channel.
- **Mutable tags and floating versions are rug-pull surface.** Pin what executes.

## Worked example (a confirm and a kill)

> **Confirm.** An internal utility package is referenced by a bare name, and the
> resolver falls back to the public registry when the private one lacks a requested
> version. The name is unclaimed publicly; an attacker publishes it at a higher
> version. The next build pulls the attacker's code, whose install script runs with
> CI secrets in scope. **Confirmed** dependency confusion into poisoned pipeline,
> `critical`, remediation = claim the name and scope, pin the private registry, and
> run installs without secret access.
>
> **Kill.** A repo pins every dependency by hash in a committed lockfile, resolves
> only from a scoped internal registry with no public fallback, pins all CI actions by
> commit, and runs fork-PR jobs in a no-secrets context. A confusion attempt cannot
> resolve and a swapped tag fails the hash. **Killed**, `kill_reason` = "hash-pinned,
> single scoped registry with no fallback, secret-free untrusted-trigger jobs; no
> shadowing or rug-pull path."

## Rationalizations to reject

- *"The package name looked right."* → Looked right is not verified. Confirm it is
  the established package, not a near-name.
- *"It's an internal package, no one else has it."* → Unclaimed public names are
  claimable. Own the namespace.
- *"It's only a CI script."* → CI holds secrets and ships artifacts. Code execution
  there is production impact.
- *"We use a floating tag for convenience."* → Floating is mutable is rug-pullable.
  Pin by hash or commit.

## Executing this in practice

You need the dependency manifests and lockfiles, the resolver's registry and fallback
configuration, the CI configuration with its triggers and secret scoping, and the
ability to check public registries for name collisions. A code graph over your app
plus resolved dependencies helps confirm whether pulled code is even reachable; the
resolution rules and the trigger-to-secret tracing are the method.

## Related

- `adjudicating-dependency-cve-reachability` - whether a pulled dependency's known
  bug is reachable, once you trust what is pulled.
- `auditing-mcp-tool-integrations` - rug-pull and pinning discipline applied to the
  tool layer.
- `mapping-attack-surface` - inventorying build and CI surface alongside the app.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the shadowing package or
  untrusted trigger, sink = the build step or install that executes it.
