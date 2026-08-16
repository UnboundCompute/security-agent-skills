---
name: detecting-memory-safety-bugs
description: >-
  Find memory-safety bugs in C/C++ and other unmanaged code - use-after-free,
  double-free, out-of-bounds read/write, uninitialized use, and NULL deref - by
  reasoning about object lifetime and buffer bounds along real code paths. Use on
  an authorized source target when a candidate catalog does NOT model these
  temporal/lifetime classes (most don't), so a keyword or sink scan will miss
  them; when reviewing allocators, parsers, buffer handling, or refcounting. Pairs
  the lifetime/bounds reasoning with source→sink confirmation and the shared
  finding schema.
license: MIT
---

# Detecting memory-safety bugs

Temporal and spatial memory bugs are the ones a sink-based catalog usually
*doesn't* model: there's no single "dangerous function" to grep - the bug is a
relationship between a pointer's lifetime and its use, or between an index and a
bound, spread across code paths. So you hunt them by reasoning about lifetime and
bounds, not by matching a call. This skill covers the five workhorse classes and
how to confirm each.

## When to use

- The target is C/C++/Rust-unsafe/CGo or any unmanaged memory, and you want the
  classes a catalog leaves out (`hunting-bugs-with-a-code-graph` flags these as
  out-of-catalog and sends you here).
- You're reviewing allocators, parsers, serializers, ring buffers, refcounting,
  or anything doing pointer arithmetic.

## Scope check

Authorized source only. If you can't name the authorization, stop.

## The five classes and how to confirm each

For every candidate, the confirmation is a *path*: an allocation/definition site,
the operation that changes its state, and the use - read the source at each.

1. **Use-after-free (UAF).** A pointer is used after its object is freed. Hunt:
   for each `free`/`delete`/refcount-drop, ask *what still holds this pointer* and
   *can any path reach a use after this point* - including aliases stored in
   structs, callbacks, and error paths. Confirm: a live path free → … → deref with
   no reassignment in between. Watch the classic shapes: free-in-a-loop then use,
   free in an error branch then fallthrough use, and a cached pointer that
   outlives the object.

2. **Double-free.** The same allocation freed twice. Hunt: two frees reachable in
   sequence on some path without a nulling/reassignment between them; ownership
   handed to a callee that also frees; error paths that free then fall into a
   cleanup that frees again. Confirm: both frees hit the same object on one path.

3. **Out-of-bounds read/write.** An index or pointer crosses a buffer's bound.
   Hunt: every buffer access where the index/length is attacker-influenced or
   derived from a separate field (length prefixes are a hotspot). Confirm: trace
   the index/length source and the buffer's true size; the bug is any path where
   `index/len` can exceed `size` - off-by-one at `<= size`, unchecked `memcpy`
   length, integer-truncated length, signed/unsigned confusion at the check.

4. **Uninitialized use.** A value is read before it's written. Hunt: stack structs
   and locals used on a path that skipped their initialization (an early `goto`,
   an error branch, a partially-filled struct). Confirm: a path from declaration
   to use with no write on that path.

5. **NULL / invalid-pointer deref.** A pointer that can be NULL (or an error
   sentinel) is dereferenced without a check. Hunt: allocation and lookup returns
   that can fail; dereferences of a parameter documented as nullable. Confirm: a
   path where the returning call can yield NULL and the deref has no guard between.

## Integer overflow feeds all of the above

Overflow is rarely the *final* bug - it's the step that produces a bad
length/index/size. Treat `a * b`, `a + b`, and truncating casts on
attacker-influenced sizes as candidates *because* they flow into an allocation or
a bound. Trace overflow → size/index → memory op; report the memory bug with the
overflow as its cause.

## Worked example

> **UAF via error path.** In `parse_record`: `rec = alloc(); if (read(rec) < 0) {
> free(rec); }  … use(rec->field);` - the `free` is in the error branch but the
> function falls through to `use(rec->field)` without returning. Path: alloc →
> free (error branch) → deref (fallthrough). **Confirmed UAF**; `reachable =
> conditional` (read returns < 0); impact = attacker-triggered malformed record
> frees then dereferences. Remediation: `return` after the free, or restructure
> cleanup with a single exit.

## Rationalizations to reject

- *"There's no dangerous function call, so it's memory-safe."* → These classes
  have no single sink. Reason about lifetime and bounds, not calls.
- *"The length is checked."* → Checked *where*, against the *true* size, before
  *every* path to the access, and without integer truncation? Verify all four.
- *"It's freed once in the code I can see."* → Ownership may free again in a
  callee or a cleanup path. Follow the pointer, including aliases.
- *"The compiler/sanitizer would catch it."* → Only on inputs the test suite
  exercised. Static path reasoning finds the untested path.

## Executing this in practice

You need to follow a pointer's aliases and a value's flow across functions, and
read the exact body at each site. A code property graph with points-to/alias
information makes "what else holds this pointer" answerable; a memory-safety
static analyzer (or the compiler's sanitizers on targeted inputs) corroborates;
manual path reading closes it. Confirm every class as a concrete path, then emit
per the [finding schema](../../FINDING-SCHEMA.md) with the alloc/free/use hops in
`path`.

## Related

- `adjudicating-taint-paths` - for the attacker-input side (who controls the
  length/index).
- `hunting-bugs-with-a-code-graph` - the master loop that routes these here.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md).
