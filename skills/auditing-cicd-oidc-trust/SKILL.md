---
name: auditing-cicd-oidc-trust
description: >-
  Audit continuous-integration pipelines for the trust they extend to untrusted
  input: workflows that run on incoming change requests from forks while holding
  repository secrets, steps that let attacker-controlled content reach a privileged
  command, and cloud role trust conditions that accept a pipeline's short-lived
  token too broadly. Covers secret and token exposure on fork-triggered runs,
  poisoned-pipeline execution, and over-broad trust on the identity claim a pipeline
  presents to a cloud account. Use when reviewing CI/CD configuration, pipeline
  identity, or the boundary between a build and the cloud it can reach. An
  exploitable token or command from untrusted input is the finding.
license: MIT
---

# Auditing CI/CD OIDC trust: the pipeline is an identity, treat its inputs as hostile

A modern pipeline is a privileged identity. It holds secrets, and it presents a
short-lived token that a cloud account trusts and exchanges for real credentials.
The danger is that the same pipeline also runs code and content that outsiders can
influence: a change request from a fork, a dependency, a script in the repository.
When untrusted input reaches a privileged step, or when the cloud trusts the
pipeline's token too broadly, an outsider borrows the pipeline's identity. The audit
is about where untrusted input meets pipeline privilege.

## When to use

- You are reviewing CI/CD workflows, their triggers, and their secret and token access.
- A pipeline exchanges a short-lived identity token for cloud credentials.
- Any workflow runs on input from forks or external contributors.

## Scope check

Review pipeline configuration in repositories and accounts you own or are authorized
to test. If you can't name the authorization, stop.

## The loop

1. **Inventory triggers and what each run can reach.** For every workflow, record
   what event starts it, whether that event can be caused by an outside contributor,
   and what the run holds: repository secrets, an identity token, write permission to
   the repository or a registry. A run that both accepts outside input and holds
   privilege is the shape to chase.

2. **Find fork-triggered runs that keep secrets.** A workflow that runs on an
   incoming change from a fork and still receives repository secrets or a privileged
   token hands those to code the outsider proposed. Confirm which triggers expose
   secrets on untrusted content, rather than assuming the default is safe.

3. **Trace untrusted input to a privileged step.** Follow attacker-influenced
   content, a change-request title, a branch name, a file in the fork, a dependency
   script, into any step that runs a command, publishes an artifact, or requests
   cloud credentials. Content that reaches a shell or a publish step is
   poisoned-pipeline execution.

4. **Read the cloud trust condition on the pipeline token.** For each role the
   pipeline can assume, read the condition on the token it presents. Confirm the
   condition pins the specific repository and branch or environment, and the
   expected token audience. A condition that accepts any repository in the
   organization, any branch, or omits the audience trusts too much.

5. **Check what the borrowed identity can do.** If untrusted input can reach the
   token, or the trust is too broad, determine what the exchanged cloud credentials
   grant. A tightly scoped deployment role limits the damage; a broad role turns a
   pipeline foothold into account access. This links straight to identity-graph
   escalation.

6. **Confirm and record.** Confirm by driving untrusted input through the path in an
   authorized repository and observing a secret read, a credential minted, or a
   command run; or by showing the trust condition accepts a token it should not. Kill
   the lead if fork runs carry no secrets, untrusted input never reaches a privileged
   step, and every trust condition pins repository, branch, and audience. Record the
   trigger, the path, and the privilege reached.

## Where pipeline trust leaks

- **Fork-triggered runs with secrets are the classic exposure.** Untrusted content
  should never run in a context that holds the pipeline's privilege.
- **Content is code.** A title, a branch name, or a filename that reaches a shell is
  an execution path, not just data.
- **The trust condition is the real access-control.** A token is only as safe as the
  condition that decides which token a cloud account will exchange.
- **A pipeline foothold is worth what its role grants.** Scope the assumed role, or a
  build compromise becomes an account compromise.

## Worked example (a confirm and a kill)

> **Confirm.** A workflow runs on incoming change requests from forks so it can post
> a preview, and that trigger receives repository secrets. A contributor opens a
> change whose build script reads the secrets and sends them to an external address.
> The run executes the fork's script with secrets present. **Confirmed** secret exfil
> on fork-triggered run, `critical`, remediation = run untrusted changes in a trigger
> that carries no secrets, and move any privileged step to a separate run gated on a
> trusted review.
>
> **Kill.** Fork changes run only in a trigger with no secrets and no token, the
> privileged deploy runs on a trusted branch push, untrusted input is never
> interpolated into a shell, and the deploy role's trust condition pins the exact
> repository, branch, and token audience. Every attempt to reach a secret or mint a
> credential from a fork dead-ends. **Killed**, `kill_reason` = "no secrets on
> untrusted trigger, no untrusted input reaches a privileged step, trust condition
> pins repository, branch, and audience."

## Rationalizations to reject

- *"Only maintainers can merge."* → Exposure is at run time, not merge time. A
  fork-triggered run executes before anyone merges.
- *"It is just a preview build."* → A preview build that holds secrets or a token is
  a privileged build. What it holds is what leaks.
- *"The token is short-lived."* → Lifetime does not scope power. A broad trust
  condition exchanges that short-lived token for real credentials on demand.
- *"We pin the repository in the trust."* → And the branch and the audience? A
  condition missing either accepts a token from a context you did not intend.

## Executing this in practice

You need every workflow definition, its trigger semantics and secret and token
access, the data flow from untrusted input to privileged steps, and the cloud trust
conditions on each assumable role. A view that connects a trigger to the secrets it
exposes and traces untrusted content to a command or credential request is ideal;
driving the input through an authorized pipeline is the confirmation.

## Related

- `hunting-iam-privilege-escalation-paths` - what the borrowed pipeline identity can
  reach once its token is exchanged for cloud credentials.
- `hunting-supply-chain-risks` - poisoned dependencies and pipeline execution as a
  broader class; this is the identity and trust-boundary slice.
- `finding-fail-open-flaws` - a trust condition that accepts too much is the
  fail-open shape applied to pipeline identity.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = untrusted input to the
  pipeline or a token the trust wrongly accepts, sink = the secret, credential, or
  command it reaches.
