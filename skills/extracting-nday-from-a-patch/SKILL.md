---
name: extracting-nday-from-a-patch
description: >-
  Turn a security patch or version diff into fresh findings: infer the fixed
  vulnerability from what the fix changed, reconstruct the pre-patch bug, then
  hunt the paths the fix did not cover and the same bug in code it never touched.
  Use when you have a fix commit, a vague advisory with a linked diff, a version
  bump, or a "security release" and want to know what it silently fixed and what
  it missed. Covers reading a fix as a treasure map, incomplete-fix analysis, and
  variant discovery in the same tree and its forks.
license: MIT
---

# Extracting n-day from a patch: the fix is the map

A security fix is a confession. It tells you exactly where the bug was, what the
missing check should have been, and, by omission, which other paths the author
did not think to cover. Reading a patch backward, from fix to bug, is one of the
highest-yield techniques in vulnerability research: the hard part (locating the
defect) is already done for you, and the incomplete fixes are waiting.

## When to use

- A fix commit or security release landed and you want to know what it addressed.
- An advisory is deliberately vague but links a diff, a PR, or a tag.
- You maintain or depend on a fork and need to know if a fix upstream applies.
- You confirmed one bug via its patch and want the instances the patch missed.

## Scope check

You may read and reason about public patches and advisories freely. You may only
*test or exploit* against code and systems you are authorized for (your own, OSS
you run, CTF, in-scope engagement). Analysis is not authorization to attack.

## The loop

1. **Read the fix, infer the bug.** Look at what the patch *adds* or *tightens*,
   and name the vulnerability it implies. A new length or bounds check implies an
   out-of-bounds read or write. A new authorization call implies missing access
   control. A new escape or parameterization implies injection. A new
   normalization implies traversal or confusion. The added guard tells you the
   invariant that was being violated.

2. **Locate the pre-patch sink and its precondition.** In the *vulnerable*
   revision, find the exact operation that was unsafe and the condition under
   which it misbehaves (the input shape, size, or state the new guard now
   rejects). This is the bug, stated concretely.

3. **Reconstruct the reachable path.** Confirm an untrusted input could reach that
   sink in the vulnerable version. Because the fix exists, the bug is real by
   construction; your job is to establish the trigger, not to re-prove the flaw.

4. **Test the fix for completeness.** This is the payoff. Ask whether the new
   guard sits on *every* path to the sink or only the reported one. Common gaps:
   the check was added to one caller but a second caller still reaches the sink
   raw; the check runs after an early operation that is already unsafe; the guard
   covers one input shape but a sibling shape (a different encoding, a second
   parameter) slips past. An incomplete fix is a live bug in a patched version.

5. **Hunt variants the patch never touched.** Extract the defect's shape and sweep
   the rest of the tree, sibling modules, vendored copies, and downstream forks
   for the same mistake in code the fix did not visit. A fix is scoped to one
   report; the same author's same habit usually recurs.

6. **Adjudicate and record.** Each incomplete-fix path and each variant is a lead.
   Confirm or kill it on live source and record it in the schema, with `commit`
   set to the revision you adjudicated against and the fix commit noted as the
   seed.

## Reading a fix well

- **A guard added means a guard was missing.** Name what the guard enforces, then
  ask where else that invariant must hold and does not.
- **Refactors hide fixes.** A "cleanup" or "rename" commit next to a release
  sometimes carries the real security change. Diff behavior, not just lines.
- **Test files leak the bug.** A new test case in the patch often spells out the
  exact malicious input, saving you the reconstruction.
- **The advisory undersells it.** "Minor hardening" frequently means a real,
  reachable vulnerability the vendor chose to describe quietly.

## Worked example (a confirm and a kill)

> **Patch.** A release adds `if (nmemb > SIZE_MAX / size) return NULL;` before an
> allocation that is then filled by a copy loop.
>
> **Infer.** Pre-patch integer overflow in the size computation leads to an
> undersized buffer and an out-of-bounds write in the fill loop.
>
> **Completeness (confirm).** Two other functions compute `nmemb * size` the same
> way and allocate without the new check; one is reachable from a parser fed by
> file input. The fix missed it. **Confirmed**, incomplete fix, `high`.
>
> **Variant (kill).** A third site also multiplies, but the operands are a fixed
> compile-time constant and a bounded enum; no attacker influence and no overflow.
> **Killed**, `kill_reason` = "both operands bounded and non-attacker-controlled."

## Rationalizations to reject

- *"The patch fixed it, so the version is safe."* → The patch fixed the *reported*
  path. Completeness analysis is the whole point; assume it missed one.
- *"The advisory says low severity."* → Vendor severity is a business decision.
  Reconstruct the bug and judge impact yourself.
- *"No diff, just a version bump."* → Then diff the two versions yourself; the
  security change is in there somewhere.
- *"It's the same code, so it's the same bug."* → A structural twin can be safe
  because a caller bounds the input. Adjudicate every variant.

## Executing this in practice

You need a way to read two revisions side by side, to establish reachability to a
sink in the vulnerable version, and to sweep the tree for the defect's shape. A
diff tool plus a structural query (a code property graph or structural search)
covers it; on a small change you can trace by hand. The inference in step 1 and
the completeness judgment in step 4 are yours; the tools only surface the sites.

## Related

- `hunting-bug-variants` - the sweep in steps 4 and 5, generalized.
- `adjudicating-dependency-cve-reachability` - when the patched code is a
  dependency and you need to know if the pre-fix bug reaches your app.
- `adjudicating-taint-paths` - the confirm/kill loop for each reconstructed path.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - the shape every finding takes.
