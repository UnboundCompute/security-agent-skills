# Contributing

Thanks for adding to the skill library. A few conventions keep these skills
useful to people who do not run your stack. If you want to propose a new skill or
a larger change, open an issue first so we can agree on scope before you write it.

## Skill shape

Each skill is a directory under `skills/` containing a `SKILL.md`:

```markdown
---
name: gerund-kebab-name         # e.g. detecting-race-conditions
description: >-                  # third person; what + WHEN + trigger terms; up to 1024 chars
  What the skill does AND when to reach for it, naming the trigger conditions
  explicitly. This is the routing rule an agent matches against. If it does not
  describe how a user phrases the need, the skill never loads.
license: MIT
---

# Title

Body: the method (see structure below).
```

The `name` must equal the directory name, be **gerund + kebab-case**
(`detecting-...`, `auditing-...`, `hunting-...`), be specific rather than generic
(no `helper`, `utils`, `tools`), and must not contain `claude` or `anthropic`.

Keep `SKILL.md` under about 500 lines. Push long payload lists, checklists, or
extended walkthroughs into a `references/` subdirectory and link them **one level
deep** from `SKILL.md`. Never chain `SKILL.md` to `a.md` to `b.md`.

Group the skill into a lane in the README table (white-box, black-box, or
reporting). The lane lives in the docs, not in the directory path: every skill
sits flat under `skills/` so plugin and manual discovery both find it.

## The five rules

1. **Scope check first.** Every skill opens by stating it is for authorized
   testing only. No exceptions.
2. **Method before tools.** The body is tool-agnostic reasoning: the order of
   operations, what to check, how to decide. Someone with different tools should
   still be able to follow it.
3. **Name capabilities, not products.** Describe the *capability* a step needs
   ("something that answers who calls this from a real parse"), never a specific
   tool, product, or internal system. Do not name or describe proprietary tooling
   or an internal pipeline. The skill must stand on the method alone.
4. **Adjudication discipline.** A lead or candidate is a fact, never a verdict.
   Rank is triage, never a filter. Enumerate the whole taxonomy before working
   any single family. Report coverage honestly: an empty result over a partial
   taxonomy is not "clean."
5. **Emit the shared schema.** Every decided finding uses
   [FINDING-SCHEMA.md](FINDING-SCHEMA.md), either `confirmed` or `killed`, never
   "maybe." Keep killed findings; they record what was ruled out and why.

## Body structure

Same structure in every skill, so they read as one system:

1. One-line intro.
2. **When to use**: the triggers, spelled out.
3. **Scope check or gate**: authorized testing only.
4. **The loop**: numbered steps with decision points.
5. **Worked example**: a concrete input to output (a confirm *and* a kill where
   it fits). This is the highest-value section; do not skip it.
6. **Rationalizations to reject**: the shortcuts that cause missed or false
   findings, each with why it fails.
7. **Executing this in practice**: the tool-agnostic capability note.
8. **Related**: sibling skills plus the schema link.

## Style

- Prefer concrete asymmetries and worked shapes over abstract advice.
- One term per concept, used consistently (pick "sink", do not drift to
  "target", "call", or "endpoint").
- No time-sensitive content in the body; no magic constants without a reason.
- One skill is one sharp job. Link to siblings rather than inlining them.
- Adapting an existing OSS skill? Keep its license and attribution, and genuinely
  re-author it to this structure. Improve it, do not reskin it.

## Submitting

1. Fork and branch.
2. Add or edit the skill under `skills/`, following the shape and rules above.
3. Add a row to the README skills table.
4. Open a PR describing the method the skill encodes and, if adapted, its source
   and license.
