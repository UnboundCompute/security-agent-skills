---
name: hunting-redos-and-complexity-dos
description: >-
  Hunt single-request denial of service from super-linear work: untrusted input reaching a
  backtracking regular expression, a quadratic or worse algorithm, or a hash-keyed structure
  with attacker-chosen keys, with no size or complexity guard between. Covers regular
  expressions with nested or ambiguous quantifiers that explode on a crafted non-matching
  string, accidental nested scans and unbounded parsers over attacker-sized input, repeated
  string building in a loop, and hash flooding where predictable unseeded keys turn constant-time
  lookups quadratic. Use when reviewing code where request-controlled strings, collections, or
  numbers reach an expensive operation and the cost can grow faster than the input. The input's
  size or content is the source, the super-linear operation is the sink, and the missing bound is
  the bug.
license: MIT
---

# Hunting ReDoS and algorithmic-complexity denial of service: when one request burns the CPU

Most denial of service is discussed as flooding: many requests exhausting a pool. The quieter and
more dangerous kind is a single small request that pins a CPU core for seconds or minutes, because
somewhere the work grows faster than the input. A regular expression with an ambiguous quantifier,
a loop that is accidentally quadratic, a parser with no depth limit, or a hash table fed
attacker-chosen keys all turn a few kilobytes of input into a stall that a timeout may not even
interrupt. You find these by tracing untrusted input to any operation whose cost is super-linear
and asking whether anything bounds the work before it runs.

## When to use

- Untrusted input reaches a regular expression, a parser, a sort or dedup, or a hash-keyed structure.
- A single small request could pin a CPU core or stall the event loop, not just exhaust connections.
- Input size, nesting depth, or total work is not bounded before the expensive operation runs.

## Scope check

Drive worst-case input only against systems you own or are authorized to test, and coordinate:
a confirmed complexity bug can take a shared service offline. Measure on a target and load where a
stall is acceptable. If you can't name the authorization, stop.

## The loop

1. **Map untrusted input to expensive operations.** Inventory where request-controlled strings,
   collections, or numbers reach a regular expression, a recursive or backtracking parser, a nested
   loop over input, a sort or dedup, or a hash-keyed collection. These are the operations whose cost
   can grow faster than linearly in the size of the input, so these are where a small request buys
   disproportionate work.

2. **Find the super-linear regular expressions.** Read each expression that touches untrusted input
   for the backtracking shapes: a quantified group inside another quantifier such as `(a+)+`,
   overlapping alternation under a quantifier such as `(a|a)*`, and a quantified class followed by a
   character the class can also match. On an input that almost matches and then fails at the end, the
   engine explores exponentially or polynomially many paths. A crafted string of a few kilobytes can
   run for seconds.

3. **Find the quadratic and worse algorithms.** Look for cost that is super-linear in input size with
   no cap: an accidental nested scan over an attacker-sized list, string building by repeated
   concatenation in a loop, a parser with no depth or length limit, or repeated membership checks
   against a growing list. The attacker controls the size, so the attacker controls the cost.

4. **Find the hash-flooding sinks.** Look for maps, sets, dedup, or grouping built from
   attacker-chosen keys. If the hash function is predictable and unseeded, an attacker who submits
   many colliding keys collapses average constant-time operations into worst-case linear ones, making
   the whole build quadratic in the number of entries they send.

5. **Check for a guard between source and sink.** Determine whether input length, collection size,
   nesting depth, or a total-work budget is enforced before the expensive operation, and whether the
   operation runs under a timeout that actually preempts it. A length check that runs after the regex
   has already backtracked, or a timeout that cannot interrupt a busy CPU loop, is not a guard.

6. **Confirm and record.** Confirm by driving the operation with a crafted worst-case input in scope
   and measuring the time or CPU it burns against a normal input of the same size. Kill the lead if
   input is tightly bounded before the operation, the expressions carry no ambiguous backtracking
   shape, the algorithms are linear or capped, hash keys are seeded or not attacker-chosen, and an
   enforced timeout preempts the work. Record with the input, the operation, and the observed cost.

## Where complexity denial of service leaks

- **A backtracking expression is a CPU bomb with a tiny fuse.** Cost is exponential in input length
  for the worst shapes, so a few kilobytes is enough. The danger is the shape of the expression, not
  the size of the input.
- **The size check often runs on the wrong side of the work.** A length limit applied after the
  expensive call, or only to a different field, does nothing. The bound has to precede the sink.
- **A timeout that does not preempt is decoration.** A wall-clock deadline that cannot interrupt a
  synchronous CPU loop lets the work run to completion anyway while the deadline quietly passes.
- **Unseeded hashing lets the attacker pick your worst case.** Predictable keys turn a hash structure
  into a linked list; the defense is a per-process seed, not a bigger table.
- **Linear-looking code hides a quadratic loop.** A membership check inside a loop, or concatenation
  that copies the whole string each time, is super-linear even though nothing looks nested.

## Worked example (a confirm and a kill)

> **Confirm.** A profile field is validated by a regular expression with a nested quantifier applied
> before any length limit. A submitted value of a few kilobytes of a repeated character followed by a
> non-matching byte makes the request thread run for over thirty seconds at full CPU; a handful of
> concurrent submissions saturate every worker. **Confirmed** regular-expression denial of service,
> `high`, remediation = bound the field length before matching, rewrite the expression to remove the
> ambiguous quantifier or use a non-backtracking engine, and enforce a preempting time budget.
>
> **Kill.** The same field is capped to a small maximum length before it is ever matched, the
> expression has no nested or overlapping quantifier, the surrounding parsing is linear with a depth
> limit, hash-keyed structures are seeded per process, and the request runs under a budget that
> interrupts it. Worst-case input finishes in the same time as normal input. **Killed**, `kill_reason`
> = "input length bounded before a non-ambiguous expression, linear capped parsing, seeded hashing,
> preempting time budget; crafted input shows no super-linear cost."

## Rationalizations to reject

- *"It is just input validation."* → A validating expression is exactly where the bomb hides, because
  it runs on attacker input before anything else. Validation is a sink here, not a safeguard.
- *"We have a request timeout."* → Does it preempt a busy CPU loop, or only fire once the work returns?
  A deadline that cannot interrupt the work is not a mitigation.
- *"The input is only a few kilobytes."* → For an exponential-backtracking shape, a few kilobytes is
  far more than enough. Size is not the protection; the algorithm is the problem.
- *"A hash map is constant time."* → On average, with a good seed. With predictable unseeded keys an
  attacker forces the worst case and it becomes linear per operation.
- *"No one would send that input."* → It costs the attacker one small request to take a core offline.
  Cheap for them, expensive for you, is the whole point.

## Executing this in practice

You need the set of places untrusted input reaches a regular expression, a parser, a
super-linear algorithm, or a hash-keyed structure, the exact expressions and loops involved, and the
guards (length, depth, size, time budget, hash seeding) that run before each. For each, ask what a
crafted worst-case input costs and whether anything bounds it in time. Reading the code tells you the
shape; measuring a crafted input against a normal one of the same size tells you whether the cost is
super-linear in practice.

## Related

- `auditing-graphql-attack-surface` - query depth and cost amplification is the same single-request
  denial of service one layer up, at the query planner.
- `hunting-business-logic-flaws` - an unbounded expensive operation is a resource the logic forgot to
  meter; both hunt for a missing limit.
- `detecting-race-conditions` - a sibling resource-and-timing class where the bug is in how work is
  scheduled rather than how much of it runs.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the input's size or content, sink = the
  super-linear operation, evidence = the crafted input and the CPU or time it burns.
