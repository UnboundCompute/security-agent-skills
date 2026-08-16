---
name: hunting-bug-variants
description: >-
  Given one confirmed vulnerability, systematically find its siblings: the same
  defect shape repeated elsewhere in the codebase, and the parts of it the fix
  left uncovered. Use right after you confirm or read about a bug (your own
  finding, a CVE, a patch, a writeup) and want the other instances instead of
  stopping at one. Turns a single seed into a structural signature and sweeps the
  whole tree for same-shape code, copy-paste clones, sibling handlers, and
  incomplete fixes. Covers signature extraction, the variant sweep, and
  adjudicating each candidate.
license: MIT
---

# Hunting bug variants: one seed, every sibling

One bug is almost never alone. The same mistake gets copy-pasted, re-implemented
by a second author, or left uncovered by a fix that only patched the reported
call site. Variant hunting takes a *single confirmed defect* and turns it into a
signature you can sweep the whole codebase with, so you find the three other
copies before an attacker does.

## When to use

- You just confirmed a bug and want its siblings, not a victory lap.
- A CVE, advisory, or writeup describes a bug in code you can read.
- A fix landed and you want to know whether it covered every instance.
- Someone says "we already patched that" and you want to verify it is really gone.

## Scope check

Authorized source only (your own, OSS, CTF, in-scope engagement). If you can't
name the authorization, stop.

## The loop

1. **Anchor the seed.** State the confirmed defect precisely: the exact sink, the
   exact missing or wrong step (no bounds check, no authz check, unnormalized
   path, unescaped context), and the untrusted input that drives it. A vague seed
   ("there was an injection somewhere") produces a useless sweep.

2. **Extract the shape, not the string.** Abstract the seed into a structural
   signature that survives renaming and reformatting. Usually one of: *this sink
   called without that guard*; *this unsafe idiom* (raw concat into a query,
   `base / user_name` without resolve-and-contain); *this type of value reaching
   this argument*. Grepping the literal code finds only exact copies; the shape
   finds the re-implementations.

3. **Sweep for same-shape sites.** Enumerate every place matching the signature:
   other callers of the same sink, other functions with the same step missing,
   copy-paste clones, sibling handlers in the same module, the same pattern in a
   different subsystem. Cast wide here; ranking is triage, not a filter.

4. **Check the fix itself for gaps.** If the seed came with a patch, treat the fix
   as suspect: does it cover *every* reachable path to the sink, or only the
   reported one? Was the check added to the caller but not a second caller? Is
   there a pre-existing path that bypasses the new check? Incomplete fixes are the
   highest-yield variants because everyone believes the bug is dead.

5. **Adjudicate each candidate.** A same-shape site is a lead, never a verdict.
   Confirm or kill each by tracing input to the sink and reading the hops on live
   source, exactly as for any lead. A structural twin can still be safe because a
   caller bounds the input or a guard sits one frame up.

6. **Record every decision in the schema.** Each confirmed variant is its own
   finding (its own `id`, path, evidence). Killed twins are kept with the reason
   they differ from the seed, so the next person doesn't re-chase them.

## What makes a good signature

- **Tight enough to rank, loose enough to catch re-implementations.** "Any call
  to `write`" is too loose; "the literal three lines from the seed" is too tight.
  Aim at "a filesystem write whose path is a user value joined to a base without a
  containment check."
- **Anchored on the invariant that was violated,** not on incidental details.
  The bug is "the containment check is missing," not "the function was named
  `export`." Sweep on the invariant.
- **Include the negative half.** Part of the signature is what the *correct*
  siblings do (the guard that the seed lacked). The correct ones tell you exactly
  what the broken ones are missing.

## Worked example (a confirm and a kill from one seed)

> **Seed (confirmed).** Path traversal in report export: `name = body['filename']`
> flows to `open(base / name, 'w')` with no containment check. `/etc/x`-style
> input escapes `base`.
>
> **Signature.** A filesystem sink whose path is `base / <user value>` with no
> resolve-and-reject-outside-base step between the input and the sink.
>
> **Sweep** surfaces three same-shape sites: backup restore, avatar upload, and
> log export.
>
> **Confirm.** Backup restore builds `restore_dir / archive['name']` and extracts
> there with no containment check; `../` in the archive entry escapes. **Confirmed**,
> new finding, `high`.
>
> **Kill.** Avatar upload builds `base / fname` but reading the hop shows `fname`
> is a server-generated UUID, never the client's name. **Killed**, `kill_reason` =
> "path component is a generated UUID, not attacker input; the seed's invariant
> holds here."

## Rationalizations to reject

- *"I found and fixed the bug, done."* → You fixed one instance. The sweep is the
  actual work; the seed was just the sample.
- *"Grep found no other copies, so it's unique."* → Grep finds strings. A second
  author's re-implementation has none of the same tokens and the same flaw. Sweep
  the shape.
- *"The fix obviously covers it."* → Fixes routinely patch the reported call site
  and miss a sibling caller or a pre-existing bypass. Verify coverage, don't
  assume it.
- *"It matches the pattern, so it's a bug."* → A match is a lead. Adjudicate it;
  a structural twin can be safe for a reason the seed wasn't.

## Executing this in practice

The sweep needs a way to enumerate structural matches across the tree from a real
parse: every caller of a sink, every function missing a step its peers have,
clones of an idiom. A code property graph or a structural-search tool does this;
on a small target you read the neighbors by hand. Extraction (step 2) and
adjudication (step 5) are human judgment the tool only feeds.

## Related

- `auditing-guard-gaps` - when the seed's invariant is specifically a missing
  guard and you want the unguarded peers of a guarded function.
- `adjudicating-taint-paths` - the confirm/kill loop each candidate goes through.
- `extracting-nday-from-a-patch` - when the seed is a patch rather than your own
  finding.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - the shape every confirmed and
  killed variant takes.
