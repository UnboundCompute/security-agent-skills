---
name: hunting-mass-assignment-and-property-authz
description: >-
  Hunt mass assignment and broken object-property authorization: handlers that bind a
  client request payload straight onto a record or model and let the caller write fields
  it should never control - role, is_admin, owner_id, tenant, price, balance, verified,
  status, or another user's foreign key. Covers auto-binding and hydration that take the
  whole payload, blocklist filters that miss a field, nested and relation fields that
  reopen the hole, type juggling that flips a flag, and read paths that return properties
  the caller should not see. Use when reviewing any create or update handler that maps
  request fields onto a persisted object. The payload field is the source, the record
  write is the sink, and the server-controlled property is the bug.
license: MIT
---

# Hunting mass assignment: which fields, not which object

Broken object-level authorization asks whether the caller may reach *this object*. Mass
assignment asks the next question: given an object the caller may write, which of its
*fields* may they set? The bug is a handler that binds a request payload onto a record
and trusts the client to send only the fields it should, so an attacker adds
`"role":"admin"`, `"owner_id": <someone else>`, `"price": 0`, or `"verified": true` to a
payload the endpoint was never meant to accept, and the framework writes it. You find it
by listing what the client can put into the payload, following it to the record write,
and asking which of those fields the server, not the client, is supposed to own.

## When to use

- You are reviewing a create or update handler that maps request fields onto a
  persisted object.
- The framework auto-binds or hydrates a payload onto a model, struct, or record.
- Some properties of the object are meant to be server-controlled: privilege, ownership,
  tenancy, money, state, or verification flags.

## Scope check

Test property writes only in systems you own or are authorized to test, with accounts
that let you observe a privilege, ownership, or state change take effect. If you can't
name the authorization, stop.

## The loop

1. **List the writable fields the client can reach.** For each create and update
   handler, determine how the payload becomes a record: an explicit field-by-field
   assignment, an allowlisted binder, or a whole-payload hydration. Whole-payload
   binding means every column is a candidate; a binder means whatever the allowlist
   admits. Enumerate the fields the client can actually set, not the ones the form shows.

2. **Follow the payload to the record write.** Trace the bound fields from the request to
   the persistence call, the object mutation and save. That write is the sink. Note
   whether the binding is an allowlist (only named fields) or a blocklist (everything
   except named fields); a blocklist is a standing bet that no sensitive field was ever
   added later.

3. **Name the server-controlled fields.** Which properties of this object must the server
   own, not the client: `role`, `is_admin`, permissions, `owner_id`, `tenant`, foreign
   keys to other users, `price`, `balance`, credit, `status`, `verified`, `created_by`,
   timestamps? Any of these reachable through the binding is the bug. The audit is the
   overlap between client-writable and server-owned.

4. **Check the ways the hole reopens.** A clean top-level allowlist can still leak
   through a nested object or relation that binds recursively, a field whose name differs
   from the column but maps through an alias, a second update path with looser binding, or
   type juggling where a string, array, or object where a scalar was expected flips a
   check or a flag. Enumerate nested, aliased, and sibling-path bindings, not just the
   flat top level.

5. **Test the injected field.** Confirm by adding the server-owned field to the payload
   and observing it take effect: set `role`/`is_admin` and gain privilege, set `owner_id`
   or a tenant key and reassign or reach another party's object, set `price`/`balance` and
   move money, set `verified`/`status` and skip a gate. On the read side, check whether the
   response serializes properties the caller should not see. The state change or leaked
   property is the proof.

6. **Confirm and record.** Confirm with the request that sets a server-owned field and the
   observed effect. Kill the lead if every write path binds an explicit allowlist that
   excludes all server-owned fields, nested and relation binds are constrained the same
   way, and reads serialize only permitted properties. Record with the handler, the
   injected field, and the privilege, ownership, money, or state change it caused.

## Where property authorization leaks

- **Blocklist binding is a bet against the future.** It protects the fields someone
  remembered to exclude; the sensitive column added six months later walks straight
  through. Allowlist per handler.
- **The form is not the API.** Client-writable is every field the binder accepts, not the
  ones the UI renders. Test fields that never appear on screen.
- **Nested and relation fields reopen a clean top level.** Recursive binding on an embedded
  object or an association is where the allowlisted handler still leaks.
- **The same object has more than one write path.** A strict create and a loose update, or
  an admin path with whole-payload binding, and the loose one governs.
- **Reads leak properties too.** Serializing the whole record hands the caller fields
  (internal flags, other users' keys, secrets) that the write side carefully guarded.

## Worked example (a confirm and a kill)

> **Confirm.** `PATCH /api/profile` binds the whole JSON body onto the user record with a
> blocklist that excludes `password_hash` but not `role`. Sending `{"role":"admin"}`
> alongside a normal profile edit persists the new role, and the next request is
> privileged. **Confirmed** mass assignment to privilege, `critical`, remediation = bind an
> explicit allowlist of user-editable profile fields and reject or ignore everything else,
> with a test that `role` cannot be set through this path.
>
> **Kill.** Every create and update handler assigns a named allowlist of client-editable
> fields; `role`, `owner_id`, tenant, money, and state columns are set only by server code;
> nested objects bind through their own allowlists; responses serialize an explicit view,
> not the record. Injecting each server-owned field has no effect and none appear in reads.
> **Killed**, `kill_reason` = "per-handler allowlist binding, server-owned fields never
> client-writable at any path including nested and update, explicit read view."

## Rationalizations to reject

- *"The UI never sends that field."* → The API accepts it. Property authorization is about
  what the binder writes, not what the form shows.
- *"We strip the dangerous fields."* → A blocklist is only as complete as its last update.
  Name what may be written, not what may not.
- *"Only the create path is exposed."* → And the update path, the admin path, the nested
  bind? The loosest write path governs the object.
- *"The field is validated."* → Validated for type, or for who may set it? A well-typed
  `role` still should not be client-writable.
- *"It is just a profile edit."* → A profile edit that reaches `role` or `owner_id` is a
  privilege or ownership change wearing a profile edit's clothes.

## Executing this in practice

You need, per write handler, the set of fields the binding actually admits (not the form
fields), a trace from payload to the record write, the list of properties that must be
server-owned for this object, and accounts that let you watch an injected field take
effect. A dataflow view that carries request fields to the columns they write, and flags
whole-payload or blocklist binding versus an explicit allowlist, turns this into a coverage
pass over every write path; the list-fields, follow-to-write, name-the-server-owned method
is the same by hand.

## Related

- `hunting-broken-object-level-authorization` - the sibling bug: which *object* the caller
  may reach, where this skill is which *fields* of it they may write.
- `auditing-declarative-authorization` - authorization expressed as config; property writes
  are the field-granular layer beneath it.
- `hunting-business-logic-flaws` - where a writable field (quantity, price, limit) drives an
  outcome the rules did not intend.
- `finding-fail-open-flaws` - a binder that admits everything by default is the fail-open
  shape at the field layer.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the injected payload field, sink =
  the record write, evidence = the privilege, ownership, money, or state change observed.
