---
name: hunting-bugs-with-a-code-graph
description: >-
  Hunt security bugs across a whole codebase by reasoning over its structure —
  call graph and dataflow — instead of grepping for keywords. Use when you have
  source access to an authorized target (your own code, an OSS project, or an
  in-scope engagement) and want systematic coverage of a bug taxonomy rather
  than a single hunch; when the question is "who calls this, what reaches this
  sink, which peer function is unguarded." Orients on an unfamiliar codebase,
  enumerates the full bug taxonomy before drilling in, and turns structural
  leads into decided findings.
license: MIT
---

# Hunting bugs with a code graph

Grep finds strings; it misses the caller behind a rename, an alias, or an import
indirection, and it cannot answer "what can flow into this argument." Reasoning
over a codebase's *structure* — its call graph and dataflow — can. This skill is
the master loop for a source-level hunt. Two companions go deeper on single
moves: `adjudicating-taint-paths` (lead → decided finding) and
`auditing-guard-gaps` (the unguarded peer of a guarded function).

## When to use

- You have source and want *coverage*, not a one-off keyword search.
- You're cold on an unfamiliar codebase and need to find where input arrives.
- You want to work a bug *taxonomy* systematically and prove what you ruled out.

## Scope check (do this first)

Only run on code you're authorized to analyze: your own, an OSS project you
contribute to, a CTF, or an engagement where source review is in scope. If you
can't name why you're allowed to read this source, stop.

## The loop

1. **Index the target.** Build a structural index of the source tree. It is a
   *snapshot* — re-index whenever the code changes in a way that matters, or you
   will adjudicate against stale structure.

2. **Orient before hunting.** Start from the most-connected functions — the
   structural spine, where input arrives and trust boundaries sit — not from a
   file you happened to open. Map the top-level entry points and the module
   layout before drilling in.

3. **Enumerate the whole taxonomy — never one family.** List *every* bug class
   your catalog covers before examining any single one. Do not scope the hunt to
   the class you expect (injection, memory copy, whatever) because it's familiar.
   Coverage is over the whole taxonomy; a hunt that only ever looks at one family
   is not a hunt, it's a confirmation of a hunch.

4. **Rank is triage, not a filter.** A ranked lead list orders your *attention*.
   Inclusion is exhaustive; a low rank never justifies dropping a candidate from
   examination. Work down the list — don't truncate it.

5. **Leads are facts, not verdicts.** A lead means "the structure here matches a
   known-dangerous shape," never "this is a bug." Adjudicate each by tracing its
   source→sink provenance and confirming against live source. Only survivors are
   findings. (→ `adjudicating-taint-paths`.)

6. **Diff guarded against unguarded peers.** When one function validates before a
   sink and a sibling reaching the same sink does not, the sibling is the bug.
   (→ `auditing-guard-gaps`.)

7. **Record coverage honestly.** Emit every decided finding in the shared
   [finding schema](../../FINDING-SCHEMA.md), *including* killed leads. Then state
   what the taxonomy did **not** cover. Temporal/lifetime classes (use-after-free,
   double-free), integer-overflow-as-a-class, and uninitialized/NULL deref are
   commonly *not* modeled as catalog families — hunt those with
   `detecting-memory-safety-bugs` and `detecting-race-conditions`. An empty result
   over a partial taxonomy is not "the code is clean"; say which classes were out
   of scope of the catalog versus genuinely checked and clean.

## Worked example

> **Cold start on a Python web service.**
> 1. Orient: the spine surfaces `app.dispatch` and three request handlers as the
>    highest-traffic nodes — that's where untrusted input lands.
> 2. Enumerate taxonomy: catalog lists path-traversal, injection, SSRF, open-
>    redirect, deserialization, missing-authz (and flags memory/temporal classes
>    as *not modeled*).
> 3. Work each family; in path-traversal the lead is `export.filename → open()`.
> 4. Adjudicate: trace shows no normalization between source and sink →
>    **confirmed**, emitted per the schema (see the schema's worked record).
> 5. Guard audit: `delete_own_post` checks ownership; peer `admin_bulk_delete`
>    reaches the same delete sink with no ownership check → second finding.
> 6. Report: 2 confirmed, 4 families clean, memory/temporal classes flagged
>    out-of-catalog and handed to the dedicated skills.

## Rationalizations to reject

Shortcuts that cause misses and false positives — refuse them:

- *"This family is where the bug will be, I'll start there."* → You'll stop there.
  Enumerate the whole census first.
- *"Low rank, skip it."* → Rank orders attention, never filters. Read it.
- *"The lead looks real, log it."* → A lead is not a finding until you've read the
  source on every hop. Adjudicate or drop.
- *"No leads came back, the code is clean."* → Only true for classes the catalog
  actually models. Name the gaps.

## Executing this in practice

Run the loop with whatever structural tooling you have. At minimum you need
something that answers, from a real parse rather than text matching: who calls a
function, what a value can flow into, and the exact source of a declaration. A
code property graph gives you all three; a good static analyzer covers much of
it; careful manual tracing covers the rest on a small target. Whatever you use,
keep the snapshot caveat — re-index after the source changes.

## Related

- `adjudicating-taint-paths` — confirm or kill a source→sink lead.
- `auditing-guard-gaps` — find the unguarded peer of a guarded function.
- `detecting-memory-safety-bugs`, `detecting-race-conditions` — the classes a
  catalog usually doesn't model.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) — the shape every finding takes.
