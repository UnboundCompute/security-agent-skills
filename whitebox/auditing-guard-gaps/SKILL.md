---
name: auditing-guard-gaps
description: >-
  Find the missing-check bug by comparing sibling functions that reach the same
  sink — one validates its input, its peer does not. Use on an authorized source
  target to surface broken access control, missing bounds checks, and skipped
  sanitization that linear file-reading hides; when you suspect one handler in a
  family forgot the check its siblings all perform. Covers finding a guarded
  anchor, enumerating structural peers, diffing guard-for-guard by what each
  actually enforces, and confirming the unguarded peer is reachable with attacker
  input.
license: MIT
---

# Auditing guard gaps: the unguarded peer is the bug

Most access-control and missing-validation bugs aren't exotic — they're one
handler in a family that forgot the check its siblings perform. The
`admin_delete` that checks a role and the `bulk_delete` that doesn't. The
`read_bounded` that validates length and the `read_fast` that trusts it. Reading
files top-to-bottom hides these; comparing *peers* surfaces them. This is one of
the highest-yield whitebox moves.

## The asymmetry you're hunting

```
guard  →  sink     (the intended, safe path)
  ???  →  sink     (a peer path with the guard missing — the bug)
```

## When to use

- A sink is reachable from several call sites and you suspect one skips a check.
- You're auditing an authz model, a parser family, or a handler group for a
  forgotten check.
- A finding needs its "why is this wrong" framed as a concrete diff against a
  correct peer.

## Scope check

Authorized source only. If you can't name the authorization, stop.

## The loop

1. **Pick a guarded anchor.** Find a function that *does* validate before a
   sensitive sink — an authz/ownership check, a bounds check, an allowlist, a
   sanitizer. This is your reference for "what correct looks like here."

2. **Pin the guard to the sink it protects.** Name the exact check and the exact
   sink. "Correct" means the guard **dominates** the sink: on *every* path
   through the anchor, the check runs before the sink. Mere presence in the
   function is not domination.

3. **Enumerate the structural peers.** Widen past same-file neighbors — the point
   is structural kinship: functions that reach the *same* sink; overrides of the
   same interface/virtual method; handlers in the same route table / dispatch map
   / command switch; parallel members of a class (`get_x`/`set_x`/`delete_x`);
   callers of a shared helper where some sanitize the argument and some pass it
   raw. Peers that reach the same sink are the strongest signal.

4. **Diff each peer against the anchor — by what the guard enforces, not its
   name.** For every peer, ask whether an equivalent check dominates the same
   sink. Classify the gap:
   - **absent** — no equivalent check at all.
   - **bypassable** — present but skippable on some path (early return, a
     short-circuiting `||`, a branch that jumps the check).
   - **weaker-than-peer** — a *different kind* of check: authentication where the
     anchor does authorization; a type check where the anchor does bounds; one
     field where the anchor checks the sibling field too.
   - **after-sink** — the check runs, but *after* the dangerous operation.
   - **wrong-context** — a real check that doesn't cover this sink's payload class
     (an encoder that's wrong for the sink's language).

5. **Confirm reachability with attacker input.** An unguarded sink is only a bug
   if untrusted input reaches it. Trace the peer's dangerous argument back to a
   real source (hand to `adjudicating-taint-paths`). A peer only ever called with
   constants is not a finding.

6. **Record as a diff — in the schema.** State it as: "anchor A guards sink S
   with check C (dominating); peer B reaches S with `guard_status` = <absent /
   bypassable / weaker-than-peer / after-sink / wrong-context>, and B's argument
   is attacker-controlled via <source>." Emit per the
   [finding schema](../../FINDING-SCHEMA.md).

## Worked example

> **Anchor:** `delete_own_post(user, post_id)` — checks `post.owner == user.id`
> (dominates) before `db.delete(post)`.
> **Peer:** `admin_bulk_delete(ids)` reaches the same `db.delete` sink; the only
> gate is a route-level `@login_required`, no per-object ownership check.
> **Diff:** `guard_status = weaker-than-peer` — authentication present,
> authorization (ownership) absent.
> **Reachability:** `ids` comes straight from the request body; any
> authenticated user hits it. **Confirmed IDOR/BFLA**, severity high — "any
> logged-in user deletes any user's posts by id."

## Rationalizations to reject

- *"There's a check in the function, so it's guarded."* → Only if it **dominates**
  the sink. A check on a branch the sink skips is no guard.
- *"Both functions have auth, they're equivalent."* → Authentication is not
  authorization; a type check is not a bounds check. Compare what's *enforced*.
- *"The peer looks unguarded — report it."* → Not until you confirm attacker input
  reaches the sink. Unreachable ≠ finding.
- *"Same-file neighbors only."* → The dangerous peer often lives in another module
  reaching the same sink. Enumerate by structure, not by file.

## Executing this in practice

You need tooling that can group functions by the sink they reach and show the
exact body of each, so you can diff guard-for-guard and check domination. A code
property graph makes "which peers reach this sink, and which validate first"
directly answerable; otherwise enumerate peers by hand from the call sites of the
shared sink or helper and read each. The reachability step reuses
`adjudicating-taint-paths`.

## Related

- `adjudicating-taint-paths` — confirm the peer's sink is reachable with input.
- `hunting-bugs-with-a-code-graph` — the master loop this feeds.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) — the shape every finding takes.
