---
name: hunting-xpath-and-xml-query-injection
description: >-
  Hunt XPath and XQuery injection where untrusted input is concatenated into a query expression that is
  then evaluated against an XML document or an XML database, so the input changes the structure of the
  query rather than supplying a value. A predicate closed early and rewritten to always be true bypasses
  an authentication or authorization lookup, and a rewritten path or an injected union walks the document
  to read nodes the caller was never meant to reach, including blind boolean and out-of-band variants
  where the response only reflects true or false. Use when an XPath or XQuery string is built from
  request data. The untrusted input concatenated into the expression is the source, the evaluation of
  that expression is the sink, and the attacker-controlled query structure is the bug.
license: MIT
---

# Hunting XPath and XQuery injection: when input rewrites the query

XPath and XQuery are to XML what SQL is to a relational database, and they carry the same injection flaw.
When an application builds a query by pasting untrusted input into the expression string, an attacker who
supplies query syntax instead of a plain value changes what the query means. A login check that looks up
a user node by matching a name and password predicate can be turned into a predicate that is always true,
logging in as the first user with no credential. A lookup scoped to one node can be widened to walk the
entire document and return records the caller should never see. Because XML documents have no table
grants, the whole document is usually reachable once the expression is under attacker control. You find
these by locating every expression built from request data and asking whether the input is bound as a
parameter or concatenated as syntax.

## When to use

- Code builds an XPath or XQuery expression by concatenating or interpolating request-derived strings.
- Authentication, authorization, or lookup logic evaluates such an expression against an XML document.
- An XML database or an XML-backed configuration store is queried with input-derived expressions.

## Scope check

Test XPath and XQuery injection only against applications you own or are authorized to assess, with test
accounts and test data, because a confirming query can read other users' records out of the document.
Prefer boolean and structural probes over bulk extraction, and coordinate before dumping data. If you
can't name the authorization, stop.

## The loop

1. **Establish that the expression is concatenated, not parameterized, first.** Locate every XPath or
   XQuery evaluation and read how its expression string is assembled. This is the false-positive killer:
   if the expression is a fixed literal and untrusted input is bound through a variable resolver or a
   precompiled expression with typed parameters, the input can only be a value and the query structure is
   fixed, so there is no injection. Name the evaluations where input is pasted into the string.

2. **Locate the injection point in the expression.** Determine where in the expression the untrusted value
   lands: inside a predicate string comparison, as part of a path step, inside a function argument. The
   position decides what breaking out of it costs and what syntax the attacker must supply to stay
   well-formed.

3. **Test for structural break-out.** Supply input that closes the current string or predicate and adds
   syntax: a quote that ends a comparison followed by an always-true predicate, an `or` that widens a
   match, a union or an additional path step that reaches sibling nodes. Confirm the evaluation accepts
   the rewritten expression rather than treating the whole thing as a literal value.

4. **Map the reachable document.** Once structure is controllable, determine how much of the document the
   expression can walk: whether an always-true predicate returns the first node, whether a union or an
   absolute path reaches nodes outside the intended scope, and whether node names and counts can be
   inferred. This is what turns a bypass into data extraction.

5. **Handle the blind and out-of-band cases.** When the response only reflects success or failure, confirm
   a boolean channel: an injected predicate that is true for a guessed character and false otherwise,
   inferred one node and one character at a time. Where the evaluator supports document or URL functions,
   check for an out-of-band channel that sends inferred data to a host you control.

6. **Confirm and record.** Confirm by turning an authentication or lookup predicate always-true, or by
   extracting a benign marker node the account should not reach, on a test document. Kill the lead if the
   expression is a fixed literal with typed variable binding, if the input is strictly validated or
   allowlisted to a value shape that cannot carry syntax, or if the evaluation is over a document with no
   sensitive nodes and no auth decision. Record the evaluation site, the injection position, the break-out
   used, and what it reached. Set `kill_reason` when killing.

## Where XPath and XQuery injection leaks

- **XML has no row-level grants.** Unlike a database with per-table privileges, an XML document is usually
  readable in full once the expression is attacker-controlled, so scope collapses immediately.
- **Login predicates are the classic target.** A name-and-password predicate concatenated from input is
  rewritten to an always-true condition that returns the first user node, an authentication bypass.
- **Blind is still injection.** A response that only says yes or no is a boolean oracle; the absence of an
  error message is not the absence of the bug.
- **Escaping quotes is not parameterization.** Manually escaping a quote misses numeric contexts, function
  arguments, and alternate quoting, whereas a bound typed variable removes the syntax path entirely.
- **The parser is often lenient.** XPath and XQuery engines accept a wide range of rewritten expressions,
  so a broken-out predicate frequently evaluates rather than erroring.

## Worked example (a confirm and a kill)

> **Confirm.** A login handler authenticates by evaluating an XPath expression that matches a user node on
> a name-and-password predicate built by string concatenation. A username value that closes the predicate
> and appends an always-true `or` condition makes the expression return the first user node regardless of
> the password. On a test document the handler signs in as that user with no valid credential. **Confirmed**
> XPath injection to authentication bypass, `high`, remediation = evaluate a fixed precompiled expression
> and bind the name and password as typed variables so input can never alter the predicate structure.
>
> **Kill.** A search feature evaluates a precompiled XQuery expression whose only input is bound through a
> typed external variable, and the input is additionally validated to an alphanumeric token. A value
> carrying quotes and predicate syntax is passed through as a literal string and matches nothing. **Killed**,
> `kill_reason` = "expression is fixed and precompiled with the input bound as a typed variable, so query
> structure is not attacker-controllable."

## Rationalizations to reject

- *"It is only XML, not a database."* -> XML documents commonly hold credentials, roles, and records, and
  an injectable expression can read all of it because there are no per-node grants.
- *"We escape single quotes."* -> Escaping one quote style misses numeric and function-argument contexts and
  alternate delimiters; only typed variable binding removes the structural path.
- *"There is no error, so it is not injectable."* -> A silent yes or no response is a boolean oracle that
  extracts data character by character; blind is still injection.
- *"The input is a small field."* -> Break-out syntax is short; an always-true predicate or a union step
  fits in a username field.
- *"We validate on the client."* -> The evaluation is server-side and the client check is bypassed; only a
  server-side bound expression or allowlist counts.

## Executing this in practice

You need every XPath and XQuery evaluation, how each expression string is assembled, and the origin of any
interpolated value. For each, decide whether the input is bound as a typed variable or concatenated as
syntax, where in the expression it lands, and what the document holds that a rewritten query could reach.
Reading the assembly shows whether structure is fixed; a break-out probe against a test document shows
whether the expression is rewritten, and a boolean probe confirms a blind channel.

## Related

- `hunting-orm-and-query-builder-injection` - the relational sibling; the same concatenation-versus-binding
  question decides both, here over XML instead of SQL.
- `hunting-blind-and-second-order-sql-injection` - the blind boolean and out-of-band extraction technique
  transfers directly to an XPath oracle.
- `hunting-xxe-and-xml-parser-trust` - the same XML input often reaches both a parser and a query; check the
  entity-expansion surface alongside the expression.
- `adjudicating-taint-paths` - use it to connect the request field to the exact evaluation site through the
  string assembly.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the untrusted input concatenated into the
  expression, sink = the XPath or XQuery evaluation, evidence = an always-true predicate or an extracted
  out-of-scope node on a test document.
