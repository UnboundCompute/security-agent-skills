# redteam-agent-skills

Agent skills that encode security-testing *methodology* — the reasoning,
ordering, and adjudication discipline behind real black-box and white-box
testing — as portable [Claude Code skills](https://docs.claude.com/en/docs/claude-code/skills).

The bet: a skill's value is not a script, it's judgment. *Enumerate the whole
taxonomy before you look at one family. Rank is triage, not a filter. A
candidate is a fact, never a verdict.* That transfers to anyone, whatever tools
they run — so every skill here is tool-agnostic method. Each one names the
*capability* a step needs ("something that answers who calls this from a real
parse") without prescribing a product; bring your own.

## Scope & ethics

These skills are for **authorized** security work only: your own code, OSS you
contribute to, CTFs, and engagements where testing is in scope. Every skill
opens with a scope check. Nothing here targets systems you don't have permission
to test.

## Layout

```
whitebox/    source-available analysis over a code property graph
  code-graph-bughunt/    master workflow: orient, enumerate taxonomy, adjudicate
  taint-adjudication/    turn a source→sink lead into a confirmed finding or kill it
  guard-sibling-audit/   find the unguarded peer of a guarded function
blackbox/    (coming) live-target web pentest & bug-bounty playbooks
```

Each skill is a directory with a `SKILL.md` (YAML frontmatter + body). The
whitebox skills assume you have *some* way to answer structural questions from a
real parse — a code property graph, a static analyzer, or careful manual tracing
on a small target. The method doesn't depend on which.

## Using a skill

**Claude Code** — drop a skill directory into `.claude/skills/` in your project
(or `~/.claude/skills/` globally); it's picked up by name and description.

**Any agent / by hand** — the `SKILL.md` bodies are readable as standalone
playbooks. Read the method section, run your own tools against each step.

## Roadmap

- [x] Whitebox: code-graph bug hunting, taint adjudication, guard/sibling audit
- [ ] Blackbox: recon & triage, IDOR/BOLA, SSRF, auth-bypass/BFLA, OAuth/JWT,
      request smuggling, GraphQL, deserialization
- [ ] Reporting: finding → reproducible submission

## Contributing

See [CONTRIBUTING.md](./CONTRIBUTING.md). New skills should lead with
tool-agnostic method, keep the scope check, and separate reference-implementation
tool calls from the reasoning.

## License

MIT — see [LICENSE](./LICENSE).
