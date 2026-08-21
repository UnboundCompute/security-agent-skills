---
name: auditing-serverless-event-source-trust
description: >-
  Audit event-driven function handlers that trust the event because it arrived from inside the
  platform: a handler that treats any delivered event as authentic without verifying its true source
  or integrity, event fields flowing untrusted into a database write, command, downstream call, or
  constructed path, a function reachable by several event types or actors that runs privileged logic
  for one that should not trigger it, and an over-broad execution identity that turns a single spoofed
  or injected event into wide reach. Covers source authentication, event injection, per-event-type and
  per-actor authorization, and execution-role blast radius across queues, object-store notifications,
  schedules, topics, and gateways. Use when functions are triggered by events and the handler trusts
  the payload or its claimed origin. The event payload is the source, the handler's action is the sink,
  and the assumed-not-verified trust is the bug.
license: MIT
---

# Auditing serverless event-source trust: when the handler trusts the event because it came from inside

Event-driven functions invert the usual request model: instead of a caller reaching an endpoint, a
platform delivers an event to a handler. The recurring mistake is treating that delivery as
authentication, the handler assumes the event is genuine, well-formed, and authorized simply because it
came through the platform's plumbing. But the event source may accept writes from more parties than
intended, the payload is attacker-influenceable input, and the same handler may be reachable by triggers
it should never honor. The event payload is the source, the handler's action is the sink, and the bug is
trust that was assumed rather than verified. You find it by mapping each function to what can trigger it
and asking what the handler checks before it acts.

## When to use

- Functions are triggered by events from queues, object-store notifications, schedules, topics, or gateways.
- A handler trusts the event payload's contents or its claimed source without verifying either.
- Per-event-type authorization or source authentication is assumed rather than checked in the handler.

## Scope check

Audit functions, event sources, and execution identities you own or are authorized to assess, and inject
crafted events only into non-production triggers where a spurious run is safe. A forged event can drive a
real privileged action. If you can't name the authorization, stop.

## The loop

1. **Map each function to its triggers and its assumptions.** For every handler, identify what can invoke
   it, a message queue, an object-store notification, a schedule, a topic, an upstream gateway, and what the
   handler takes for granted: that the event came from a trusted producer, that its fields are well-formed,
   that the actor behind it is already authorized. Those assumptions are the surface.

2. **Check whether the event's true source is verified.** A handler often treats any event the platform
   delivers as authentic. But an event source can be writable by more parties than intended: an object store
   others can write to, a queue with broad send permission, a public-facing gateway. Determine whether the
   handler confirms the event's origin and integrity, or merely that it arrived.

3. **Trace event fields to dangerous sinks.** The event payload is untrusted input. Follow its fields into
   the handler's operations: a database write, a spawned command, a downstream service call, a path or key
   built from a field, a URL fetched. An event field reaching a sink with no validation is injection
   delivered through the event channel rather than an HTTP parameter.

4. **Check per-event and per-actor authorization.** A function reachable by several event types or actors may
   run privileged logic for a trigger that should not be allowed to cause it. Determine whether the handler
   authorizes the specific event type and the actor behind it, or executes the same sensitive path for
   anything that arrives on any of its triggers.

5. **Check the function's own privileges.** The handler runs under an execution identity; if that identity is
   broader than the function needs, a single spoofed or injected event reaches everything the identity can
   touch. Treat an over-broad execution role as the amplifier that turns a small trust gap into wide impact.

6. **Confirm and record.** Confirm by delivering a crafted or spoofed event in scope and showing the handler
   act on it: perform an unauthorized privileged action, carry an injected field into a sink, or run for a
   trigger it should reject. Kill the lead if the handler verifies the event's source and integrity, validates
   every field before use, authorizes the specific event type and actor, and runs under a least-privilege
   execution identity. Record with the event, the trigger, and the handler action it drove.

## Where event-source trust leaks

- **"From inside the platform" is not authentication.** Delivery proves the event arrived, not that it came
  from a trusted producer. The handler still has to verify origin and integrity.
- **An event field is untrusted input.** The payload is chosen upstream; treating its fields as safe because
  they came through the event channel is the same mistake as trusting a request body unchecked.
- **Shared event sources have many writers.** An object store or queue writable by broad permission lets an
  attacker plant the event; the trigger is not a trust boundary by itself.
- **One handler, many triggers, one missing check.** A function that runs privileged logic for any event it
  receives is exposed through its weakest trigger, not its intended one.
- **The execution identity sets the blast radius.** A least-privilege role contains a trust gap; an
  over-broad one lets a single forged event reach everything the function can.

## Worked example (a confirm and a kill)

> **Confirm.** A function is triggered by notifications from an object store and trusts the event's object
> key as a trusted, well-formed path, using it directly to read and then act on the object. The object store
> accepts writes from a broad set of principals, so an attacker writes an object with a crafted key that the
> handler follows into a path outside the intended prefix, and the function's broad execution identity lets
> the result reach sensitive data. **Confirmed** event-source trust and injection to data exposure, `high`,
> remediation = verify the event's source and validate the key against the expected prefix and shape before
> use, restrict who can write the source, and narrow the execution identity to least privilege.
>
> **Kill.** The handler verifies the event's origin and integrity, validates and constrains every field
> before using it, authorizes the specific event type and actor for the action, and runs under an execution
> identity scoped to only what it needs. Spoofed events, crafted fields, and unexpected trigger types are all
> rejected or contained. **Killed**, `kill_reason` = "event source and integrity verified, fields validated,
> event type and actor authorized, least-privilege execution identity; forged and injected events do not
> drive a privileged action."

## Rationalizations to reject

- *"The event came from our own queue, so it is trusted."* → Who can write to that queue or store? Delivery
  through your infrastructure is not proof of a trusted producer.
- *"It is an internal function, not an endpoint."* → Its triggers are its endpoints. If an attacker can place
  an event on any of them, the function is reachable.
- *"The payload is structured, so it is safe."* → Structure is not validation. A well-formed event with a
  hostile field reaches the sink just the same.
- *"The function only does one small thing."* → It does that one thing with its execution identity's full
  reach. The blast radius is the role, not the code size.
- *"Every trigger is legitimate."* → Then authorize per trigger and per actor. A handler that acts the same for
  any event it receives is exposed through the weakest one.

## Executing this in practice

You need each function's triggers and event sources, what the handler assumes and what it verifies about the
event's origin and fields, the trace from event fields into the handler's sinks, the per-event-type and
per-actor authorization, and the breadth of the execution identity. For each, ask what an attacker who can
place or influence an event achieves. Reading the handler shows the intended trust; delivering a crafted or
spoofed event shows what it actually accepts.

## Related

- `hunting-iam-privilege-escalation-paths` - the execution identity's reach is a role graph; use that skill to
  measure the blast radius a forged event inherits.
- `hunting-non-human-identity-and-secret-reachability` - a function's identity and the secrets it can reach are
  the machine-credential surface this trust gap amplifies.
- `auditing-declared-vs-used-permissions` - an over-broad execution role is an over-grant in this setting;
  narrowing it is the containment.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the event payload, sink = the handler's action,
  evidence = the crafted or spoofed event and the action it drove.
