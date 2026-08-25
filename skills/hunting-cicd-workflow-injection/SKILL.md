---
name: hunting-cicd-workflow-injection
description: >-
  Hunt a CI/CD pipeline for attacker-controlled repository data that reaches a privileged execution
  context, after the trigger and the token scope are resolved. Covers an untrusted event field (an issue
  or pull-request title, a branch name, a commit message) interpolated directly into a run-step shell
  command, a pull_request_target or workflow_run job that checks out and builds the pull-request head with
  secrets in scope, a third-party action pinned to a mutable tag or branch rather than a commit digest, a
  cache a low-trust job writes and a privileged job restores and executes, a self-hosted runner reachable
  from fork pull requests, and an over-scoped pipeline token widening the blast radius. Use when reviewing
  workflow definitions, their triggers, and their token permissions, not the OIDC trust boundary the CI
  OIDC skill owns. Attacker-controlled repository data is the source, a privileged pipeline run step or
  checkout is the sink, and untrusted input reaching execution is the bug.
license: MIT
---

# Hunting CI/CD workflow injection: attacker repository data reaching a privileged run step

A CI/CD pipeline turns dangerous where data an outside contributor controls, an issue title, a branch
name, a pull-request body, the contents of a fork, reaches a step that runs with the repository's secrets
and write token. The two headline shapes are an expression that interpolates an untrusted event field
straight into a shell command, and a privileged trigger that checks out and builds the untrusted
pull-request head. You hunt it by resolving, per workflow, what triggers it (and therefore whether secrets
and a write token are in scope) and then following each untrusted field to the run step or checkout it
reaches. The discipline is separating an untrusted value interpolated into a shell from one passed safely
through an environment variable, and a privileged trigger that runs untrusted code from one that only
reads metadata. The OIDC trust boundary is the CI OIDC skill's; this skill owns data reaching execution.

## When to use

- You are reviewing CI/CD workflow definitions, their triggers, and their token or secret scope.
- A workflow interpolates event data into a run step, or builds a pull request, or uses third-party actions.
- You want to know whether an outside contributor can reach a privileged step through repository data.

## Scope check

Audit only pipelines in repositories you own or are authorized to assess, and trigger a workflow only
against a repository in scope, a proof-of-concept run executes with real secrets and a real token.
Adjudicate on the workflow and its trigger. If you can't name the authorization, stop.

## The loop

1. **Resolve the trigger and the token scope first.** For each workflow, read the trigger and determine
   whether it runs with secrets and a write-scoped token in scope. A plain pull-request trigger runs a
   fork's job without secrets and with a read-only token; a pull_request_target, a workflow_run, or a push
   or issue trigger runs in the base repository's context with secrets. The trigger decides whether an
   injected value has anything worth reaching, so settle it before flagging.

2. **Check untrusted expression interpolation.** Look for an event field an outsider controls (an issue or
   pull-request title or body, a branch or label name, a commit message, an author name) interpolated
   directly into a run-step shell command, so the field's shell metacharacters execute.

3. **Check the privileged-checkout shape.** Look for a pull_request_target or workflow_run job that checks
   out the pull-request head ref and then builds, tests, or runs a script from it, executing untrusted
   code with the base repository's write token and secrets available.

4. **Check action pinning and cache trust.** Look for a third-party action referenced by a mutable tag or a
   branch rather than a full commit digest, so an upstream tag move injects new code, and for a cache a
   low-trust job writes with a key a privileged job restores and executes.

5. **Check runner and token blast radius.** Look for a self-hosted runner reachable from fork pull requests
   (non-ephemeral, carrying persistent state) and for a token scoped wider than the workflow needs (a
   broad write-all permission) that widens the damage any of the above can do.

6. **Confirm and record.** Confirm by setting the untrusted field to a payload (or opening a fork pull
   request) and showing the injected command runs, or the untrusted head executes, with secrets or the
   write token in scope. Kill the lead if the untrusted expression is assigned to an environment variable
   first and the step references the quoted variable (the canonical safe pattern) rather than interpolating
   it into the command string, if the trigger is a plain pull request so the fork job runs without secrets
   and a read-only token (and no self-hosted runner or explicit secret is present), if a pull_request_target
   job never checks out the head ref and only reads metadata (a labeler), if the action is pinned to a full
   commit digest even with a version comment, if the untrusted field is structurally constrained (a numeric
   identifier) and cannot become a shell token, or if repository or organization policy blocks fork
   workflows from running without approval and the runner is ephemeral. Record the workflow, the trigger,
   the untrusted field, and the privileged step it reaches.

## Where pipeline injection leaks

- **The trigger decides whether there is anything to steal.** A plain pull-request job runs without secrets
  and read-only; a target, workflow_run, push, or issue trigger runs with secrets, which is where injection
  pays.
- **Interpolation into the shell is the bug; an env variable is the fix.** An untrusted field spliced into a
  run command executes its metacharacters; the same field passed through a quoted environment variable does
  not.
- **A privileged trigger that builds the head runs untrusted code.** pull_request_target and workflow_run
  give the base-repo token and secrets; checking out and building the fork head hands them to the attacker.
- **A mutable action tag is a supply-chain sink.** A tag or branch reference lets upstream inject new code; a
  full commit digest pins it, and the pinning satisfied kills the finding.
- **Scope amplifies everything.** A self-hosted non-ephemeral runner and a write-all token turn a small
  injection into lateral movement; ephemeral runners and least-scoped tokens bound the blast radius.

## Worked example (a confirm and a kill)

> **Confirm.** A workflow triggered on new issues has a run step `echo "Issue: ${{ github.event.issue.title
> }}"`. An attacker opens an issue titled with a shell payload; the pipeline interpolates it into the
> command and executes it in a context that has repository secrets. **Confirmed** command injection through
> an untrusted event field interpolated into a run step with secrets in scope, `high` rising to `critical`
> with a write token, remediation = pass the title through a quoted environment variable and never
> interpolate untrusted event data into a run command.
>
> **Kill.** The same title flows through `env: TITLE: ${{ github.event.issue.title }}` and the step runs
> `echo "Issue: $TITLE"`, referencing the quoted variable. The value is data to the shell, not command
> text, and the workflow uses the default read-only token. **Killed**, `kill_reason` = "the untrusted value
> is passed through a quoted environment variable rather than interpolated into the command string, so it
> cannot become shell syntax."

## Rationalizations to reject

- *"It just echoes the title."* -> Interpolated into the command with `${{ }}`? Then the title's
  metacharacters run; only a quoted environment variable makes it inert data.
- *"Pull requests can't touch secrets."* -> A plain pull-request trigger cannot, but pull_request_target and
  workflow_run run in the base context with secrets; check which trigger this is.
- *"The action is pinned to v3."* -> A tag is mutable; upstream can move it. Only a full commit digest is a
  pin; a floating tag is a supply-chain sink.
- *"The target workflow only labels."* -> Does it check out the head ref? If it only reads metadata it is
  fine; if it builds the fork head, it runs untrusted code with the token.
- *"The runner is ours."* -> A self-hosted non-ephemeral runner reachable from fork pull requests carries
  state between runs; that persistence is the lateral-movement path.

## Executing this in practice

You need each workflow, its trigger and whether that trigger carries secrets and a write token, every place
an untrusted event field is interpolated into a run step, every privileged-trigger job and whether it
checks out the head ref, the action references and whether they are digest-pinned, the cache keys crossed
between jobs, the runner type, and the token permissions. For each untrusted field, decide whether it
reaches a privileged step as command text or untrusted code. Reading the workflow tells you the path;
triggering it tells you whether the injection runs.

## Related

- `auditing-cicd-oidc-trust` - owns the pipeline's OIDC trust boundary (which cloud identities the workflow
  can assume); this skill owns attacker-controlled data reaching execution, an orthogonal source and sink.
- `hunting-supply-chain-risks` - the broader dependency and artifact-integrity audit; the mutable-action-tag
  shape here is the pipeline-local edge of that supply-chain surface, cross-referenced rather than duplicated.
- `hunting-non-human-identity-and-secret-reachability` - the secrets a compromised pipeline step can reach;
  a confirmed injection here is the execution primitive that makes those secrets reachable.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = attacker-controlled repository data, sink = a
  privileged pipeline run step or checkout, evidence = the trigger scope, the untrusted field, and the
  executed command or untrusted head.
