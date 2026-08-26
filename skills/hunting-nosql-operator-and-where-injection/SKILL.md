---
name: hunting-nosql-operator-and-where-injection
description: >-
  Hunt NoSQL injection where untrusted input becomes query structure rather than a bound value: a request
  body whose keys turn into query operators, a value that arrives as an object instead of a scalar, or
  input reaching a server-side JavaScript evaluation such as $where, a mapReduce function, or an aggregation
  expression. Covers document stores where a filter built from a request object lets the caller inject
  comparison operators, always-true conditions, or code, and key-value or wide-column stores where input
  shapes the query language. Use when data access takes structured input from the request into a query
  filter or a server-side expression. The untrusted value that becomes an operator or an expression is the
  source, the query or evaluation call is the sink, and the missing type and shape check is the bug.
license: MIT
---

# Hunting NoSQL operator and where injection: when a value arrives as an operator

NoSQL injection rarely looks like classic SQL injection because there is no string to concatenate.
Instead, a document store filter is often built directly from a request object, so if the attacker sends
an object where the code expected a scalar, its keys become query operators. A login check that compares a
password to a request field can be turned always-true by sending an operator that matches anything. Worse,
some stores evaluate server-side JavaScript, in a `$where` clause, a mapReduce function, or an aggregation
expression, and untrusted input reaching that path is code execution inside the database. You find these by
separating filters that bind scalar values from those that accept a request-shaped object or an expression,
and tracing untrusted input into the latter.

## When to use

- Data access uses a document, key-value, or wide-column store with request-built query filters.
- A request field expected to be a scalar could arrive as an object whose keys become operators.
- Untrusted input can reach a server-side JavaScript evaluation: `$where`, mapReduce, or an aggregation.

## Scope check

Test NoSQL injection only against databases and applications you own or are authorized to assess, on
non-production data. A confirming operator or expression can read or alter records outside the intended
scope, so stay inside the authorized boundary. If you can't name the authorization, stop.

## The loop

1. **Establish the query sinks and split them.** Inventory where the store is queried, then separate calls
   that bind scalar values from calls that accept a request-shaped filter object or evaluate a server-side
   expression. This is the false-positive killer: a query with typed scalar parameters cannot be operator-
   injected. Name the structure-accepting and expression-evaluating sinks first.

2. **Trace untrusted input into filter objects.** Follow request bodies and query strings into filters
   built from them. The key question is whether a field expected to be a scalar can arrive as an object: if
   the code does `find({ user: req.body.user })` and `req.body.user` can be an object, the attacker sends
   an operator that changes the match. Confirm whether the value is type-checked to a scalar before use.

3. **Trace untrusted input into server-side expressions.** `$where`, mapReduce functions, and some
   aggregation expressions run code in the database engine. If untrusted input is concatenated into or
   supplied as one of these, it is code injection with the database's privileges. These paths are rarer but
   far more severe than operator injection.

4. **Confirm the impact class.** Operator injection typically yields authentication bypass, filter
   bypass, or blind data extraction by making a restricted query match more than intended. Expression
   injection yields code execution or heavy resource use. Determine which the reachable sink permits and how
   far it reaches.

5. **Check the defenses that actually stop it.** Casting each request field to its expected scalar type
   before it enters a filter, rejecting objects where scalars are expected, disabling server-side JavaScript
   evaluation, and validating the request body against a schema each remove the vector. A generic sanitizer
   that only strips characters does not, because the attack is structural, not textual. Determine which
   stands at the sink.

6. **Confirm and record.** Confirm by supplying an in-scope input that changes the query's meaning: a
   scalar field sent as an operator object that makes a restricted query match, or an expression that
   returns a distinguishable result, without exfiltrating out-of-scope data. Kill the lead if every filter
   binds type-checked scalars, if objects are rejected where scalars are expected, if no untrusted input
   reaches a server-side expression, and if server-side JavaScript is disabled. Record the input, the sink,
   and the change in the query's meaning.

## Where NoSQL injection leaks

- **A scalar field that accepts an object is the core bug.** If the request field can be an object, its keys
  become operators; the fix is a type check, not escaping.
- **Login and lookup checks are the classic operator-injection targets.** An equality comparison rewritten
  into a match-anything operator bypasses authentication or a filter.
- **Server-side JavaScript is code execution.** `$where`, mapReduce, and some aggregation expressions run
  code in the engine; untrusted input there is the most severe path.
- **A character sanitizer misses structural injection.** NoSQL injection changes the query's shape via types
  and keys, not via dangerous characters, so escaping does nothing.
- **Body parsers preserve nested objects.** Frameworks that parse bracketed query keys into nested objects
  hand the attacker operator injection through the URL, not just the body.

## Worked example (a confirm and a kill)

> **Confirm.** A login endpoint queries the user store with `{ email: body.email, password: body.password }`
> and never checks that the fields are strings. Sending `password` as an operator object that matches any
> value makes the query return the user without knowing the password. The response confirms authentication
> bypass. **Confirmed** NoSQL operator injection to authentication bypass, `critical`, remediation = cast
> `email` and `password` to strings and reject non-string values before building the filter, and validate the
> body against a schema that requires scalar types.
>
> **Kill.** Every request field entering a filter is cast to its expected scalar type and non-scalar values
> are rejected, the body is validated against a strict schema, and server-side JavaScript evaluation is
> disabled at the database. A field sent as an operator object is refused before the query. **Killed**,
> `kill_reason` = "filter fields are type-checked scalars, non-scalar input is rejected, and server-side
> JS is disabled; no untrusted value becomes an operator or expression."

## Rationalizations to reject

- *"NoSQL is not injectable like SQL."* → It is injectable structurally: an object where a scalar was
  expected turns values into operators, and `$where` runs code.
- *"We escape the input."* → Escaping targets characters; NoSQL injection is about types and keys, so a
  character filter does nothing. Type-check instead.
- *"The field is always a string in the client."* → The attacker is not using your client. Enforce the scalar
  type on the server before the value reaches the filter.
- *"We do not use `$where`."* → Confirm it, and check mapReduce and aggregation expressions too; any
  server-side evaluation on untrusted input is the severe path.
- *"It only affects one query."* → An operator that bypasses a login or a tenant filter is a full
  authorization break, not a narrow bug.

## Executing this in practice

You need every query sink split into scalar-binding and structure-accepting groups, the server-side
expression evaluations, the untrusted inputs that reach each, and the type-check or schema validation at
each field. For each structure-accepting sink, ask whether a scalar field can arrive as an object, and for
each expression sink whether untrusted input reaches it. Reading the query construction shows the intent;
sending an operator object or a distinguishing expression shows whether the boundary holds.

## Related

- `hunting-orm-and-query-builder-injection` - the same value-versus-structure distinction applied to
  relational mappers; operator injection there rhymes with filter injection here.
- `hunting-blind-and-second-order-sql-injection` - a sibling extraction technique when the response reveals
  only a boolean or a delay.
- `hunting-search-engine-injection` - another structured-query store where request objects become query
  clauses.
- `adjudicating-taint-paths` - use it to confirm an untrusted field reaches a filter or a `$where` path as
  structure rather than a bound scalar.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the untrusted value that becomes an operator or
  expression, sink = the query or evaluation call, evidence = the change in the query's meaning.
