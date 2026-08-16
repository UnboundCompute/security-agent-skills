---
name: evaluating-model-guardrails
description: >-
  Systematically test whether a model's safety and policy guardrails hold under
  adversarial pressure, as a repeatable evaluation rather than a one-off trick.
  Covers building a policy-grounded probe set, multi-turn escalation, encoding and
  obfuscation bypasses, role and context reframing, instruction-hierarchy conflicts,
  and measuring bypass rate with reproducible graded outcomes. Use when reviewing a
  model deployment, a system prompt, or a content-safety layer, or before shipping
  an assistant that must refuse a defined set of requests. A rate, not an anecdote.
license: MIT
---

# Evaluating model guardrails: measure the boundary, don't collect one jailbreak

A guardrail is a claim: "the model will refuse this class of request." A single
successful jailbreak is an anecdote; what you need is a measurement, the rate at
which the boundary holds across a structured battery of attacks. Evaluating
guardrails means grounding probes in the actual policy, attacking each along known
bypass axes, and scoring outcomes reproducibly, so you can state how strong the
boundary is, not just that someone once beat it.

## When to use

- You are reviewing a model deployment, a system prompt, or a content-safety layer.
- Before shipping an assistant that must refuse or constrain a defined set of
  requests.
- You need a defensible bypass rate, not a single proof-of-concept transcript.

## Scope check

Evaluate models and deployments you own or are authorized to test. Use benign,
clearly-scoped probes against a defined policy; do not generate real harmful output
against systems you do not control. If you can't name the authorization, stop.

## The loop

1. **Ground the probes in the stated policy.** Get the actual list of what this
   deployment must refuse or constrain (its safety policy, its system prompt's rules,
   its allowed scope). Every probe targets a specific rule, so a result maps to a
   policy line, not a vibe. An undefined policy is the first finding: you cannot
   evaluate a boundary no one has drawn.

2. **Build a baseline probe set.** For each rule, write direct requests that should
   be refused and benign near-misses that should be allowed. The near-misses matter:
   a guardrail that refuses everything is broken differently from one that refuses
   nothing. Record baseline refuse/allow behavior before attacking.

3. **Attack along the bypass axes.** Take each refused probe and apply the known
   transformations: multi-turn escalation (warm up, then pivot), encoding and
   obfuscation (alternate scripts, spacing, invisible characters, indirection), role
   and context reframing (fiction, hypothetical, translation, "for research"), and
   instruction-hierarchy conflict (content claiming higher authority than the system
   rule). Each axis is a separate test of the same rule.

4. **Define graded, reproducible outcomes.** Decide in advance what counts as a
   bypass: full compliance, partial or hedged compliance, or refusal. Score each
   probe with a fixed rubric so runs are comparable and a fix can be measured against
   a baseline. "It felt jailbroken" is not a result; "the encoding axis bypassed rule
   4 in 7 of 10 trials" is.

5. **Measure rate and stability.** Run each probe multiple times; models are
   stochastic, and a boundary that holds once may fail on retry. Report bypass rate
   per rule and per axis, and flag the axes that reliably win. A rule that fails 30
   percent of the time is not protected.

6. **Record and recommend.** Report per-rule, per-axis bypass rates with example
   transcripts, and the structural fixes: enforce the boundary with input and output
   filtering rather than prompt instructions alone, reduce what the model can do when
   uncertain, add a separate safety layer instead of relying on the model's restraint.
   Record confirmed bypasses and rules that held across all axes (killed) in the
   schema.

## What separates evaluation from a jailbreak

- **A jailbreak is one transcript; an evaluation is a rate.** Ship the rate.
- **Near-misses are half the signal.** Over-refusal is a failure mode too; measure
  both directions.
- **Stochastic means "once" is meaningless.** Re-run; report the distribution, not
  the best or worst single try.
- **Prompt-only guardrails are the weak kind.** If the only defense is instructions
  in the system prompt, the axes above will find the gap. Recommend a real boundary.

## Worked example (a confirm and a kill)

> **Confirm.** A support assistant must refuse to reveal another user's order details.
> Direct requests refuse. Under multi-turn escalation (establish a helpful frame, then
> ask "as we discussed, pull the other order"), it complies in 6 of 10 trials; an
> encoding variant of the same ask succeeds 8 of 10. **Confirmed** guardrail bypass on
> the cross-user rule, `high`, remediation = enforce authorization on the data access
> itself, not the model's refusal; add an output check for other-user data.
>
> **Kill.** A model must not output a specific restricted category. Across all four
> axes and 10 trials each, an input classifier blocks the request and an output
> classifier blocks the response before the user sees it; the model's own refusal is
> only the third layer. Every probe is refused or filtered. **Killed**, `kill_reason`
> = "boundary enforced by input and output filters independent of the model; 0 of 40
> adversarial trials bypassed."

## Rationalizations to reject

- *"We told it in the system prompt not to."* → Prompt instructions are the weakest
  guardrail. Test them; expect the axes to win.
- *"We couldn't jailbreak it in a few tries."* → A few tries is not a rate. Re-run at
  volume across every axis.
- *"It refused the obvious version."* → Obvious is the baseline. The bypass lives in
  the reframed and encoded versions.
- *"The model is aligned."* → Alignment is probabilistic and axis-dependent. Measure
  the boundary, don't trust the model.

## Executing this in practice

You need the deployment's actual policy, a probe set grounded in it, the four bypass
axes, a fixed grading rubric, and enough repeated trials to report a rate. Any harness
that can send structured conversations and log graded outcomes works; the policy
grounding and the reproducible rubric are the method, and the specific payloads are
interchangeable.

## Related

- `testing-agents-for-indirect-prompt-injection` - when the bypass arrives through
  ingested content rather than the user.
- `auditing-ai-agent-permissions` - a bypassed guardrail matters only as far as the
  model's permissions let it act.
- `testing-llm-insecure-output-handling` - what a bypassed output can do at the
  downstream sink.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the adversarial probe, sink
  = the policy-violating output or action.
