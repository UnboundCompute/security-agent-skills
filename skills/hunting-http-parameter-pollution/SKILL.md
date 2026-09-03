---
name: hunting-http-parameter-pollution
description: >-
  Hunt HTTP parameter pollution, where the same parameter name appears more than once, or a parameter is
  shaped as an array, and two components in the request path disagree on how to resolve it. A validator, a
  gateway, or a filter reads one occurrence while the backend sink reads another, so a value that passes a
  check is not the value that is used, and a control is bypassed, an authorization decision is confused, or
  a payload is smuggled past a filter. First-versus-last-wins differentials, array-versus-scalar coercion,
  and a gateway that concatenates duplicates while the app splits them are the usual mechanisms. The bug
  exists only when two parsers differ, so proving the differential is the whole task. Use when duplicate or
  array-shaped parameters cross more than one parser. The duplicated parameter is the source, the sink that
  resolves it differently from an earlier check is the sink, and the parser disagreement is the bug.
license: MIT
---

# Hunting HTTP parameter pollution: when two parsers disagree

An HTTP request can carry the same parameter name twice, or shape a parameter as a list, and nothing in
the protocol says which value wins. Each component decides on its own: one framework takes the first
occurrence, another the last, a third concatenates them, a fourth exposes an array. Parameter pollution is
the bug that appears when two components on the path make different choices about the same duplicated
input. A gateway or a validator inspects one occurrence and approves the request; the backend that acts on
the request reads a different occurrence. The value that was checked is not the value that is used. That
gap bypasses a security filter, confuses an authorization or routing decision, or slips a payload past a
web application firewall that examined the harmless copy. The vulnerability is entirely relational: a
single parser is never wrong with itself, so the work is to find two parsers that differ and a check on
one side of the difference.

## When to use

- A request crosses more than one component that parses parameters (gateway, framework, backend, service).
- A security check, filter, or authorization decision reads a parameter that is also read by a later sink.
- Parameters can be duplicated or array-shaped, in the query string, the body, or both at once.

## Scope check

Test parameter pollution only against systems you own or are authorized to assess, on non-production
infrastructure, because a confirming request can bypass an access control or reach a privileged action.
Probe with benign values and observe which occurrence is honored rather than exercising the bypass against
real data. If you can't name the authorization, stop.

## The loop

1. **Establish that two components parse duplicates differently first.** Map the components a request
   traverses and determine how each resolves a repeated or array-shaped parameter: first wins, last wins,
   concatenation, array exposure, error. This is the false-positive killer: if every component on the path
   resolves duplicates identically, the checked value and the used value are always the same and there is
   no pollution bug. Name at least two components whose resolution differs before proceeding.

2. **Locate a check and a sink that read the same parameter.** Find a parameter that is both inspected by a
   validator, gateway, filter, or authorization step and consumed by a downstream sink (a query, a
   redirect, an access decision, a spawned action). The bug needs the same name read on both sides of the
   parser difference.

3. **Determine which occurrence each side reads.** Establish, for the specific parameter, which occurrence
   the checking component honors and which the sink honors. When they diverge (the check reads the first
   and the sink reads the last, or the check sees a scalar and the sink sees an array), an attacker can put
   a benign value where the check looks and a malicious value where the sink looks.

4. **Test the array-versus-scalar coercion.** Many frameworks turn a repeated name or a bracketed name into
   an array or a nested structure, while a check expects a scalar. Probe whether an array-shaped value
   changes a type-sensitive comparison, defeats an equality or membership check, or alters how the sink
   interprets the value.

5. **Locate the filter or gateway divergence.** Where a web application firewall or an upstream gateway
   inspects one occurrence and passes the request, check whether a second occurrence carrying the payload
   reaches the backend unexamined. The pollution is then a delivery mechanism for another injection whose
   payload was smuggled past the inspector.

6. **Confirm and record.** Confirm by sending a request with duplicated or array-shaped parameters where
   one occurrence is benign and the other is the attack, and observing that the check approves while the
   sink acts on the attacker occurrence, on an isolated instance. Kill the lead if all components resolve
   duplicates identically, if the check and the sink read the same occurrence, or if the framework rejects
   duplicates outright. Record the parameter, the two components and their differing resolution, the check,
   and the sink. Set `kill_reason` when killing.

## Where parameter pollution leaks

- **The protocol picks no winner.** Nothing standardizes which duplicate is authoritative, so every hop is
  free to choose differently, and differences between hops are where the bug lives.
- **The check and the sink are the two parsers that matter.** A gateway that validates the first copy and a
  backend that uses the last copy is the canonical split; the value inspected is not the value used.
- **Array coercion breaks scalar checks.** A comparison written for a single string behaves unexpectedly
  when the framework hands it an array, flipping equality, membership, or emptiness tests.
- **Web application firewalls inspect one copy.** A payload placed in a second occurrence can pass an
  inspector that only examined the first, turning pollution into a smuggling channel for another injection.
- **Query and body can collide.** A parameter present in both the query string and the body may be merged
  or shadowed differently by different layers, another source of divergence.

## Worked example (a confirm and a kill)

> **Confirm.** A gateway authorizes a money-movement endpoint by reading the first occurrence of an account
> parameter and confirming the caller owns it, while the backend service reads the last occurrence when it
> executes the transfer. A request that lists the caller's own account first and a victim's account last
> passes the ownership check and moves funds from the victim account. On an isolated instance the transfer
> executes against the last occurrence. **Confirmed** parameter pollution to authorization bypass, `high`,
> remediation = resolve duplicates identically across the gateway and the backend, reject requests with
> repeated parameters on sensitive endpoints, and authorize on the exact value the sink will use.
>
> **Kill.** A search endpoint sits behind a framework that rejects any request containing a repeated
> parameter name with an error, and array-bracket names are not enabled, so the validator and the sink
> always receive the same single value. A duplicated parameter never reaches the handler. **Killed**,
> `kill_reason` = "the framework rejects duplicate parameters before handling, so the check and the sink
> always resolve to the same value and no parser differential exists."

## Rationalizations to reject

- *"Parameters are never duplicated."* -> A client can duplicate any parameter freely; the question is how
  each component resolves the duplicate, not whether your own client sends one.
- *"The framework handles it."* -> One framework handles it one way and the next hop another; a single
  component being self-consistent does not remove a difference between two components.
- *"We validate the input."* -> If the validator reads a different occurrence than the sink, validation
  approves a value the sink never uses; the check must read the value the sink will act on.
- *"The firewall blocks the payload."* -> If the firewall inspects only one occurrence, a payload in a second
  occurrence is delivered unexamined; inspection and use must agree.
- *"It is only a duplicate string."* -> Duplication also drives array coercion and query-versus-body merges
  that change types and shadow values, not just which string wins.

## Executing this in practice

You need the ordered list of components a request crosses, how each resolves repeated and array-shaped
parameters, and every parameter that is both checked and used. For each such parameter, decide whether the
checking component and the sink read the same occurrence, whether array coercion changes a comparison, and
whether a filter inspects only one copy. Reading each parser's resolution rule shows where two differ; a
request with a benign and a malicious occurrence shows which side honors which, confirming the split.

## Related

- `hunting-orm-and-query-builder-injection` - pollution frequently delivers an injection payload past a
  filter to a query sink; that skill covers the sink it lands in.
- `hunting-host-header-and-url-parsing-trust` - another class where two components parse the same request
  element differently, with the trust decision on the wrong copy.
- `testing-request-smuggling` - the request-boundary cousin, where front and back ends disagree about where
  a request ends rather than which parameter value wins.
- `adjudicating-taint-paths` - use it to connect the checked occurrence and the used occurrence through the
  components that resolve them.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the duplicated or array-shaped parameter, sink =
  the component that resolves it differently from an earlier check, evidence = a request whose check-side
  and sink-side occurrences diverge on an isolated instance.
