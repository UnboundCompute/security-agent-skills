---
name: adjudicating-taint-paths
description: >-
  Decide whether a whitebox lead is a real bug by tracing taint from an untrusted
  source to a dangerous sink and confirming every hop against live source. Use
  after a scanner, a candidate list, or your own reading surfaces a "this looks
  dangerous" sink (SQL exec, system/exec, file open, deserialize, template
  render, redirect target, memcpy) and you must decide whether attacker-
  controlled input actually reaches it — or kill the lead with evidence. Covers
  forward and reverse taint, witness paths, sanitizer analysis, and the evidence
  rules that separate a finding from a false positive.
license: MIT
---

# Adjudicating taint paths: lead → decided finding

A lead is a *fact* about structure — "an input-shaped value can reach a
dangerous sink." It is never a verdict. Adjudication is the disciplined work of
deciding whether that structural possibility is a real, reachable bug on the
current source, and recording the decision so it isn't re-litigated next pass.

## When to use

- A scanner or candidate list flagged a sink and you must confirm or kill it.
- You spotted a sink by hand and want to know if attacker input reaches it.
- You need to *kill* a plausible-looking lead with evidence, not vibes.

## Scope check

Authorized source only (your own, OSS, CTF, in-scope engagement). If you can't
name the authorization, stop.

## The loop

1. **Name source and sink precisely.** Which exact argument of which sink is
   dangerous, and what is the *actual* untrusted entry — a request param, header,
   filename, env var, deserialized field? Vague framing ("user input reaches it
   somewhere") is how false positives survive.

2. **Trace the reverse cone into the sink.** What values can flow *into* this sink
   argument? This enumerates every origin. If none trace back to an untrusted
   source, the lead is dead — kill it, record why.

3. **Trace the forward cone from the source.** Where does the untrusted value go?
   If it never touches the sink, the lead is dead. Forward and reverse must agree;
   if they don't, you mis-specified an endpoint — fix it and redo.

4. **Get a witness path.** The strongest evidence is a concrete `source → … →
   sink` path. Good tooling returns either a witness or an *honest negative*
   ("no path"). A witness is a hypothesis to verify, not a proof.

5. **Confirm every hop against live source.** Read the actual body of each
   function on the path at the commit you're adjudicating. Verify the value is
   genuinely carried hop-to-hop and is not: reassigned to a constant/trusted
   value; validated, sanitized, or encoded by a guard on the path; narrowed to a
   safe type or bounded before the sink; or never actually passed by any caller.

6. **Decide and record — in the schema.** Survivor → `confirmed`: source, path,
   sink, evidence, impact. Killed → record the exact hop where taint breaks. Both
   go in the [finding schema](../../FINDING-SCHEMA.md); killed findings are kept.

## Evidence rules

- **Confidence is not truth.** A high-confidence edge is strong support; a
  conservative/over-approximated edge is included to avoid missing a path and is
  frequently spurious — a witness leaning on one demands extra source
  confirmation.
- **An absent edge is not proof of safety.** The tool may not model that path.
  Dynamic dispatch (attribute/vtable), function pointers, and
  `getattr`/`eval`/reflection are standard blind spots — "no path" through one of
  those is inconclusive, not clean. Confirm by reading source.
- **A sanitizer only helps if it covers the payload class.** An HTML encoder does
  nothing for a SQL context; a `realpath` check does nothing for a symlink race.
  Match the guard to the sink's *context*, not to its name.

## Worked example (a kill and a confirm)

> **Kill.** Lead: `GET /search?q=` → `cursor.execute(sql)`. Reverse cone shows
> `q` reaches `execute`, but reading the hop shows `execute(sql, (q,))` — `q` is
> a bound parameter, never concatenated into `sql`. **Killed**, `kill_reason` =
> "bound param at the sink; q never enters the SQL string."
>
> **Confirm.** Lead: `body.filename` → `open(path, 'w')`. Forward cone reaches
> the sink; reading each hop shows `name = body['filename']` (unchecked) →
> `path = base / name` → `open`. `/etc/x`-style input escapes `base`. **Confirmed**,
> `high`, impact = arbitrary file write → RCE via config/cron drop.

## Rationalizations to reject

- *"The witness path is enough."* → Not without reading source on every hop.
- *"There's a sanitizer, it's fine."* → Only if it covers this sink's context.
  Check what it actually enforces.
- *"No path found, so it's safe."* → Not if the path would run through a blind
  spot (reflection, function pointers, dynamic dispatch). Confirm by hand.
- *"I'll skip writing down why I killed it."* → Then you re-open it next pass.
  Record the kill reason.

## Executing this in practice

Run the loop with whatever answers three questions from a real parse: what
reaches a sink argument (reverse cone), where a source value flows (forward
cone), and the exact current source of any function on the path. A code property
graph answers all three; a taint-tracking analyzer covers most; on a small
target you trace by hand. Step 5 (source confirmation) is never optional — the
tool proposes, you confirm.

## Related

- `hunting-bugs-with-a-code-graph` — the master loop that surfaces leads.
- `auditing-guard-gaps` — when the "sanitizer" is present on one path but missing
  on a sibling.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) — the shape every decision takes.
