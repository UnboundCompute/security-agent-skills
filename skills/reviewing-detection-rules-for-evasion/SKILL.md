---
name: reviewing-detection-rules-for-evasion
description: >-
  Stress detection-as-code rules the way an attacker who has read them would: a rule keyed on one
  literal spelling of an action that a casing, quoting, whitespace, path, flag-ordering, or encoding
  variant slips past, a left-anchored or misplaced-wildcard match defeated by added noise, an
  exclusion or allowlist keyed on a field the attacker sets, and a rule over telemetry the log source
  never actually emits. Covers matching the spelling instead of the behavior, anchor and wildcard
  placement, self-defeating negation, and coverage gaps in the underlying events. Use when reviewing
  or threat-modeling detection content for brittleness rather than authoring it. The attacker-set
  field is the source, the rule's match decision is the guard, and a malicious event that performs
  the action yet does not match is the finding.
license: MIT
---

# Reviewing detection rules for evasion: whether the rule survives an attacker who reads it

A detection rule is a guard, and like any guard it is only as strong as the exactness of the thing it
checks against what the attacker can vary. Detection content is often written to match one observed
spelling of an attack: a specific command line, a known tool name, a particular flag. An attacker who
can read the rule, or simply reason about it, expresses the same action a different way and walks past.
This review treats a rule the way an attacker treats a filter: find the field it trusts, find the variant
it does not cover, and produce an event that does the malicious thing without tripping the match. The
attacker-controlled field is the source, the rule's match logic is the guard, and a bypass is the bug.

## When to use

- You maintain detection-as-code rules and want to know which are brittle before an attacker does.
- Rules match on attacker-controlled fields: command lines, arguments, user-agents, paths, names.
- You are reviewing or threat-modeling existing detections, not writing new ones.

## Scope check

Test evasion only against detections, telemetry, and systems you own or are authorized to assess, and
generate malicious-looking events only where that is expected and coordinated. A realistic bypass event
can trigger a real response. If you can't name the authorization, stop.

## The loop

1. **Inventory each rule and the field it trusts.** For every detection rule, identify the fields it
   matches on and which of those an attacker controls freely: a process command line and its arguments,
   environment values, a user-agent, a file path or name. A rule keyed on an attacker-controlled field is
   only as strong as how exactly it pins that field against every equivalent form.

2. **Test literal and encoding variants.** For each match, ask whether the same malicious action can be
   expressed a way the rule does not cover: alternate casing, quoting, extra whitespace, an equivalent
   path or name form, reordered flags, an encoded argument, or a different tool that has the same effect.
   A rule that matches one literal spelling misses every synonym of it.

3. **Test the anchor and wildcard placement.** A match anchored to the start, or requiring a fixed prefix,
   is bypassed by prepending harmless noise; a wildcard in the wrong position widens or narrows the match
   in ways the author did not intend. Determine what an attacker can add before, after, or inside the
   matched region without changing the outcome of their action.

4. **Test negation and allowlist fields.** A rule that suppresses events where a field equals a known-good
   value is defeated if the attacker can set that field. An exclusion keyed on a user-agent, a parent
   process name, a path, or any attacker-controlled datum is a self-service bypass. Find every exclusion
   and ask who controls the excluded field.

5. **Check the telemetry actually carries the field.** A flawless rule over an event the log source does
   not emit, or a field it does not populate, never fires. Determine whether the events the rule depends on
   are collected at all, with the needed fields and volume, from the systems in scope. A silent coverage
   gap looks identical to a quiet environment.

6. **Confirm and record.** Confirm by producing an event that performs the malicious action yet does not
   match the rule, a variant spelling, an added prefix, or a set exclusion field, or by showing the
   required telemetry is absent. Kill the lead if the rule matches an invariant of the behavior rather than
   a spelling, covers casing, encoding, and ordering variants, uses no attacker-settable exclusion, and the
   telemetry reliably carries the field. Record with the rule, the bypass, and the action that still runs.

## Where detection rules leak

- **A rule keyed on a spelling dies to a synonym.** Matching one literal command or tool name misses every
  equivalent form; pin the invariant behavior, not the observed string.
- **A left-anchored match invites a prefix.** If prepending noise breaks the match without breaking the
  attack, the anchor is in the wrong place.
- **An exclusion on attacker-controlled data is a self-service bypass.** Suppressing on a field the attacker
  sets hands them the off switch for the alert.
- **A rule over uncollected telemetry never fires.** Coverage of the underlying event is a precondition; a
  perfect match over a missing field is a blind spot dressed as a detection.
- **Matching the tool misses the next tool.** Detections tied to one utility fall to a reimplementation with
  the same effect; detect the effect where you can.

## Worked example (a confirm and a kill)

> **Confirm.** A rule flags a sensitive administrative command by matching its exact lowercase invocation
> string, anchored at the start of the command line, and suppresses events whose user-agent equals an
> internal tool's string. An attacker runs the same command with mixed casing and a leading no-op, and sets
> the excluded user-agent, evading the rule entirely while performing the action. **Confirmed** detection
> bypass on a privileged action, `high`, remediation = match a case-insensitive invariant of the command
> unanchored, cover argument and ordering variants, and remove the attacker-settable user-agent exclusion in
> favor of a trait the attacker cannot forge.
>
> **Kill.** The rule matches a case-insensitive, position-independent invariant of the action, accounts for
> quoting, whitespace, and flag-ordering variants, carries no exclusion keyed on attacker-controlled data,
> and the telemetry it needs is confirmed collected with the required fields from all in-scope systems.
> Variant, prefix, and set-field events all still match. **Killed**, `kill_reason` = "rule pins a behavioral
> invariant, covers spelling and encoding variants, uses no attacker-settable exclusion, and required
> telemetry is present."

## Rationalizations to reject

- *"The rule caught it in the test."* → It caught the one spelling you tested. The question is every
  equivalent spelling an attacker can use instead.
- *"We exclude our own tooling to cut noise."* → If the attacker can set the excluded field, the exclusion
  is their bypass. Suppress on something they cannot forge.
- *"It is anchored, so it is precise."* → Anchored to a position the attacker can pad. Precision on a movable
  boundary is not precision.
- *"We match the known tool."* → The next tool with the same effect is not the known one. Detect the effect,
  not the utility, wherever the telemetry allows.
- *"The rule is fine, so we are covered."* → Only if the events it reads are actually collected. A rule over
  missing telemetry never fires and never tells you so.

## Executing this in practice

You need the set of detection rules, the fields each matches and excludes on and which of those an attacker
controls, the variant space for each matched action, and confirmation that the underlying telemetry is
collected. For each rule, ask what an attacker who has read it would change to keep the action and lose the
match. Reading the rule shows what it intends to catch; constructing an equivalent event that does not match
shows what it actually catches.

## Related

- `auditing-security-logging-completeness` - the upstream half: a rule can only match events the application
  and hosts actually emit, so close coverage gaps there first.
- `auditing-guard-gaps` - the same exactness question applied to application guards; this skill applies it to
  detection content instead of code.
- `hunting-bug-variants` - variant thinking is the shared engine: one known case implies a family of
  equivalent forms to enumerate.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the attacker-set field, sink = the rule's match
  decision, evidence = the malicious event that performs the action without matching.
