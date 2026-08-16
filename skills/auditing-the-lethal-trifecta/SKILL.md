---
name: auditing-the-lethal-trifecta
description: >-
  Find where an AI agent becomes dangerous: the trust context in which access to
  private data, exposure to untrusted content, and an ability to send data out
  all coexist. Any two legs are usually safe; all three let planted content make
  the agent read secrets and exfiltrate them. Use when designing or reviewing a
  tool-using LLM agent, before granting it a new tool or data scope, or to judge
  whether a prompt injection is actually exploitable. Covers capability
  inventory, the three legs, kill-chain construction, and which leg to cut.
license: MIT
---

# Auditing the lethal trifecta: the agent exposure condition

An AI agent turns dangerous when three capabilities share one trust context:

- **Private data.** It can read secrets, user data, internal systems.
- **Untrusted content.** It ingests text an attacker can influence.
- **Exfiltration.** It can send data somewhere the attacker can observe.

Any two of these are usually safe. All three together mean an attacker who plants
content can make the agent read the private data and route it out, no account
takeover required. The trifecta is a *structural* property of the agent's wiring,
which is why it survives prompt-level defenses that a specific payload would slip.

## When to use

- You are designing or reviewing a tool-using agent, assistant, or automation.
- Before granting the agent a new tool, data scope, or network capability.
- You found a prompt injection and need to know if it can cause real damage.
- You are writing or reviewing an agent's permission and egress posture.

## Scope check

Audit agents and systems you own or are authorized to test. Do not plant content
in or exfiltrate from systems you do not control. If you can't name the
authorization, stop.

## The loop

1. **Fix the trust context.** A context is one session, task, or conversation
   where content and capabilities mix, sharing the same model instance and memory.
   The trifecta must co-occur *within a single context* to bite; audit context by
   context, not tool by tool.

2. **Inventory capabilities.** List everything the agent can read and everything
   it can do in this context: data stores, credentials, files, internal APIs,
   network calls, message/PR/email sends, tool invocations, memory writes.

3. **Label each against the three legs.** Mark every capability as private-data
   (leg A), untrusted-content intake (leg B), or exfiltration/egress (leg C). Be
   honest about indirect legs: a tool that writes to a shared doc an attacker can
   read is egress; a "read-only" web fetch is untrusted-content intake.

4. **Find the overlap.** A context carrying all three legs is a live exposure.
   Two legs is a latent risk to watch (one added tool completes it). Record which
   legs each context has.

5. **Build the kill chain.** For each trifecta context, write the concrete path:
   attacker plants content in leg B, the agent ingests it, the injected
   instruction drives a leg-A read, the result leaves through leg C. If you can
   trace that chain, the exposure is confirmed; if a leg is missing or isolated,
   it is killed.

6. **Cut the cheapest leg and record.** The fix is to remove one leg from the
   context: isolate untrusted content behind a data-only boundary, drop or gate
   egress, scope down data access, or require human approval on the sensitive
   action. Recommend the leg that costs the least function. Record confirmed
   exposures and killed contexts (naming the absent leg) in the schema.

## Reading the legs honestly

- **Egress hides in benign tools.** Posting a comment, opening a PR, writing a
  file to a synced folder, a "helpful" outbound webhook, even a rendered image URL
  the agent controls, are all channels an attacker can read. Enumerate them.
- **Reading is enough.** Leg A does not require write access. Read plus egress is
  the whole breach.
- **Untrusted content is broader than "the web."** Retrieved documents, tickets,
  emails, filenames, tool outputs, and other agents' messages are all leg B.
- **Two legs is a design smell.** If a context has two, treat the third as one PR
  away and design the boundary now.

## Worked example (a confirm and a kill)

> **Confirm.** A coding agent has repo and `.env` access (A), reads issues and PR
> comments from a public repo (B), and can make outbound network requests and open
> PRs (C). An attacker files an issue containing an injection; the agent ingests
> it, reads `.env`, and includes the secret in an outbound request. All three legs
> in one context, chain traced. **Confirmed**, `critical`, remediation = remove
> egress from the issue-triage context or gate outbound calls behind approval.
>
> **Kill.** A summarizer agent fetches untrusted web pages (B) but has no access
> to private data (no A) and only returns text to the user in-band (no external C).
> **Killed**, `kill_reason` = "two legs absent; injection can mislead the summary
> but cannot read secrets or exfiltrate."

## Rationalizations to reject

- *"We have a prompt-injection filter."* → Filters are payload-level and
  bypassable. The trifecta is structural; cut a leg.
- *"The data access is read-only."* → Read plus egress is the breach. Read-only
  does not remove leg A.
- *"Each tool is safe on its own."* → True and irrelevant. The exposure is the
  combination in one context.
- *"No one would plant content there."* → Leg B means someone can. That is the
  definition.

## Executing this in practice

You need the agent's real capability set for the context under review: its tool
manifest, the data and credentials it can reach, every content source that feeds
its context, and its full egress posture (network, writes, messages). Map those to
the three legs; the overlap is the finding. The tooling only enumerates
capabilities, the trifecta judgment and the kill chain are yours.

## Related

- `testing-agents-for-indirect-prompt-injection` - the mechanism that drives the
  kill chain through leg B.
- `auditing-mcp-tool-integrations` - where token passthrough and tool egress add
  leg C, and tool metadata adds leg B.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - the shape every confirmed and
  killed context takes (source = untrusted channel, sink = the egress action).
