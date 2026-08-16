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
| [hunting-bug-variants](skills/hunting-bug-variants) | research | One confirmed bug to its siblings and the paths a fix missed |
| [extracting-nday-from-a-patch](skills/extracting-nday-from-a-patch) | research | A fix or version diff to the bug it fixed and the variants it left |
| [adjudicating-dependency-cve-reachability](skills/adjudicating-dependency-cve-reachability) | research | Is a dependency CVE actually reachable from your code, or noise |
| [mapping-attack-surface](skills/mapping-attack-surface) | black-box | Authorized recon to a prioritized surface inventory |
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
- [ ] AI-agent red-teaming: the lethal trifecta, indirect prompt injection,
      MCP tool-integration abuse, multi-agent and excessive-agency failures
- [ ] Business-logic and access-control depth: logic-flaw hunting, fail-open
      access control, framework authorization-hook bypasses

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
