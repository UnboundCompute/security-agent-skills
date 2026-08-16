---
name: detecting-race-conditions
description: >-
  Find concurrency and time-of-check/time-of-use bugs — TOCTOU, unsynchronized
  shared state, check-then-act, and atomicity violations — by reasoning about
  what state is shared, what can interleave, and where a window opens between a
  check and its use. Use on an authorized source target when the risk is
  ordering, not a single tainted value; when reviewing multithreaded code, shared
  caches/counters, filesystem checks, or "verify then act" sequences (balance
  checks, auth-then-use, dedup guards). Confirms each as an interleaving witness
  and emits the shared finding schema.
license: MIT
---

# Detecting race conditions

Race bugs don't live in one line — they live in the *gap* between two operations
that another actor can slip through. There's no tainted value to trace and no
single sink to grep; you find them by asking what state is shared, what runs
concurrently, and where a window opens between deciding something and acting on
it. This skill covers the recurring shapes and how to confirm a real window.

## When to use

- Multithreaded / async / multiprocess code, or a shared resource (DB row, cache,
  counter, file, in-memory map) touched by concurrent requests.
- A "check then act" sequence where the check's truth can expire before the act:
  balance/quota checks, auth-then-use, uniqueness/dedup guards, file existence
  checks, one-time tokens.

## Scope check

Authorized source only. If you can't name the authorization, stop.

## The shapes and how to confirm each

Confirmation of a race is an **interleaving witness**: two concrete operation
orders where one is safe and the other, achievable by an attacker, is not.

1. **TOCTOU (time-of-check to time-of-use).** State is validated, then used, and
   it can change in between. Classic filesystem form: `access(path)` /
   `stat(path)` then `open(path)` — a symlink swap in the window redirects the
   open. General form: any `check(x); … ; use(x)` where `x` (or what it names) is
   mutable by another actor in the gap. Confirm: identify the shared thing, show a
   writer that can change it between check and use.

2. **Check-then-act on shared state without atomicity.** "If not exists, create";
   "if balance ≥ n, deduct n"; "if not already redeemed, redeem." Two requests
   both pass the check before either acts → double-spend, duplicate creation,
   coupon reuse. Confirm: two concurrent runs both read the pre-act value; neither
   the read+act is atomic (no transaction, lock, `SELECT … FOR UPDATE`, compare-
   and-swap, or unique constraint).

3. **Unsynchronized shared mutable state.** A field/map/counter read and written
   by multiple threads with no lock/atomic. Confirm: two threads reach the same
   memory, at least one writes, no happens-before relationship (no shared lock,
   no atomic, no ownership handoff) orders them.

4. **Atomicity violation across a compound update.** Several fields that must move
   together are updated without a single critical section, so a concurrent reader
   sees a torn, inconsistent state. Confirm: a reader path observes the fields
   mid-update.

5. **Lock misuse.** Right idea, wrong execution: the lock is taken *after* the
   shared access, a different lock guards read vs write, the lock is released
   early, or the critical section doesn't cover the whole check-then-act. Confirm:
   the guarded region doesn't actually span both the decision and the action.

## Where to look

Shared sinks are the anchors: process-global/static mutable state, singleton
caches, ORM objects reused across requests, filesystem paths derived from a check,
and any DB write preceded by an application-level read that decides it. For each,
ask: *who else touches this concurrently, and is the decision+action atomic?*

## Worked example

> **Double-spend via check-then-act.** `redeem(code)`:
> `row = db.get(code); if row.used: reject; … ; row.used = True; db.save(row)`.
> Two requests with the same `code` both read `used == False` before either saves.
> Witness: interleave R1.read, R2.read, R1.save, R2.save → both succeed.
> No transaction, no `SELECT … FOR UPDATE`, no unique/CAS on `used`. **Confirmed
> atomicity/check-then-act race**, severity high, impact = coupon/token reused N
> times. Remediation: atomic conditional update (`UPDATE … SET used=true WHERE
> code=? AND used=false`) and act on affected-row-count.

## Rationalizations to reject

- *"It's guarded by a check right above."* → A check above the act is exactly the
  TOCTOU shape unless the check+act is atomic. The gap is the bug.
- *"There's a lock in the function."* → Does the critical section span *both* the
  decision and the action, on every path, with the *same* lock as the writer?
- *"Requests are basically serial in practice."* → "Basically" is the window.
  Assume concurrent unless the runtime guarantees otherwise.
- *"The DB will sort it out."* → Only with a transaction at the right isolation,
  a constraint, or a conditional update. A plain read-then-write won't.

## Executing this in practice

You need to identify shared state and the concurrent entry points that reach it,
then read the ordering around each access. A code property graph helps you find
every reader/writer of a shared object and every path into a check-then-act; a
concurrency-aware analyzer or a targeted stress/PoC harness corroborates a real
window. State each finding as an interleaving witness and emit per the
[finding schema](../../FINDING-SCHEMA.md), with the two operation orders in
`evidence` and `reachable = conditional` on the interleaving.

## Related

- `auditing-guard-gaps` — when only *some* paths take the lock/transaction.
- `hunting-bugs-with-a-code-graph` — the master loop that routes these here.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md).
