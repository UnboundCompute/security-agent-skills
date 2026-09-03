---
name: hunting-unicode-normalization-and-canonicalization-bypass
description: >-
  Hunt security bypasses where a check passes on one representation of untrusted input and a normalization,
  decoding, or case-folding step then turns it into a different, dangerous form at the sink. Use when an
  authorization, filter, allowlist, path-containment, or identifier comparison runs on input that is later
  normalized, percent- or entity-decoded, or case-folded before it reaches the operation it guards. Covers
  Unicode normalization forms, overlong and double encoding, case-fold and locale collisions, and
  mixed-script or homoglyph identifiers. The input that passes the check in one form is the source, the
  security decision made on the wrong form is the sink, and the bypass or identity confusion that the
  later transform enables is the bug.
license: MIT
---

# Hunting unicode normalization and canonicalization bypass: when the check and the sink read different strings

A security check compares a string against a rule: this path stays under the root, this name is not on the
denylist, this identifier equals the owner's. The comparison is only sound if the string being checked is
the same string the sink will act on. Text has many representations, and systems transform it constantly:
they normalize Unicode, decode percent- or entity-encoding, and fold case, often after the check has
already passed. When the transform happens between the check and the sink, an attacker submits a form that
looks safe to the check and becomes dangerous at the sink. A denied word slips through in a composed form
that normalizes back to it; an identity check passes because two different code points fold to the same
letter; a homoglyph domain reads as the real one. The bug is an ordering bug: canonicalize after you check.
You find these by locating each security decision and asking whether it runs on the same canonical form the
sink uses.

## When to use

- An authorization, filter, allowlist, or denylist decision runs on a string that is later transformed.
- Path containment, host comparison, or an identifier equality check precedes a decode or normalization.
- Usernames, emails, domains, or filenames are compared or displayed across scripts and encodings.

## Scope check

Test normalization and canonicalization behavior only against systems you own or are authorized to assess,
from test accounts, using benign markers to demonstrate that a check is bypassed rather than acting on the
access it grants. A confirmed bypass reaches protected functions or identities, so stay within the
authorized scope. If you can't name the authorization, stop.

## The loop

1. **Establish whether the check and the sink use the same canonical form first.** For each security
   decision, determine the representation it inspects and the representation the sink finally acts on, and
   whether any normalization, decoding, or case-folding happens between them. This is the false-positive
   killer: a system that canonicalizes input to a single form and then both checks and acts on that form is
   not bypassable, while one that checks the raw or partly-decoded input and transforms it afterward is. Name
   the order of check and transform before crafting input.

2. **Map the transforms on the path.** Trace where input is normalized to a Unicode form, percent- or
   entity-decoded (and whether more than once), lowercased or case-folded, and where separators or hosts are
   parsed. Each transform that runs after a check is where a benign-looking form becomes a dangerous one.

3. **Choose the representation that survives the check.** Depending on the transform, that is a composed or
   decomposed Unicode form that normalizes to a blocked value, a double-encoded sequence a later decode
   restores, an overlong encoding, or a code point that case-folds onto an allowed or owner value. Decide
   which representation passes the check as written and reconstitutes at the sink.

4. **Consider identity and display confusion.** For identifiers, a mixed-script or homoglyph value can read
   as another user's or a trusted domain, and a case-fold or normalization collision can make two distinct
   registrations compare equal. Determine whether the system compares identifiers on a canonical form and
   restricts scripts, or trusts their apparent distinctness.

5. **Check the canonicalization that actually holds.** The reliable control is to canonicalize input once,
   up front (a single normalization form, full decoding to a fixed depth, a defined case-folding, and host
   or path resolution), and to run every check and every sink on that canonical value, rejecting input that
   changes under a second pass. Determine whether such a control exists or whether checks run ad hoc on
   whatever form arrives.

6. **Confirm and record.** Confirm by submitting the crafted representation and observing the check pass while
   the sink acts on the dangerous form, or two identifiers treated as equal, on an isolated instance with
   benign markers. Kill the lead if input is canonicalized once and both check and sink use that form, if a
   second normalization pass leaves it unchanged, if identifiers are compared canonically with script
   restrictions, or if no transform runs between check and sink. Record the check, the transform, the
   representation, and the sink reached, or set a `kill_reason`.

## Where canonicalization bypass leaks

- **It is an ordering bug.** The vulnerability is transforming after checking; the same transform done before
  the check, once, closes it. Look for the order, not the transform itself.
- **Normalization forms collide.** A composed and a decomposed sequence, or a compatibility character and its
  base, can normalize to the same string, so a denylist checked before normalization is incomplete.
- **Decoding depth matters.** A single decode plus a check on the result is beaten by double encoding; the
  canonical form must be fully decoded to a fixed depth before any check.
- **Case-folding is not lowercasing.** Locale-specific and compatibility case-folding maps characters onto
  each other in ways naive lowercasing misses, enabling identifier and allowlist collisions.
- **Homoglyphs defeat the eye and the compare.** Mixed-script identifiers that look identical to a trusted
  value pass a human check and, without canonical comparison and script limits, a machine one.

## Worked example (a confirm and a kill)

> **Confirm.** A filter blocks a reserved administrative username by exact string comparison on the submitted
> value, and the account store normalizes usernames to a canonical Unicode form before saving and comparing.
> A registration using a decomposed form of the reserved name passes the filter, normalizes to the reserved
> name at the store, and collides with the privileged account on an isolated instance. **Confirmed**
> normalization bypass to identifier collision, `high`, remediation = canonicalize the username to a single
> normalization form and case-folding once at input, run the reserved-name filter and the uniqueness check on
> that canonical value, and reject mixed-script identifiers.
>
> **Kill.** The same registration path normalizes and case-folds the username to one canonical form before
> any check, runs the reserved-name filter and the uniqueness comparison on that form, rejects input that
> changes under a second normalization pass, and restricts identifiers to a single script. A decomposed or
> homoglyph form is canonicalized before the filter sees it and is rejected. **Killed**, `kill_reason` =
> "input is canonicalized once up front and every check and the store use that form, with a second-pass and
> script check, so no alternate representation survives to the sink."

## Rationalizations to reject

- *"We block that value."* -> You block one representation; a composed, decomposed, or encoded form that
  normalizes or decodes back to it passes unless the check runs on the canonical form.
- *"We lowercase the input."* -> Lowercasing is not full case-folding and ignores normalization and encoding;
  distinct code points still fold or normalize together at the sink.
- *"The two usernames are clearly different."* -> They may fold or normalize to the same value at the store,
  or be homoglyphs that read as one; compare on the canonical form, not the raw bytes.
- *"We decode the input."* -> Once is not enough if a later stage decodes again; canonicalize fully to a fixed
  depth before any check, and reject anything that changes on a second pass.
- *"Unicode is an edge case."* -> Identifiers, paths, and hosts are all text, and the transforms that reorder
  check and sink run on every request; this is a routine ordering bug, not an exotic one.

## Executing this in practice

You need each security decision with the representation it inspects, the sink it guards with the
representation that sink acts on, and every normalization, decoding, and case-folding step in between, plus
how identifiers are compared and whether scripts are restricted. For each, decide whether check and sink
share one canonical form or a transform runs between them. Reading the order of canonicalization and checks
settles most leads; submitting a crafted representation and watching the check pass while the sink acts on
the dangerous form settles the rest.

## Related

- `hunting-path-traversal-and-file-access` - path containment checked before decoding or normalization is a
  direct instance of this ordering bug, so the two meet on filesystem sinks.
- `hunting-reflected-and-stored-xss` - a filter bypassed by an encoded or normalized form that reconstitutes
  in the output context is the same check-then-transform failure applied to scripting.
- `auditing-open-redirect-and-forced-navigation` - host allowlists defeated by encoded or homoglyph hosts
  share this canonicalization analysis on the redirect target.
- `hunting-broken-object-level-authorization` - identifier collisions from case-folding or normalization can
  make one principal's identifier match another's, feeding the authorization checks that skill hunts.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the input that passes the check in one form, sink =
  the security decision made on the wrong form, evidence = the check passing while the sink acts on the
  dangerous form, or two identifiers treated as equal, on an isolated instance.
