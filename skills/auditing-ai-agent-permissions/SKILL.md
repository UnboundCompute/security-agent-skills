---
name: auditing-ai-agent-permissions
description: >-
  Audit what an AI agent is actually allowed to do versus what its task needs.
  Covers excessive agency (tools, scopes, and autonomy beyond the job), missing
  human-in-the-loop gates on irreversible actions, over-broad credentials and
  their blast radius, sandbox and code-interpreter escape, unfiltered egress, and
  unbounded resource or spend (denial-of-wallet). Use when granting an agent a tool
  or scope, reviewing an agent's permission posture, or deciding which actions need
  approval. The model's restraint is not a control; permissions are.
license: MIT
---

# Auditing AI agent permissions: agency is what's left when the prompt defense fails

Prompt-level defenses are probabilistic and bypassable. What remains after an
injection succeeds is what the agent is permitted to do, so the durable control is
the permission set, not the model's judgment. Auditing agency means comparing every
capability the agent holds against what its task actually requires, and gating the
actions that cannot be undone.

## When to use

- You are granting an agent a new tool, scope, credential, or autonomous action.
- You are reviewing an agent's permission and egress posture.
- You are deciding which actions require human approval and which can run freely.
- You are scoping a code interpreter or shell an agent can drive.

## Scope check

Audit agents and systems you own or are authorized to test. Do not exercise
destructive or irreversible actions against systems you do not control. If you
can't name the authorization, stop.

## The loop

1. **Diff granted capability against required capability.** List every tool, scope,
   credential, and autonomous action the agent has. Beside each, write what the
   task actually needs. The gap is excessive agency: a summarizer with delete
   rights, a read task holding a write token, a support bot that can issue uncapped
   refunds.

2. **Classify actions by reversibility and blast radius.** Mark each action
   reversible or irreversible, low or high impact. Irreversible or high-impact
   actions (deleting data, sending money or messages externally, changing access,
   deploying) are the set that needs a gate, no matter how aligned the model seems.

3. **Check human-in-the-loop on the dangerous set.** For each irreversible or
   high-impact action, is there an approval gate, or does the agent execute alone? A
   gate the agent can auto-approve, pre-approve, or that fires after the effect does
   not count. The test is whether a human authorizes before the irreversible step.

4. **Test credential blast radius.** Does a credential grant more than the tool
   needs: a broad API key, an admin role, a token valid for other systems? If the
   agent is compromised through injection, its credentials are the blast radius.
   Scope each token to the minimum the specific tool requires.

5. **Test the sandbox and egress boundary.** If the agent runs code or shell, can
   that code reach the network, the host filesystem, other tenants, or the agent's
   own credentials and metadata endpoint? Can any tool send data to an
   attacker-observable destination? Unfiltered egress plus untrusted input is an
   exfiltration channel.

6. **Test resource and spend bounds.** Can a single task drive unbounded tool calls,
   model calls, or paid API usage? Without per-task caps, adversarial or injected
   input turns the agent into a cost-amplification weapon. Confirm caps on calls,
   spend, and wall-clock time (denial-of-wallet).

7. **Record.** Confirm or kill each over-grant in the schema with the minimal fix:
   drop the tool, scope the token, add the approval gate, filter egress, cap the
   budget.

## Reading agency honestly

- **Default-grant is the anti-pattern.** Agents accrete tools, and each is permanent
  authority until removed. Grant per task, revoke on completion.
- **"The model won't misuse it" is not a control.** Injection makes the model act
  for the attacker; the permission set is what constrains the damage.
- **Reversibility is the axis that matters.** Gate the irreversible, let the
  reversible run. Do not gate everything and train users to click through.
- **Credentials plus egress plus untrusted input is the trifecta.** This is the same
  exposure viewed from the permission side.

## Worked example (a confirm and a kill)

> **Confirm.** A support agent holds a token whose role can issue refunds of any
> amount and read the full user table, though its task is answering FAQs and issuing
> refunds up to a set cap. No approval gate on refunds; the token also reads
> unrelated data. An injected ticket drives a max-amount refund. **Confirmed**
> excessive agency, `high`, remediation = scope the token to FAQ plus capped refund,
> gate refunds above the cap on approval, remove the user-table read.
>
> **Kill.** A documentation agent has one tool: read-only search over a public docs
> corpus, no credentials, no writes, no network egress beyond the search backend, a
> per-task call cap in place. Even fully injected, it can only return public text.
> **Killed**, `kill_reason` = "read-only scope over public data, no credentials, no
> egress, capped; no irreversible or high-impact action to gate."

## Rationalizations to reject

- *"It's convenient to give it broad access now."* → Convenience serves the attacker
  too. Grant per task, not per maybe.
- *"There's a confirmation dialog."* → If the agent can auto-confirm or it fires
  after the effect, it is not a gate.
- *"The sandbox is isolated."* → Test the egress and the metadata endpoint;
  isolation is a claim until you prove it.
- *"No one would spend that much through it."* → Denial-of-wallet needs no one, it
  needs a loop. Cap it.

## Executing this in practice

You need the agent's true grant set (tools, scopes, credentials, autonomy), the
reversibility and impact of each action, the sandbox and egress configuration, and
the resource caps. Any harness that lists the manifest and logs actions works; the
required-versus-granted diff and the reversibility gate are the method.

## Related

- `auditing-the-lethal-trifecta` - the egress and credential legs are
  permission-level facts this skill minimizes.
- `red-teaming-multi-agent-systems` - per-agent authority and delegation bounds in a
  system of agents.
- `auditing-mcp-tool-integrations` - the tool layer whose scopes this skill scopes
  down.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the over-broad grant, sink
  = the irreversible or exfiltrating action it enables.
