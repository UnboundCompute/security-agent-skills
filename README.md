# redteam-agent-skills

Agent skills that encode security-testing *methodology* — the reasoning,
ordering, and adjudication discipline behind real black-box and white-box
testing — as portable [Claude Code skills](https://docs.claude.com/en/docs/claude-code/skills).

The bet: a skill's value is not a script, it's judgment. *Enumerate the whole
taxonomy before you look at one family. Rank is triage, not a filter. A lead is a
fact, never a verdict.* That transfers to anyone, whatever tools they run — so
every skill here is tool-agnostic method. Each one names the *capability* a step
needs ("something that answers who calls this from a real parse") without
prescribing a product; bring your own.

## Scope & ethics

These skills are for **authorized** security work only: your own code, OSS you
contribute to, CTFs, and engagements where testing is in scope. Every skill opens
with a scope check or gate. Nothing here targets systems you don't have
permission to test.

## Layout

```
whitebox/    source-available analysis over code structure (call graph + dataflow)
  hunting-bugs-with-a-code-graph/   master loop: orient, enumerate taxonomy, adjudicate
  adjudicating-taint-paths/         source→sink lead → confirmed finding or a kill
  auditing-guard-gaps/              the unguarded peer of a guarded function
  detecting-memory-safety-bugs/     UAF, double-free, OOB, uninit, NULL deref
  detecting-race-conditions/        TOCTOU, check-then-act, atomicity, lock misuse
blackbox/    live-target testing with no source
  mapping-attack-surface/           authorized recon → prioritized surface inventory
reporting/
  writing-vuln-reports/             confirmed finding → reproducible writeup
FINDING-SCHEMA.md                   the one shape every finding takes
```

Every finding — from any skill — is emitted in the shared
[finding schema](./FINDING-SCHEMA.md), so results are consistent, deduplicable,
and ready for `writing-vuln-reports` without reformatting.

Each skill is a directory with a `SKILL.md` (YAML frontmatter + body). The
whitebox skills assume you have *some* way to answer structural questions from a
real parse — a code property graph, a static analyzer, or careful manual tracing
on a small target. The method doesn't depend on which.

## Using a skill

**Claude Code** — drop a skill directory into `.claude/skills/` in your project
(or `~/.claude/skills/` globally); it's picked up by its `name` and `description`.

**Any agent / by hand** — the `SKILL.md` bodies read as standalone playbooks.
Follow the loop, run your own tools at each step, emit findings in the schema.

## Roadmap

- [x] Whitebox: code-graph bug hunting, taint adjudication, guard-gap audit,
      memory-safety, race conditions
- [x] Reporting: finding → reproducible writeup
- [x] Blackbox: attack-surface recon & triage
- [ ] Blackbox per-class: IDOR/BOLA, auth-bypass/BFLA, SSRF, open redirect,
      file upload, injection, OAuth/JWT, request smuggling, GraphQL,
      deserialization
- [ ] A meta "distill-a-skill" that turns a writeup/CVE into a new SKILL.md

## Design

Built on the patterns that make skills reliably trigger and get used: gerund
names, a `description` written as a routing rule (what + when + trigger terms,
third person), a lean body with progressive disclosure, concrete worked
examples, and a "rationalizations to reject" section per skill. See
[CONTRIBUTING.md](./CONTRIBUTING.md) for the authoring checklist. Structure
informed by Anthropic's Agent Skills best-practices and the conventions of
existing open-source security skill libraries.

## Contributing

See [CONTRIBUTING.md](./CONTRIBUTING.md). New skills lead with tool-agnostic
method, keep the scope check, name capabilities rather than products, and emit
the shared finding schema.

## License

MIT — see [LICENSE](./LICENSE).
