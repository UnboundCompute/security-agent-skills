---
name: hunting-server-side-prototype-pollution
description: >-
  Hunt server-side prototype pollution in JavaScript and TypeScript backends where untrusted input sets a
  __proto__, constructor, or prototype key through a recursive merge, a deep clone, a path-based set, or a
  query or body parser, polluting Object.prototype so a later property read returns an attacker value.
  Covers the pollution primitive (the write that reaches the prototype) and the gadget (a downstream read
  of an unset property that changes control flow, a command, a template, or a query). Use when a Node
  service merges or path-assigns untrusted structured input into objects and later reads properties that
  may be absent. The untrusted key reaching the prototype is the source, the polluting merge or set is the
  sink, and the gadget read that turns pollution into impact is the bug.
license: MIT
---

# Hunting server-side prototype pollution: when one request changes every object

In JavaScript, every plain object inherits from `Object.prototype`, so writing to that shared prototype
changes the default value of a property on every object in the process. Server-side prototype pollution
happens when untrusted input carries a `__proto__`, `constructor`, or `prototype` key into a recursive
merge, a deep clone, or a path-based set that follows the key down to the prototype. The write by itself
is a primitive; the impact comes later, when some other code reads a property that was never set and now
returns the attacker's injected default. That gadget read can flip an authorization flag, inject an
argument into a spawned command, or change a template or query. You find it by locating the pollution
primitives and then the gadget reads they enable.

## When to use

- A Node.js or server-side TypeScript service merges, clones, or path-assigns untrusted structured input.
- A parser or utility recurses into keys from a request body, query string, or JSON document.
- Later code reads object properties that may be unset, where an injected default would change behavior.

## Scope check

Test prototype pollution only against services you own or are authorized to assess, on non-production
infrastructure. Pollution is process-wide and can affect concurrent requests, so treat every confirming
write as a change to shared state within the authorized scope, and prefer an isolated instance. If you
can't name the authorization, stop.

## The loop

1. **Establish the pollution primitives first.** Inventory every operation that could write an
   attacker-controlled key down to the prototype: recursive or deep merges, deep clones, path-based setters
   that split a dotted or bracketed path, and body or query parsers that build nested objects. This is the
   false-positive killer: without a primitive that reaches `__proto__` or `constructor.prototype`, an
   isolated gadget read is not exploitable. Name the primitives first.

2. **Trace untrusted keys into each primitive.** Follow request bodies, query strings, and parsed JSON into
   the merge or set, focusing on whether the key path, not just the value, is attacker-controlled. A merge
   of an untrusted object, or a `set(obj, path, value)` where the path comes from the request, is the
   vector. Confirm the key can be `__proto__`, `constructor`, or `prototype`.

3. **Confirm the primitive reaches the shared prototype.** Read the merge or set implementation: does it
   guard against these keys, skip non-own properties, or use a null-prototype object or a Map? A guarded
   implementation writes an own property named `__proto__` harmlessly; an unguarded one walks into the
   prototype. Confirm the write actually lands on `Object.prototype`, not on an own key.

4. **Find the gadget read.** Pollution is only impactful if later code reads an unset property whose
   injected default changes behavior: an options object read for a shell argument, a flag read for an
   authorization decision, a template or view path, a property that selects a query or a callback. Locate
   the downstream reads of possibly-absent properties and identify which the pollution can steer.

5. **Check the defenses that actually stop it.** A merge that rejects `__proto__`/`constructor`/`prototype`
   keys, objects created with a null prototype, `Object.freeze(Object.prototype)`, `Map` instead of plain
   objects for untrusted keys, and schema validation that rejects unexpected keys each break the chain.
   Determine which stands between the primitive and the prototype, and whether the gadget read has its own
   default that resists an injected value.

6. **Confirm and record.** Confirm by polluting a benign marker property through the primitive on an
   isolated instance and observing the gadget read return the injected value, then, where a real gadget
   exists, showing the behavior change in scope. Kill the lead if no primitive reaches the prototype, if
   the merge or set guards the dangerous keys or uses null-prototype objects, or if no gadget read of an
   unset property changes behavior. Record the source key, the primitive, the polluted property, and the
   gadget.

## Where server-side prototype pollution leaks

- **The write and the impact are far apart.** The polluting merge and the gadget read are usually in
  different modules; auditing either alone misses the chain.
- **Path-based setters split attacker paths.** A `set(obj, "a.b.c", v)` with a request-derived path can be
  driven to `__proto__.polluted`, reaching the prototype without an explicit `__proto__` in the value.
- **Options objects are prime gadgets.** A spawn, a template render, or a query that reads options with
  defaults will pick up a polluted default for any option the caller omitted.
- **A guard on `__proto__` alone is incomplete.** `constructor.prototype` reaches the same shared prototype;
  a guard must cover all three key names.
- **Pollution crosses requests.** Because the prototype is process-global, a single request's pollution can
  affect unrelated concurrent requests until the process restarts.

## Worked example (a confirm and a kill)

> **Confirm.** A settings endpoint deep-merges the request body into a config object with a recursive merge
> that copies any key. A body with a `__proto__` key sets a default property that a later module reads as a
> shell argument when spawning a helper process. On an isolated instance, the injected default reaches the
> spawn, proving the gadget. **Confirmed** prototype pollution to command argument injection, `high`,
> remediation = reject `__proto__`, `constructor`, and `prototype` keys in the merge, build the config on a
> null-prototype object, and validate the body against a schema that allowlists known keys.
>
> **Kill.** The merge routine skips the three dangerous keys and operates on `Object.create(null)` targets,
> request bodies are validated against a strict schema that rejects unknown keys, and the spawn reads
> options with explicit own-property checks. A body carrying `__proto__` writes nothing to the shared
> prototype. **Killed**, `kill_reason` = "merge guards the prototype keys and uses null-prototype targets,
> schema rejects unexpected keys, and the gadget read uses own-property checks; no attacker key pollutes the
> prototype."

## Rationalizations to reject

- *"We do not use `__proto__` anywhere."* → The attacker supplies it in the request; the question is whether
  your merge or setter writes it to the prototype, not whether your code names it.
- *"We only merge trusted config."* → If any merge or path-set takes request-derived keys, it is a primitive
  regardless of the object's usual origin.
- *"We block `__proto__`."* → `constructor.prototype` reaches the same shared prototype; guard all three
  names or use a null-prototype target.
- *"There is no obvious sink."* → The gadget is a downstream read of an unset property, often in a different
  module; look for options objects, flags, and template paths read with defaults.
- *"It only pollutes our own request."* → The prototype is process-global; pollution persists and can affect
  concurrent and later requests.

## Executing this in practice

You need every merge, clone, and path-set that takes untrusted keys, whether each guards the prototype keys
or uses null-prototype targets, the downstream reads of possibly-unset properties, and the schema validation
if any. For each primitive, ask whether an attacker key reaches the prototype, and which gadget read it
steers. Reading the merge implementation shows whether the write lands on the prototype; polluting a marker
on an isolated instance shows the gadget read pick it up.

## Related

- `hunting-mass-assignment-and-property-authz` - the sibling where attacker-controlled keys reach fields
  they should not; here the same untrusted-key idea reaches the shared prototype.
- `hunting-php-object-injection-pop-chains` - another reconstruction-of-attacker-shaped-object class, where
  a chain rather than a prototype read carries the impact.
- `testing-client-side-dom-vulnerabilities` - client-side prototype pollution is the browser cousin; the
  primitive is the same, the gadgets differ.
- `adjudicating-taint-paths` - use it to connect a pollution primitive to a distant gadget read across
  modules.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the untrusted key reaching the prototype, sink =
  the polluting merge or set, evidence = the gadget read returning an injected default.
