---
name: hunting-php-object-injection-pop-chains
description: >-
  Hunt PHP object injection where untrusted input reaches unserialize or a framework unserializer and a
  reachable class carries a magic method that fires during or after reconstruction. Covers native
  unserialize on request data, cookies, or cache entries, phar deserialization triggered by filesystem
  functions on an attacker-controlled path, and property-oriented programming chains through __wakeup,
  __destruct, __toString, and __call that reach a file write, a command, or an SQL sink. Use when a PHP
  app deserializes data it did not produce and application or library classes define magic methods with
  side effects. The untrusted serialized string is the source, the unserialize or phar trigger is the
  sink, and the magic-method chain to a dangerous call is the bug.
license: MIT
---

# Hunting PHP object injection and POP chains: when a string rebuilds into an attack

PHP object injection follows the same shape as Java gadget chains but through a different trigger. When
`unserialize` runs on attacker-controlled data, it reconstructs arbitrary objects of any class already
loaded, and PHP's magic methods, `__wakeup`, `__destruct`, `__toString`, `__call`, run automatically
during or shortly after reconstruction. A property-oriented programming chain stitches these methods
across ordinary classes so that reconstruction alone drives a file write, a system command, or a query.
The same trigger hides inside `phar://` stream handling: a filesystem function given an attacker-chosen
path deserializes the archive's metadata without any explicit `unserialize` call. You find these by
locating every deserialization trigger, explicit or implicit, and asking which loaded classes carry a
magic method that starts a chain.

## When to use

- A PHP application calls `unserialize` on request data, a cookie, a cache entry, or a stored value.
- A filesystem function receives an attacker-influenced path that could carry a `phar://` wrapper.
- Application or library classes define magic methods with side effects that a chain could reach.

## Scope check

Test object injection only against applications you own or are authorized to assess, on non-production
data. A confirming chain runs a side effect on the target, a file write or a command, so treat each proof
as a live intrusion within the authorized scope. If you can't name the authorization, stop.

## The loop

1. **Establish the deserialization triggers first.** Inventory every `unserialize` on untrusted data and
   every filesystem call, `file_exists`, `fopen`, `md5_file`, `getimagesize`, and their siblings, that
   takes a path an attacker can steer into a `phar://` wrapper. This is the false-positive killer: a
   magic-method chain is inert unless one of these triggers fires on attacker input. Name the triggers,
   then look for chains.

2. **Trace untrusted input into each trigger.** Follow request parameters, cookies, headers, cache
   values, and database fields that an attacker previously wrote into the `unserialize` call, and follow
   attacker-controlled paths into the filesystem functions. A serialized value the application produced
   and never exposes is not a source; anything a client or an upstream writer can set is.

3. **Enumerate reachable magic-method starters.** Read the loaded application and library classes for
   `__wakeup`, `__destruct`, `__toString`, and `__call` that do something exploitable: write a file,
   invoke a command, build a query, or call another object's method with attacker-controlled arguments.
   These are the chain's entry and exit points.

4. **Assemble the chain from starter to sink.** A usable POP chain begins at a magic method that fires on
   reconstruction and threads controlled property values through further calls until it reaches a
   dangerous operation. Confirm the attacker controls the properties at each hop and that the terminal
   call, a write, an exec, or a query, is actually reachable with values they set.

5. **Check the defenses that actually stop it.** Passing `allowed_classes` to `unserialize`, or replacing
   native serialization with a data-only format such as JSON, removes object injection at the source. A
   type check after reconstruction does not, because the magic method already fired. Determine whether the
   trigger restricts classes, whether phar handling is disabled or gated, and whether the format is
   data-only.

6. **Confirm and record.** Confirm by supplying a benign in-scope object whose chain proves it reaches the
   terminal call, a controlled file write or an out-of-band signal, without a destructive payload. Kill
   the lead if no untrusted data reaches a deserialization trigger, if `unserialize` restricts allowed
   classes or the format is data-only, or if no reachable magic method starts a chain to a sink. Record
   the source, the trigger, the chain, and the terminal call.

## Where PHP object injection leaks

- **`phar://` deserializes with no unserialize in sight.** A filesystem function on an attacker path
  triggers metadata deserialization silently; grepping only for `unserialize` misses it.
- **The chain is built from library classes.** A framework or a dependency supplies the magic methods;
  auditing only application classes leaves the real gadgets unseen.
- **Serialized state travels in cookies and caches.** A serialized value in a cookie or on a shared cache
  is attacker-writable, so it is an untrusted source even though the app wrote the first copy.
- **A type check after reconstruction is too late.** `__wakeup` and `__destruct` run during and after
  reconstruction, before any later validation the code performs.
- **`__toString` fires from unexpected places.** Any later string coercion of a reconstructed object, in
  logging or concatenation, can launch a chain that reconstruction alone did not.

## Worked example (a confirm and a kill)

> **Confirm.** A caching layer stores serialized objects and reads them back with `unserialize` and no
> `allowed_classes`. A reachable library class defines a `__destruct` that writes a property to a path
> built from another property. A crafted serialized value reconstructs into a chain that writes a
> controlled file to a web-served directory, proving reachability to the write. **Confirmed** object
> injection to arbitrary file write, `critical`, remediation = pass `allowed_classes: false` (or an
> explicit small allowlist) to every `unserialize` on stored data, and move the cache format to JSON.
>
> **Kill.** Every `unserialize` on untrusted input passes an explicit allowlist limited to two plain data
> classes with no magic methods, and phar handling is disabled in the runtime configuration. No crafted
> string reconstructs a chain-bearing class. **Killed**, `kill_reason` = "unserialize restricted to
> magic-method-free data classes and phar deserialization disabled; no attacker object reaches a starter."

## Rationalizations to reject

- *"We do not call unserialize on user input."* → A filesystem function on an attacker path deserializes a
  phar with no explicit call. Inventory those triggers too.
- *"The gadget classes are in a library, not our code."* → Loaded library classes are reachable by the
  chain; that is where most POP gadgets live.
- *"We validate the object after unserialize."* → `__wakeup` and `__destruct` already ran. Validation
  after reconstruction cannot undo a fired chain.
- *"It is stored server-side, so it is trusted."* → A cache entry or a database field an attacker once
  wrote is untrusted on read; storage is not authentication.
- *"There is no obvious sink in the magic method."* → A `__toString` or `__call` that forwards controlled
  values into another object can still terminate in a write, a query, or a command downstream.

## Executing this in practice

You need every deserialization trigger, explicit `unserialize` and implicit phar paths, the untrusted
inputs that reach each, the loaded classes with exploitable magic methods, and the class restriction at
each trigger if any. For each trigger, ask whether attacker data arrives, whether a reachable chain runs
from a magic method to a sink, and whether an allowlist stands between. Reading the trigger and the chain
shows intent; a benign controlled side effect shows the chain fires.

## Related

- `hunting-java-deserialization-gadget-chains` - the same property-oriented idea in Java, triggered by
  readObject rather than a magic method.
- `hunting-unsafe-archive-extraction` - phar handling overlaps with archive trust; both turn an
  attacker-controlled path or archive into a side effect.
- `hunting-server-side-prototype-pollution` - another reconstruction-of-attacker-shaped-object class, in
  JavaScript, where merged keys instead of magic methods drive the effect.
- `adjudicating-taint-paths` - use it to confirm untrusted data reaches an unserialize call or a phar path
  through framework indirection.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the untrusted serialized string, sink = the
  unserialize or phar trigger, evidence = the magic-method chain reaching a dangerous call.
