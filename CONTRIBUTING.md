# Contributing

Thanks for adding to the skill library. A few conventions keep these skills
useful to people who don't run our stack.

## Skill shape

Each skill is a directory under `whitebox/`, `blackbox/`, or `reporting/`
containing a `SKILL.md`:

```markdown
---
name: gerund-kebab-name        # e.g. detecting-race-conditions
description: >-                 # third person; what + WHEN + trigger terms; ≤1024 chars
  What the skill does AND when to reach for it, naming the trigger conditions
  explicitly. This is the routing rule an agent matches against — if it doesn't
  describe how a user phrases the need, the skill never loads.
license: MIT
---

# Title

Body: the method (see structure below).
```

The `name` must equal the directory name, be **gerund + kebab-case**
(`detecting-…`, `auditing-…`, `hunting-…`), specific not generic (no
`helper`/`utils`/`tools`), and must not contain `claude`/`anthropic`.

Keep `SKILL.md` under ~500 lines. Push long payload lists, checklists, or
extended walkthroughs into a `references/` subdirectory and link them **one level
deep** from `SKILL.md` (never chain `SKILL.md → a.md → b.md`).

## The four rules

1. **Scope check first.** Every skill opens by stating it is for authorized
   testing only. No exceptions.
2. **Method before tools.** The body is tool-agnostic reasoning — the order of
   operations, what to check, how to decide. Someone with different tools should
   still be able to follow it.
3. **Name capabilities, not products.** Describe the *capability* a step needs
   ("something that answers who calls this from a real parse"), never a specific
   tool, product, or internal system. Do not name or describe proprietary
   tooling or an internal pipeline — the skill must stand on the method alone.
4. **Adjudication discipline.** A lead/candidate is a fact, never a verdict.
   Rank is triage, never a filter. Enumerate the whole taxonomy before working
   any single family. Report coverage honestly — an empty result over a partial
   taxonomy is not "clean."

5. **Emit the shared schema.** Every decided finding uses
   [FINDING-SCHEMA.md](./FINDING-SCHEMA.md) — `confirmed` or `killed`, never
   "maybe." Keep killed findings; they record what was ruled out and why.

## Body structure

Same structure in every skill, so they read as one system:

1. One-line intro.
2. **When to use** — the triggers, spelled out.
3. **Scope check / gate** — authorized testing only.
4. **The loop** — numbered steps with decision points.
5. **Worked example** — a concrete input→output (a confirm *and* a kill where it
   fits). This is the highest-value section; don't skip it.
6. **Rationalizations to reject** — the shortcuts that cause missed or false
   findings, each with why it fails.
7. **Executing this in practice** — the tool-agnostic capability note.
8. **Related** — sibling skills + the schema link.

## Style

- Prefer concrete asymmetries and worked shapes over abstract advice.
- One term per concept, used consistently (pick "sink", don't drift to
  "target"/"call"/"endpoint").
- No time-sensitive content in the body; no magic constants without a reason.
- One skill = one sharp job. Link to siblings rather than inlining them.
- Adapting an existing OSS skill? Keep its license/attribution, and genuinely
  re-author to this structure — improve it, don't reskin it.
