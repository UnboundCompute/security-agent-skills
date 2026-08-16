---
name: guard-sibling-audit
description: >-
  Find the missing-validation bug by comparing sibling functions that reach the
  same sink — one guards its input, its peer does not. Use on an authorized
  source target when you want to surface access-control gaps, missing bounds
  checks, or skipped sanitization by diffing a function against its structural
  peers rather than reading files linearly. Covers identifying guarded call
  sites, enumerating siblings, and confirming the unguarded peer is truly
  reachable with attacker input.
---

# Guard/sibling audit: the unguarded peer is the bug

Many real bugs are not exotic — they are one handler in a family that forgot the
check its siblings all perform. The `admin_delete` that checks a role and the
`bulk_delete` that doesn't. The `read_bounded` that validates length and the
`read_fast` that trusts it. Reading files linearly hides these; comparing peers
*structurally* surfaces them. This is one of the highest-yield whitebox moves.

## The core pattern

Given a dangerous sink reachable from several call sites, the bug is usually the
call site that skips a guard its peers apply. You are looking for an asymmetry:

```
guard  →  sink   (safe path — the intended design)
   ???  →  sink   (peer path with the guard missing — the bug)
```

## The audit loop

1. **Pick a guarded anchor.** Find a function that *does* validate before a
   sensitive sink — an auth/role check, a bounds check, an allowlist, an
   ownership check, an input sanitizer. This is your reference for "what correct
   looks like here."

2. **Identify the guard and the sink it protects.** Name the exact check and the
   exact sink. "Correct" means: this specific guard dominates this specific sink
   on every path through the anchor.

3. **Enumerate the siblings.** Find the structural peers — functions that reach
   the *same* sink, or that share the anchor's role/shape (same handler family,
   same base class, same dispatch table). These are your suspects.

4. **Diff each sibling against the anchor.** For every peer, ask: does the same
   guard dominate the same sink here? Look specifically for:
   - the guard entirely absent,
   - the guard present but *bypassable* on some path (early return, an `||` that
     short-circuits, a branch that skips it),
   - a *weaker* guard (checks authentication but not authorization; checks type
     but not bounds; checks one field but not the sibling field),
   - the guard applied *after* the sink instead of before.

5. **Confirm reachability with attacker input.** An unguarded sink is only a bug
   if untrusted input actually reaches it. Trace the peer's dangerous argument
   back to a real source (hand off to `taint-adjudication`). A sibling that is
   only ever called with constants is not a finding.

6. **Record the asymmetry.** State it as a diff: "anchor A guards sink S with
   check C; peer B reaches S with no equivalent of C, and B's argument is
   attacker-controlled via <source>." That framing is the finding.

## What counts as a "sibling"

Cast the net wider than same-file neighbors — structural peers are the point:

- functions that reach the same sink (the strongest signal),
- overrides of the same interface/virtual method,
- handlers registered in the same route table / dispatch map / command switch,
- members of the same class or module with parallel names
  (`get_x`/`set_x`/`delete_x`),
- callers of a shared helper, where some sanitize the argument first and some
  pass it raw.

## Executing this in practice

The audit needs tooling that can group functions by the sink they reach and show
you the exact body of each, so you can diff guard-for-guard. A code property
graph makes "which peers reach this sink, and which of them validate first"
answerable directly; otherwise, enumerate the peers by hand from the call sites
of the shared sink or helper and read each. The reachability step (5) needs the
same source→sink tracing as `taint-adjudication` — reuse it.

## Anti-patterns

- Reporting a "missing guard" without confirming attacker input reaches the sink
  — an unreachable unguarded sink is not a finding.
- Matching guards by name instead of by what they enforce (an auth check is not
  an authz check; a type check is not a bounds check).
- Only comparing same-file neighbors and missing the peer in another module that
  reaches the same sink.
- Missing a guard that is present but bypassable via an early return or a
  short-circuiting branch — read for domination, not mere presence.
