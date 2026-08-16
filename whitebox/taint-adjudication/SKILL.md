---
name: taint-adjudication
description: >-
  Turn a whitebox lead into a confirmed finding (or kill it) by tracing taint
  from source to sink over a code property graph and confirming against live
  source. Use after a candidate registry, a scanner, or your own read surfaces a
  "this looks dangerous" lead, and you need to decide whether real attacker-
  controlled input actually reaches the sink. Covers forward flow, reverse
  source-of analysis, witness paths, and the evidence rules that separate a bug
  from a false positive.
---

# Adjudicating a taint lead: fact → confirmed finding

A lead is a *fact* about structure: "input-shaped value X can reach dangerous
sink Y." It is never a verdict. Adjudication is the work of deciding whether that
structural possibility is a real, reachable bug on the current source. This skill
is the disciplined version of that decision.

## When to use

- A candidate registry or scanner flagged a sink (SQL exec, `system`, `memcpy`,
  deserialize, path join, template render, redirect target...).
- You spotted a sink by hand and want to know if attacker input reaches it.
- You need to *kill* a plausible-looking lead with evidence, not vibes.

## The adjudication loop

1. **Name the sink and the untrusted source precisely.** Which exact argument of
   which sink is dangerous? What is the *actual* attacker-controlled entry — a
   request param, a header, a filename, an env var, a deserialized field? Vague
   framing ("user input reaches it somewhere") is how false positives survive.

2. **Trace the reverse cone into the sink.** Ask: what values can flow *into*
   this sink argument? This enumerates every origin. If none of them trace back
   to an untrusted source, the lead is dead — kill it and record why.

3. **Trace the forward cone from the source.** From the untrusted entry, where
   does the value go? If the forward cone never touches the sink, the lead is
   dead. Forward and reverse should agree; if they don't, you mis-specified one
   endpoint — fix it and redo.

4. **Get a witness path.** The strongest evidence is a concrete labeled path
   source → … → sink. A good tool returns either a witness or an *honest
   negative* ("no path found"). A witness is a hypothesis to verify, not a
   proof — the next step confirms it.

5. **Confirm every hop against live source.** Read the actual body of each
   function on the path at current HEAD. Verify the value is genuinely carried
   hop-to-hop and is not:
   - reassigned to a constant or a trusted value along the way,
   - validated / sanitized / encoded by a guard on the path,
   - narrowed to a safe type or bounded before the sink,
   - unreachable because a caller never passes the tainted argument.

6. **Decide and record.** Survivor → confirmed finding: state the source, the
   path, the sink, and *why* the value is dangerous when it lands. Killed →
   record the exact hop where taint is broken, so the adjudication is auditable
   and you don't re-open it next pass.

## Evidence rules (read edges as evidence, not verdicts)

- **Confidence matters.** An `exact`/`high` edge is strong; a `conservative`
  edge is a safe over-approximation the tool added to avoid missing a path — it
  is frequently spurious, so a witness that leans on a conservative edge demands
  extra source confirmation.
- **An absent edge is not proof of safety.** The graph may simply not model that
  path. Dynamic dispatch (attribute/vtable), function pointers, and
  `getattr`/`eval`/reflection are standard blind spots — a "no path" over one of
  those is inconclusive, not clean. Confirm by reading source.
- **A sanitizer on the path only helps if it actually covers the payload class.**
  An HTML-encoder does nothing for SQL context; a `realpath` check does nothing
  for a symlink race. Match the guard to the sink's context, not to its name.

## Executing this in practice

Run the loop with whatever tooling answers three questions from a real parse:
what values reach a given sink argument (the reverse cone), where a given source
value flows (the forward cone), and the exact current source of any function on
the path. A code property graph answers all three directly; a taint-tracking
static analyzer covers most of it; on a small target you can trace it by hand.
Whatever you use, the source-confirmation step (5) is not optional — the tool
proposes a path, you confirm it against live code.

## Anti-patterns

- Promoting a witness path to a bug without reading the source on every hop.
- Ignoring a sanitizer because it's inconvenient — or trusting one by its name
  without checking it covers the sink's context.
- Treating "no path found" as "safe" when the path would run through a known
  blind spot (reflection, function pointers, dynamic dispatch).
- Re-opening a killed lead next pass because the kill reason wasn't recorded.
