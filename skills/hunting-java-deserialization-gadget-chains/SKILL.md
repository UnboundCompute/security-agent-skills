---
name: hunting-java-deserialization-gadget-chains
description: >-
  Hunt Java deserialization that turns an untrusted byte stream into code execution: attacker-controlled
  data reaching readObject, an ObjectInputStream, or a framework endpoint that deserializes, with a
  gadget on the classpath whose readObject or finalizer drives a property-oriented chain to a dangerous
  call. Covers native serialization, JNDI lookups reached through deserialized objects, and framework
  entry points that accept a serialized object over HTTP, a message queue, a cache, or a cookie. Use when
  a service reads serialized Java objects it did not produce and reachable library versions carry a known
  gadget. The untrusted serialized stream is the source, the deserialization call is the sink, and the
  gadget chain from readObject to a runtime or naming call is the bug.
license: MIT
---

# Hunting Java deserialization gadget chains: when reading an object runs code

Java native serialization was never a trust boundary, but it is treated as one every time a service
accepts a serialized object from a client, a queue, a cache, or a cookie and calls `readObject`. The
danger is not the deserialization call by itself; it is the combination of that call with a gadget, an
already-present library class whose own `readObject`, `readResolve`, or finalizer performs a side effect
an attacker can steer. Chained together, ordinary library classes reconstruct into a sequence that ends
in a runtime command, a naming lookup, or a template evaluation. You find these by locating every place
untrusted bytes reach a deserializer and then asking which reachable library versions on the classpath
carry a chain that fires on reconstruction.

## When to use

- A service reads serialized Java objects from a client, a message queue, a cache, a file, or a cookie.
- Untrusted input reaches `readObject`, an `ObjectInputStream`, or a framework method that deserializes.
- Reachable library versions on the classpath are known to carry a deserialization gadget.

## Scope check

Test deserialization only against services and applications you own or are authorized to assess, on
non-production infrastructure. A confirming payload runs code on the target, so treat every proof as a
live intrusion and stay inside the authorized scope. If you can't name the authorization, stop.

## The loop

1. **Establish the deserialization sinks first.** Before chasing gadgets, inventory every call that turns
   bytes into objects: `ObjectInputStream.readObject`, `readUnshared`, framework converters that accept a
   serialized object, and any wrapper that hides one. This is the false-positive killer: a gadget on the
   classpath is harmless if no untrusted stream ever reaches a deserializer. Name the sinks, then work
   backward.

2. **Trace untrusted bytes into each sink.** Follow request bodies, headers, cookies, queue messages,
   cache entries, and uploaded files into the deserializer. A stream the service itself produced and
   never exposes is not a source; a stream any client, or any producer on a shared bus, can supply is.
   Confirm the bytes crossing the boundary are attacker-influenced, not merely internal.

3. **Enumerate reachable gadget-bearing libraries.** Read the dependency set and the effective versions
   after resolution. A gadget only fires if its class is on the runtime classpath at a vulnerable version;
   a fixed version or an absent library kills the chain. Match the present libraries against known
   property-oriented chains rather than assuming any deserializer is exploitable.

4. **Confirm the chain fires on reconstruction.** A usable gadget performs its side effect during
   deserialization, in `readObject`, `readResolve`, a finalizer, or a lazily evaluated map or comparator,
   not only when the application later uses the object. Read the gadget's reconstruction path and confirm
   the attacker controls the value that reaches the terminal call, a runtime exec, a naming lookup, or a
   template evaluation.

5. **Check the defenses that actually stop it.** A look-ahead filter or an allowlist of permitted classes
   at the `ObjectInputStream` defeats most chains; a denylist of known-bad names does not, because a new
   gadget sidesteps it. Determine whether the sink enforces a class allowlist, whether the format was
   moved off native serialization entirely, and whether the reachable versions are patched.

6. **Confirm and record.** Confirm by delivering a benign, in-scope object that proves reconstruction
   reaches the terminal call, an out-of-band callback or a controlled side effect, without deploying a
   destructive payload. Kill the lead if no untrusted stream reaches a deserializer, if a class allowlist
   or look-ahead filter constrains reconstruction to safe types, or if every gadget-bearing library is at
   a patched version. Record the source, the sink, the specific chain, and the terminal call.

## Where Java deserialization leaks

- **The deserializer is the boundary, and it is invisible.** A single `readObject` on an untrusted stream
  is the whole vulnerability; the surrounding code looks like ordinary object handling.
- **The gadget is library code, not application code.** Teams audit their own classes and miss that a
  transitive dependency ships the chain; the effective resolved version is what matters.
- **A denylist of class names ages badly.** Blocking today's known gadgets does nothing against the next
  one; only an allowlist of expected classes holds.
- **Serialized state hides in cookies and caches.** A serialized session object in a cookie, or a cache
  value on a shared store, is an untrusted stream the moment an attacker can write it.
- **JNDI turns a deserialized field into remote code.** A gadget that performs a naming lookup on an
  attacker-controlled name reaches a remote factory, so the chain need not end in a local exec.

## Worked example (a confirm and a kill)

> **Confirm.** An internal service consumes task messages from a shared queue and calls `readObject` on
> each message body with no class filter. A reachable library version on the classpath carries a
> property-oriented chain whose comparator evaluates a template during reconstruction. A crafted message
> reconstructs into a chain that performs an out-of-band lookup, proving reachability to the terminal
> call. **Confirmed** untrusted deserialization to remote code execution, `critical`, remediation =
> install a look-ahead class allowlist at the `ObjectInputStream` limited to the expected message types,
> and move the message format off native serialization to a data-only codec.
>
> **Kill.** The endpoint deserializes with a resolver that permits only three application message classes
> and rejects everything else, and the two gadget-bearing libraries are at patched versions. A crafted
> stream of any known chain is refused before reconstruction. **Killed**, `kill_reason` = "class allowlist
> at the deserializer confines reconstruction to expected types and gadget libraries are patched; no
> attacker-supplied class reaches readObject."

## Rationalizations to reject

- *"We only deserialize our own objects."* → The stream is attacker-writable if it arrives in a cookie, a
  queue message, or a cache entry; provenance is what you must prove, not assume.
- *"There is no gadget in our code."* → The gadget lives in a dependency. Enumerate the resolved classpath,
  not just the first-party classes.
- *"We block the known dangerous classes."* → A denylist is bypassed by the next chain. Only an allowlist
  of expected classes is durable.
- *"The object is only used later, safely."* → A real gadget fires during reconstruction, before the
  application ever touches the object.
- *"It is behind the firewall."* → A shared queue, cache, or internal caller is still an untrusted producer;
  internal placement is not authentication.

## Executing this in practice

You need every deserialization sink, the untrusted streams that reach each one, the resolved classpath
with effective library versions, and the class-filtering defense at each sink if any. For each sink, ask
whether attacker bytes arrive, whether a reachable gadget chain exists at a vulnerable version, and
whether an allowlist stands between. Reading the sink and the gadget's reconstruction path shows the
intent; a benign out-of-band proof shows the chain actually fires.

## Related

- `hunting-php-object-injection-pop-chains` - the same property-oriented chain idea in PHP, where magic
  methods stand in for readObject as the reconstruction trigger.
- `hunting-dotnet-deserialization-type-injection` - the .NET sibling, where an attacker-chosen type name
  drives the chain rather than a native serialized stream.
- `adjudicating-dependency-cve-reachability` - use it to prove a gadget-bearing library version is
  actually on the runtime classpath and reachable, not merely declared.
- `adjudicating-taint-paths` - use it to confirm untrusted bytes reach the deserializer through wrappers
  and framework indirection.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the untrusted serialized stream, sink = the
  deserialization call, evidence = the gadget chain reaching a terminal runtime or naming call.
