---
name: code-graph-bughunt
description: >-
  Hunt for security bugs in a codebase using a code property graph instead of
  grep/read sweeps. Use when the question is structural — who calls this, what
  reaches this sink, which sibling functions are unguarded — on an authorized
  target you have source access to (your own code, an OSS project, or an
  engagement with source in scope). Covers building/loading a graph, orienting
  from the centrality spine, enumerating the whole candidate taxonomy, and
  turning leads into confirmed findings.
---

# Whitebox bug hunting over a code property graph

A code property graph (CPG) is a compiler-precise index of a codebase: symbols,
call edges, and dataflow, built from a real parse rather than text matching.
Hunting over a CPG beats grep/read because it does not miss a caller behind a
rename, an alias, or an import indirection, and because it lets you ask
*structural* questions ("what can flow into this parameter") that text search
cannot answer.

This skill is the master workflow. Two companions go deeper on specific moves:
`taint-adjudication` (turning a lead into a confirmed finding) and
`guard-sibling-audit` (finding the unguarded peer of a guarded function).

## Scope check first

Only run this on code you are authorized to analyze: your own, a project you
are contributing to, an OSS codebase, or an engagement where source review is
in scope. If you cannot name why you are allowed to read this source, stop.

## The method (works with any CPG tooling)

1. **Get a graph of the target.** Build an index of the source tree. A CPG is a
   *snapshot* — rebuild it whenever the code changes in a way that matters to
   the answer, or you will adjudicate against stale structure.

2. **Orient before you hunt.** Start from the centrality spine — the most
   connected functions — not from a file you happened to open. These hubs are
   where input arrives, where trust boundaries sit, and where a bug has the most
   reach. Map the folders and the top-level entry points before drilling in.

3. **Enumerate the whole taxonomy — never one family.** If your tooling has a
   candidate/lead registry, list *every* family it carries before looking at any
   single one. Do not scope the hunt to `memory.copy`, or injection, or any one
   class because it is what you expect. Coverage is over the whole taxonomy.

4. **Treat rank as triage, not a filter.** A registry ranks leads to order your
   attention. Inclusion is exhaustive; the rank never justifies dropping a
   low-ranked candidate from examination. Work down the list; do not truncate it.

5. **Candidates are facts, never verdicts.** A lead means "the structure here
   matches a known-dangerous shape," not "this is a bug." Adjudicate each one by
   reading its provenance — the source→sink path — and confirming against the
   live current source. Only survivors are bugs. (See `taint-adjudication`.)

6. **Audit guarded/unguarded peers.** When one function validates its input and
   a sibling that reaches the same sink does not, the sibling is the bug. Diff
   guarded against unguarded systematically. (See `guard-sibling-audit`.)

7. **Report honestly about coverage.** An empty result over a *partial* taxonomy
   is not "the code is clean." Say which categories were out of scope of the
   catalog (temporal/lifetime bugs like use-after-free and double-free,
   integer-overflow-as-a-class, and uninitialized/NULL deref are commonly *not*
   modeled as families) versus which were covered and came back genuinely clean.

## Reading the evidence

Every edge in a CPG carries a confidence and an origin. Read them as evidence,
not as verdicts:

- An **exact** or **high** edge is strong support.
- A **conservative** edge is a safe over-approximation — the tool included it to
  avoid missing a real path, so it may be spurious.
- An **absent** edge is *not* proof that no path exists.

Known blind spots in most CPGs: dynamic dispatch through attributes, C function
pointers, and anything through `getattr`/`eval`/reflection. Confirm those by
reading the source directly — the graph will under-report them.

## Executing this in practice

The steps above are the method; run them with whatever structural tooling you
have. At minimum you need something that answers, from a real parse rather than
text matching: who calls a function, what a value can flow into, and the exact
source of a declaration. A code property graph gives you all of that; a good
static analyzer covers much of it; careful manual tracing covers the rest on a
small target. Keep the *snapshot* caveat in mind whatever you use — re-index
after the source changes.

## Anti-patterns

- Hardcoding one bug family as "the" target. Enumerate the census first, always.
- Letting a low rank drop a candidate. Rank orders attention; it never filters.
- Calling a candidate a bug without reading its provenance against live source.
- Trusting an absent edge as proof of safety, or a conservative edge as proof of
  a bug.
- Declaring "clean" when the taxonomy only covered part of the bug space.
