---
name: hunting-orm-and-query-builder-injection
description: >-
  Hunt injection that survives an object-relational mapper or query builder: untrusted input
  reaching a raw-query escape hatch, an unparameterizable identifier (a column, table, or sort
  order), or a structured filter or update object whose keys become query operators or column
  references. Covers raw-query methods that take a string or fragment, sort and column selectors
  taken from the request, and operator injection where a request body passed as a filter turns a
  comparison always-true or references a field it should not. Use when data access goes through an
  ORM or query builder and untrusted input reaches a raw method, an identifier argument, or a
  filter or update object rather than a bound value. The untrusted value that becomes query
  structure is the source, the data-access call is the sink, and the missing allowlist between them
  is the bug.
license: MIT
---

# Hunting ORM and query-builder injection: when the abstraction is not the safeguard

Teams reach for an object-relational mapper or a query builder partly to avoid injection, then assume
the abstraction handles it. It handles values: a bound parameter is safe no matter what it contains.
It does not handle structure. The moment untrusted input becomes a column name, a sort direction, a
raw fragment, or an operator inside a filter object, the placeholder machinery is bypassed and the
query means something the developer did not write. Classic string-concatenation injection is largely
gone from ORM codebases; the live risk moved to the escape hatches and the structured inputs. You find
it by separating the calls that bind values from the calls that accept structure and tracing untrusted
input into the latter.

## When to use

- Data access goes through an object-relational mapper or query builder, not hand-written statements.
- Untrusted input reaches a raw-query method, a column, table, or sort argument, or a filter object.
- Request bodies are passed as filter or update objects into the data layer.

## Scope check

Test injection only against databases and applications you own or are authorized to assess, on
non-production data. Confirming an operator-injection read can expose records outside your test scope.
If you can't name the authorization, stop.

## The loop

1. **Map the data-access sinks and separate them.** Inventory where the mapper or builder is used, then
   split the calls into two groups: parameterized calls that bind values, and escape hatches that accept
   structure. The escape hatches are raw-query methods, methods that take a column, table, or order
   argument, and methods that accept a structured filter or update object. Only the second group can be
   injected.

2. **Trace untrusted input into raw-query escape hatches.** A raw method takes a query string or a
   fragment. If untrusted input is concatenated or interpolated into that string, it is classic injection
   even though the file is full of ORM calls. Parameter placeholders inside a raw fragment protect the
   values only; anything spliced in as text is live.

3. **Trace untrusted input into identifiers.** Column names, table names, sort fields, and sort direction
   usually cannot be bound as parameters. If any of them comes from the request, a sort field or a column
   selector, it must be checked against a fixed allowlist. An attacker-chosen identifier can read columns
   the caller was never meant to see, and in some builders can smuggle a subexpression.

4. **Trace structured filter and update objects (operator injection).** When a request body is passed
   directly as a filter, its keys can become operators or references the caller never intended: a
   comparison rewritten into an always-true condition, a key that names a different column, or a nested
   operator that changes the query's logic. Determine whether the object is constrained to an expected
   shape and set of fields, or merged in whole.

5. **Find where value crosses into structure.** The safe line is simple: untrusted data may be a value
   bound as a parameter, but untrusted data that becomes an identifier, an operator, a fragment, or a raw
   string must be allowlisted. Locate every place that line is crossed and whether an allowlist stands
   there. A cast or a type check is not an allowlist.

6. **Confirm and record.** Confirm by supplying an input that changes the query's structure in scope: an
   identifier that returns a column outside the intended set, a filter operator that makes a restricted
   query return everything, or a raw fragment that appends a condition. Kill the lead if every raw method
   takes only bound parameters, all identifiers and sort inputs are allowlisted, filter and update objects
   are constrained to an expected shape and field set, and no untrusted value becomes query structure.
   Record with the input, the call, and the change in the query's meaning.

## Where ORM and query-builder injection leaks

- **The mapper protects values, not structure.** Bound parameters are safe; identifiers, operators,
  fragments, and raw strings are not. Injection lives wherever untrusted input becomes structure.
- **Sort and column selectors are the common identifier sink.** A `sort by` field taken from the request
  and dropped into the query is not parameterizable, so without an allowlist it is injectable.
- **A whole request body as a filter is operator injection waiting to happen.** Merging untrusted keys
  into a `where` object lets the caller add operators and references, not just supply values.
- **A raw method inside ORM code is easy to miss.** The surrounding parameterized calls create a false
  sense of safety around the one call that concatenates.
- **A type check is not an allowlist.** Confirming a value is a string does nothing to stop it being a
  malicious identifier; the field must be one of a known, fixed set.

## Worked example (a confirm and a kill)

> **Confirm.** A listing endpoint takes a `sort` parameter and passes it straight into the builder's
> order clause, which does not parameterize identifiers. A crafted `sort` value injects a subexpression
> that orders by a boolean over a sensitive column, letting an attacker read that column one bit per
> request across the result set. **Confirmed** identifier injection to data exfiltration, `high`,
> remediation = map the `sort` parameter through a fixed allowlist of permitted columns and directions
> and reject anything else, and never pass request text into an order or identifier position.
>
> **Kill.** Every raw method receives only bound parameters, the `sort` and column selectors are mapped
> through a fixed allowlist to known identifiers, and filter objects are validated to an expected field
> set and operator whitelist before reaching the builder. No crafted identifier, operator, or fragment
> changes the query's structure. **Killed**, `kill_reason` = "values bound, identifiers and sort
> allowlisted, filter objects constrained to a known shape; no untrusted input becomes query structure."

## Rationalizations to reject

- *"We use an ORM, so we are safe from injection."* → Safe for bound values. The raw methods, identifier
  arguments, and filter objects are the part the ORM does not protect.
- *"The sort field is just a string."* → A string that becomes an identifier, which cannot be
  parameterized. Without an allowlist, a string is exactly the injection vector.
- *"We pass the filter object straight through for flexibility."* → That flexibility is operator
  injection: the caller supplies logic, not just values. Constrain the shape.
- *"There is a placeholder in the raw query."* → Placeholders bind the values around them; anything
  concatenated into the fragment as text is still injected.
- *"It only reads data."* → An identifier or operator injection that reads arbitrary columns or rows is a
  disclosure bug of the same severity as a write.

## Executing this in practice

You need every data-access call split into value-binding and structure-accepting groups, the raw-query
methods and their arguments, the identifier and sort inputs and whether each is allowlisted, and the
filter and update objects and whether their shape is constrained. For each structure-accepting sink, ask
whether untrusted input can reach it and whether an allowlist stands between. Reading the call shows the
intent; supplying an input that changes the query's meaning shows whether the boundary holds.

## Related

- `hunting-mass-assignment-and-property-authz` - the write-side sibling: an untrusted structured object
  reaching fields it should not; here the same object shape reaches query logic.
- `adjudicating-taint-paths` - use it to confirm a request value actually reaches a raw method or
  identifier sink through the mapper's indirection.
- `auditing-graphql-attack-surface` - a query API is a common source of the filter and sort objects that
  land in these sinks; trace the two together.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the untrusted value that becomes query
  structure, sink = the data-access call, evidence = the change in the query's meaning.
