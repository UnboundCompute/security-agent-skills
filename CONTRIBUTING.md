# Contributing

Thanks for adding to the skill library. A few conventions keep these skills
useful to people who don't run our stack.

## Skill shape

Each skill is a directory under `whitebox/` or `blackbox/` containing a
`SKILL.md`:

```markdown
---
name: kebab-case-name
description: >-
  One paragraph. Say what the skill does AND when to reach for it, in the third
  person. This is what an agent matches against, so name the trigger conditions
  explicitly.
---

# Title

Body: the method.
```

Optionally add a `references/` subdirectory for longer payload lists, checklists,
or worked examples the body links to.

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

## Style

- Lead with when-to-use, then the loop, then evidence rules, then reference impl,
  then anti-patterns.
- Prefer concrete asymmetries and worked shapes over abstract advice.
- Keep each skill focused on one move; link to sibling skills rather than
  inlining them.
