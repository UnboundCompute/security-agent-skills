---
name: hunting-dotnet-deserialization-type-injection
description: >-
  Hunt .NET deserialization where untrusted input reaches a formatter that resolves the type from the
  data itself: BinaryFormatter, SoapFormatter, NetDataContractSerializer, LosFormatter or ObjectStateFormatter
  on ViewState, or Json.NET and similar with type-name handling enabled. Covers formatters that instantiate
  an attacker-named type and drive a gadget through a set accessor, a callback, or a converter to a command,
  a process start, or a file operation. Use when a service reads serialized .NET objects it did not produce
  and the formatter honors an embedded or annotated type name. The untrusted serialized payload is the
  source, the type-resolving formatter is the sink, and the attacker-chosen type driving a gadget to a
  dangerous call is the bug.
license: MIT
---

# Hunting .NET deserialization and type injection: when the payload picks the type

.NET deserialization is dangerous for a specific reason: several formatters read the concrete type to
instantiate from the serialized data, not from the declared target. When `BinaryFormatter`,
`SoapFormatter`, `NetDataContractSerializer`, the ViewState formatters, or a JSON serializer with
type-name handling turned on processes attacker data, the attacker chooses which class gets constructed.
That lets them instantiate a gadget type whose property setters, deserialization callbacks, or type
converters perform a side effect, chaining to a process start, a command, or a file operation. The fix is
almost always to stop resolving types from the data. You find these by locating every formatter that
honors an embedded or annotated type and asking whether untrusted bytes reach it.

## When to use

- A service reads serialized .NET objects from a request, ViewState, a cookie, a queue, or a cache.
- A formatter in use resolves the type from the payload or has type-name handling enabled.
- Reachable assemblies carry a gadget type whose construction or callback has an exploitable side effect.

## Scope check

Test deserialization only against services you own or are authorized to assess, on non-production
infrastructure. A confirming payload constructs an attacker-chosen type and can run code, so treat every
proof as a live intrusion inside the authorized scope. If you can't name the authorization, stop.

## The loop

1. **Establish the type-resolving sinks first.** Inventory every formatter that reads the type from the
   data: `BinaryFormatter`, `SoapFormatter`, `NetDataContractSerializer`, `LosFormatter` and
   `ObjectStateFormatter` behind ViewState, and any JSON or XML serializer configured with type-name
   handling or a permissive binder. This is the false-positive killer: a formatter that deserializes to a
   fixed known type with no type resolution is not injectable. Name the resolving sinks first.

2. **Trace untrusted bytes into each sink.** Follow request bodies, ViewState fields, cookies, headers,
   queue messages, and cache values into the formatter. ViewState is a common source when the machine key
   is weak or absent, letting an attacker forge a signed payload. Confirm the bytes reaching the formatter
   are attacker-influenced.

3. **Confirm the formatter honors an attacker type.** For binary and SOAP formatters this is inherent; for
   JSON and XML it depends on the type-name-handling setting and the binder. A serializer restricted to a
   known type or guarded by a strict binder that rejects unexpected types does not let the attacker pick
   the class. Read the exact configuration, not the default.

4. **Locate a reachable gadget type.** A usable gadget is a type on the loaded assemblies whose
   construction path, a property setter, an `OnDeserialized` callback, a type converter, or a
   `System.Object` returned into a dangerous API, performs a side effect the attacker steers. Match the
   present assemblies against known chains rather than assuming any type resolution is exploitable.

5. **Check the defenses that actually stop it.** Replacing the formatter with one that deserializes to a
   fixed contract, disabling type-name handling, enforcing a strict allowlist binder, and setting a strong
   machine key for ViewState each remove the vector. A post-deserialization type check does not, because
   construction already ran. Determine which of these stands at the sink.

6. **Confirm and record.** Confirm by supplying a benign in-scope payload that constructs a marker type or
   triggers an out-of-band callback, proving the formatter honors the attacker's type, without a
   destructive gadget. Kill the lead if no untrusted data reaches a type-resolving formatter, if type
   resolution is disabled or bound to an allowlist, or if ViewState is integrity-protected and no
   reachable gadget exists. Record the source, the formatter, the type-resolution setting, and the chain.

## Where .NET deserialization leaks

- **The payload names the type.** With binary, SOAP, or type-name-handling serializers, the attacker
  chooses the class; the declared target type is irrelevant.
- **ViewState is a serialized object in the page.** A weak or missing machine key lets an attacker forge a
  `LosFormatter`/`ObjectStateFormatter` payload that deserializes on postback.
- **Type-name handling is a footgun setting.** Enabling it on a JSON serializer to preserve polymorphism
  reintroduces arbitrary type construction unless a strict binder constrains it.
- **A denylist binder ages badly.** Blocking known gadget types is bypassed by the next one; only an
  allowlist of expected types holds.
- **The gadget is in a referenced assembly.** The exploitable type is usually framework or library code,
  not application code, so auditing only your own types misses it.

## Worked example (a confirm and a kill)

> **Confirm.** A legacy endpoint accepts a serialized object in a hidden field and deserializes it with
> `BinaryFormatter`. The machine key is derivable, so an attacker forges a valid payload. A crafted object
> names a gadget type whose set accessor starts a process, and a benign marker payload confirms the type
> is constructed. **Confirmed** untrusted deserialization to remote code execution, `critical`,
> remediation = remove `BinaryFormatter`, deserialize to a fixed data contract with a data-only serializer,
> and rotate to a strong machine key so ViewState cannot be forged.
>
> **Kill.** The service deserializes only with a JSON serializer whose type-name handling is off and whose
> binder rejects any type outside a three-type allowlist, and ViewState uses a strong machine key with
> integrity protection. A crafted payload naming any gadget type is refused before construction.
> **Killed**, `kill_reason` = "no type-resolving formatter on untrusted data; JSON binder allowlists three
> data types and ViewState is integrity-protected, so no attacker type is constructed."

## Rationalizations to reject

- *"We deserialize to a specific class."* → Binary, SOAP, and type-name-handling formatters read the type
  from the data regardless of the declared target. Check the formatter, not the variable type.
- *"Type-name handling is convenient for polymorphism."* → It reintroduces arbitrary construction; if you
  must keep it, constrain it with a strict allowlist binder.
- *"ViewState is signed."* → Only if the machine key is strong and secret. A weak or shared key makes the
  signature forgeable, so the payload is attacker-controlled.
- *"We block the dangerous types."* → A denylist is bypassed by the next gadget; only an allowlist of
  expected types is durable.
- *"We check the object's type afterward."* → Construction and its callbacks already ran before your check.

## Executing this in practice

You need every formatter on untrusted input, its type-resolution setting or binder, the untrusted sources
that reach it, the ViewState machine-key posture, and the reachable gadget types on the loaded assemblies.
For each sink, ask whether attacker bytes arrive, whether the formatter honors an attacker type, and
whether a reachable gadget exists. Reading the formatter configuration shows the intent; a benign marker
payload shows whether the attacker's type is actually constructed.

## Related

- `hunting-java-deserialization-gadget-chains` - the JVM sibling, where the stream rather than a type name
  drives reconstruction of a gadget.
- `hunting-php-object-injection-pop-chains` - the PHP sibling, triggered by magic methods on reconstruction.
- `adjudicating-dependency-cve-reachability` - use it to prove a gadget-bearing assembly is loaded at a
  vulnerable version, not merely referenced.
- `adjudicating-taint-paths` - use it to confirm untrusted bytes reach the formatter through ViewState and
  framework indirection.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the untrusted serialized payload, sink = the
  type-resolving formatter, evidence = the attacker-chosen type driving a gadget to a dangerous call.
