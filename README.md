# redteam-agent-skills

Agent skills that encode security-testing *methodology*: the reasoning, ordering,
and adjudication discipline behind real black-box and white-box testing, written
as portable [Claude Code skills](https://docs.claude.com/en/docs/claude-code/skills).

The bet is that a skill's value is judgment, not a script. Enumerate the whole
taxonomy before you look at one family. Rank is triage, not a filter. A lead is a
fact, never a verdict. That reasoning transfers to anyone, whatever tools they
run, so every skill here is tool-agnostic method. Each one names the *capability*
a step needs (for example, "something that answers who calls this from a real
parse") without prescribing a product. Bring your own.

## Scope and ethics

These skills are for **authorized** security work only: your own code, OSS you
contribute to, CTFs, and engagements where testing is in scope. Every skill opens
with a scope check or gate. Nothing here targets systems you do not have
permission to test.

## Install

### As a plugin (recommended)

Installs all skills at once and keeps them updatable. In Claude Code:

```
/plugin marketplace add UnboundCompute/redteam-agent-skills
/plugin install redteam-agent-skills@unboundcompute
```

The skills are then available to Claude automatically, matched by their name and
description. Update later with `/plugin marketplace update unboundcompute`.

### One skill by hand

Copy any single skill directory into your skills path:

```
# project-local (checked in with your repo)
cp -r skills/detecting-race-conditions .claude/skills/

# or personal (available in every project)
cp -r skills/detecting-race-conditions ~/.claude/skills/
```

### Any other agent

The `SKILL.md` bodies are standalone playbooks. Point any agent runtime that
follows the [Agent Skills](https://agentskills.io) format at the `skills/`
directory, or read a `SKILL.md` and follow the loop by hand, running your own
tools at each step and emitting findings in the shared schema.

## The skills

| Skill | Lane | What it does |
|-------|------|--------------|
| [hunting-bugs-with-a-code-graph](skills/hunting-bugs-with-a-code-graph) | white-box | Master loop: orient, enumerate the whole taxonomy, adjudicate |
| [adjudicating-taint-paths](skills/adjudicating-taint-paths) | white-box | Turn a source-to-sink lead into a confirmed finding or a documented kill |
| [auditing-guard-gaps](skills/auditing-guard-gaps) | white-box | Find the unguarded peer of a guarded function |
| [detecting-memory-safety-bugs](skills/detecting-memory-safety-bugs) | white-box | UAF, double-free, OOB, uninitialized, NULL deref |
| [detecting-race-conditions](skills/detecting-race-conditions) | white-box | TOCTOU, check-then-act, atomicity, lock misuse |
| [hunting-business-logic-flaws](skills/hunting-business-logic-flaws) | white-box | Step-skip, limit-overrun, replay, and paywall bypass the scanners miss |
| [reviewing-ai-generated-code](skills/reviewing-ai-generated-code) | white-box | Hallucinated imports, insecure defaults, and claim-versus-behavior gaps in model-written code |
| [hunting-bug-variants](skills/hunting-bug-variants) | research | One confirmed bug to its siblings and the paths a fix missed |
| [extracting-nday-from-a-patch](skills/extracting-nday-from-a-patch) | research | A fix or version diff to the bug it fixed and the variants it left |
| [adjudicating-dependency-cve-reachability](skills/adjudicating-dependency-cve-reachability) | research | Is a dependency CVE actually reachable from your code, or noise |
| [hunting-supply-chain-risks](skills/hunting-supply-chain-risks) | research | Dependency confusion, slopsquatting, poisoned pipeline execution, CI privilege |
| [auditing-the-lethal-trifecta](skills/auditing-the-lethal-trifecta) | agent | Where private data, untrusted content, and exfiltration meet in one agent context |
| [testing-agents-for-indirect-prompt-injection](skills/testing-agents-for-indirect-prompt-injection) | agent | Does the agent obey instructions hidden in content it ingests |
| [auditing-mcp-tool-integrations](skills/auditing-mcp-tool-integrations) | agent | Tool poisoning, shadowing, rug-pulls, token passthrough, output injection |
| [red-teaming-multi-agent-systems](skills/red-teaming-multi-agent-systems) | agent | Agent-to-agent injection, delegation loops, confused deputy, distributed trifecta |
| [auditing-ai-agent-permissions](skills/auditing-ai-agent-permissions) | agent | Excessive agency, missing approval gates, egress, denial-of-wallet |
| [testing-rag-and-memory-poisoning](skills/testing-rag-and-memory-poisoning) | agent | Poisoned index, memory, and search results that fire on an innocent query |
| [testing-llm-insecure-output-handling](skills/testing-llm-insecure-output-handling) | agent | Model output to XSS, markdown-image exfil, terminal escapes, smuggled unicode |
| [auditing-ml-model-supply-chain](skills/auditing-ml-model-supply-chain) | agent | Deserialization RCE on model load, poisoned weights, hub name confusion |
| [evaluating-model-guardrails](skills/evaluating-model-guardrails) | agent | Policy-grounded, reproducible bypass-rate testing of safety guardrails |
| [mapping-attack-surface](skills/mapping-attack-surface) | black-box | Authorized recon to a prioritized surface inventory |
| [testing-web-cache-attacks](skills/testing-web-cache-attacks) | black-box | Cache poisoning and cache deception through the key-versus-response gap |
| [writing-vuln-reports](skills/writing-vuln-reports) | reporting | Confirmed finding to a reproducible writeup |

Every finding, from any skill, is emitted in the shared
[finding schema](FINDING-SCHEMA.md), so results are consistent, deduplicable, and
ready for `writing-vuln-reports` without reformatting.

Each skill is a directory with a `SKILL.md` (YAML frontmatter plus body). The
white-box skills assume you have *some* way to answer structural questions from a
real parse: a code property graph, a static analyzer, or careful manual tracing
on a small target. The method does not depend on which.

## Roadmap

- [x] White-box: code-graph bug hunting, taint adjudication, guard-gap audit,
      memory safety, race conditions
- [x] Reporting: finding to reproducible writeup
- [x] Black-box: attack-surface recon and triage
- [x] Research: bug-variant hunting, n-day from a patch, dependency-CVE
      reachability
- [x] AI-agent red-teaming: the lethal trifecta, indirect prompt injection,
      MCP tool-integration abuse, multi-agent and delegation abuse, agent
      permissions and least-privilege, RAG and memory poisoning, insecure
      output handling
- [x] AI/ML security depth: untrusted model supply chain, guardrail evaluation,
      AI-generated-code review
- [x] Supply chain: dependency confusion, slopsquatting, poisoned pipeline
      execution, CI privilege abuse
- [x] Business-logic depth: step-skip, limit-overrun, replay, and paywall bypass
- [x] Web depth: cache poisoning and cache deception
- [ ] More access-control depth: fail-open access control, framework
      authorization-hook bypasses
- [ ] Appsec depth: crypto misuse, federation flows, request smuggling,
      client-side DOM attacks

## Design

These are built on the patterns that make skills reliably trigger and get used:
gerund names, a `description` written as a routing rule (what plus when plus
trigger terms, third person), a lean body with progressive disclosure, concrete
worked examples, and a "rationalizations to reject" section per skill. See
[CONTRIBUTING.md](CONTRIBUTING.md) for the authoring checklist. The structure is
informed by Anthropic's Agent Skills guidance and the conventions of existing
open-source security skill libraries.

## Contributing

Contributions are welcome. New skills lead with tool-agnostic method, keep the
scope check, name capabilities rather than products, and emit the shared finding
schema. Read [CONTRIBUTING.md](CONTRIBUTING.md) before opening a PR, and file an
issue first if you want to discuss a new skill or a larger change.

## License

MIT. See [LICENSE](LICENSE).
