---
name: hunting-search-engine-injection
description: >-
  Hunt injection into search and analytics engines such as Elasticsearch, OpenSearch, and Solr where
  untrusted input reaches a query DSL body, a query-string or Lucene query, a script field, or a stored
  scripting expression. Covers request data that becomes query structure (a filter clause, a field
  selector, an aggregation) rather than a bound term, a raw query-string parameter whose Lucene syntax the
  caller controls, and script injection through inline or stored scripts that run in the engine. Use when an
  application forwards user input into a search cluster as query JSON, a query string, or a script. The
  untrusted value that becomes query or script structure is the source, the search or script call is the
  sink, and the missing field allowlist or the enabled dynamic script is the bug.
license: MIT
---

# Hunting search-engine injection: when a search query is attacker-shaped

Search and analytics engines take rich structured queries, and applications frequently build those queries
from user input. The injection surface has three shapes. First, the query DSL body: when a request object
is merged into the query JSON, the attacker can add filter clauses, select fields the caller was not meant
to read, or attach aggregations that reveal data across the index. Second, the raw query string: a
`query_string` or Lucene query parameter exposes operators, field selectors, and wildcards, so a caller who
controls it can query fields and ranges outside the intended scope. Third, scripting: inline or stored
scripts run inside the engine, and untrusted input reaching a script is code execution in the cluster. You
find these by separating bound search terms from query structure and scripts, and tracing untrusted input
into the latter.

## When to use

- An application forwards user input into a search or analytics cluster as query JSON or a query string.
- A request object is merged into the query DSL, or a raw Lucene/query-string parameter is caller-controlled.
- Inline or stored scripts run in the engine and untrusted input can reach a script or its parameters.

## Scope check

Test search-engine injection only against clusters and applications you own or are authorized to assess, on
non-production data. A confirming query or script can read across indices or run code in the cluster, so
stay inside the authorized boundary. If you can't name the authorization, stop.

## The loop

1. **Establish the query and script sinks and split them.** Inventory every search call and separate the
   parts that bind a search term from the parts that carry structure: the query DSL body, field and index
   selectors, aggregations, raw query-string parameters, and inline or stored scripts. This is the
   false-positive killer: a query that binds the user's term into a fixed match clause over a fixed field is
   not injectable. Name the structure-carrying and script sinks first.

2. **Trace untrusted input into the query DSL body.** Follow request objects merged into the query JSON. If
   the caller can add clauses, choose the field to match, or attach an aggregation, they can read beyond the
   intended documents and fields. Confirm whether the query is assembled from a fixed template with the term
   bound, or built by merging an untrusted object.

3. **Trace untrusted input into raw query strings.** A `query_string`, `simple_query_string`, or Lucene
   query parameter exposes the full query syntax: field selectors, ranges, wildcards, and boolean operators.
   If the raw parameter is caller-controlled, the caller can query fields and documents outside scope even
   without touching the JSON structure. Confirm whether the raw syntax is exposed or the term is bound into a
   constrained query.

4. **Trace untrusted input into scripts.** Inline scripts, stored-script parameters, and script fields run in
   the engine. Untrusted input concatenated into a script body is code injection; untrusted input as a script
   parameter is safe only if the script body is fixed. Confirm whether dynamic scripting is enabled and
   whether any script body is built from input.

5. **Check the defenses that actually stop it.** Binding the user term into a fixed query template,
   allowlisting the fields and indices the caller may select, avoiding `query_string` in favor of a
   constrained match query, disabling inline dynamic scripts, and using stored scripts with bound parameters
   each remove a vector. A per-request field or index restriction enforced by the application, not just by a
   convention, is what holds. Determine which stands at the sink.

6. **Confirm and record.** Confirm by supplying an in-scope input that changes the query's reach, selecting a
   field or an index outside the intended scope, or, where scripting is exposed, a script that returns a
   distinguishing result, without exfiltrating out-of-scope data. Kill the lead if the user term is bound
   into a fixed template, if field and index selection is allowlisted, if raw query-string syntax is not
   exposed, and if dynamic scripting is disabled. Record the input, the sink, and the change in the query's
   reach.

## Where search-engine injection leaks

- **A merged request object becomes query structure.** Merging user JSON into the query DSL lets the caller
  add clauses and pick fields; bind the term into a fixed template instead.
- **`query_string` exposes the whole query language.** A raw Lucene parameter hands the caller field
  selectors and ranges, so a search box becomes a cross-field query tool.
- **Field and index selectors are authorization boundaries.** If the caller chooses the field or index, they
  read data the endpoint was scoped to hide; allowlist both.
- **Scripts run in the cluster.** Inline dynamic scripts built from input are code execution; stored scripts
  with bound parameters are the safe form.
- **The engine has no per-user document security by default.** Application-level scope is the only boundary
  unless document-level security is configured, so a widened query reads everything the service can.

## Worked example (a confirm and a kill)

> **Confirm.** A search endpoint merges the request body into the query DSL to support flexible filters. A
> crafted body adds a clause selecting a field the endpoint was meant to exclude and an aggregation over it,
> returning aggregated values of a sensitive field across the index. The response confirms field-selection
> injection. **Confirmed** search-engine query injection to cross-field disclosure, `high`, remediation =
> build the query from a fixed template that binds only the user's search term, and allowlist the fields,
> indices, and aggregations the endpoint may use.
>
> **Kill.** The endpoint binds the user term into a fixed match query over an allowlisted field set on a
> fixed index, never exposes `query_string`, and dynamic inline scripting is disabled cluster-wide with only
> stored scripts taking bound parameters. A crafted body cannot add clauses, choose a field, or run a script.
> **Killed**, `kill_reason` = "user term bound into a fixed template over allowlisted fields and indices, no
> raw query-string exposed, inline scripting disabled; no untrusted value becomes query or script structure."

## Rationalizations to reject

- *"We pass the search term straight through."* → If it enters as raw `query_string` or as merged JSON, the
  caller controls query structure, not just the term. Bind it into a fixed template.
- *"The engine is internal."* → The application scope is the only per-user boundary by default; a widened
  query reads across the index regardless of network placement.
- *"We only allow filtering."* → An attacker-chosen filter field or aggregation is exactly the disclosure
  vector; allowlist which fields and aggregations are permitted.
- *"Scripts are convenient for scoring."* → Inline dynamic scripts from input are code execution; use stored
  scripts with bound parameters and disable inline scripting.
- *"There is no sensitive data in search."* → Field and index selection can reach adjacent indices and fields
  the endpoint never intended to expose; confirm the allowlist, do not assume the contents.

## Executing this in practice

You need every search call split into term-binding and structure-carrying parts, the script sinks, the
untrusted inputs that reach each, and the field/index allowlist and scripting configuration. For each
structure-carrying sink, ask whether the caller can add clauses or pick fields and indices; for each script
sink, whether input reaches a script body. Reading the query construction shows the intent; selecting an
out-of-scope field or running a distinguishing script shows whether the boundary holds.

## Related

- `hunting-nosql-operator-and-where-injection` - the document-store sibling; both let a request object
  become query structure rather than a bound value.
- `hunting-orm-and-query-builder-injection` - the same value-versus-structure line applied to relational
  mappers.
- `auditing-graphql-attack-surface` - a query API is a common source of the filter and field objects that
  land in these search sinks; trace the two together.
- `adjudicating-taint-paths` - use it to confirm an untrusted field reaches the query DSL, a raw query
  string, or a script through framework indirection.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the untrusted value that becomes query or script
  structure, sink = the search or script call, evidence = the change in the query's reach.
