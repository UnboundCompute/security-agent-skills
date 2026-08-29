---
name: hunting-blind-and-second-order-sql-injection
description: >-
  Hunt the SQL injection that first-order testing misses: blind injection where the response carries no
  error or data and the signal is a boolean difference or a timing delay, and second-order injection where
  input is stored safely on one request and later concatenated into a query on a different code path. Covers
  values that reach a query only after being read back from the database, a cache, or a log, and sinks that
  reveal nothing directly so confirmation depends on an inferential channel. Use when a value is stored now
  and used in a query later, or when a parameter reaches a query whose response shows no direct output. The
  untrusted value that becomes query structure at the later or blind sink is the source, that query is the
  sink, and the unparameterized use on the second path is the bug.
license: MIT
---

# Hunting blind and second-order SQL injection: the paths first-order testing walks past

Parameterized queries have made obvious SQL injection rare, but two variants survive because they defeat the
usual way people test. Blind injection reaches a real query, but the response returns no error and no data,
so a single probe looks inert; the vulnerability is real and confirmable only through an inferential
channel, a boolean that flips the response or a deliberate time delay. Second-order injection is subtler:
the input is stored correctly and safely on the request that accepts it, then read back later and
concatenated into a query on an entirely different code path, so the endpoint that accepts the payload is
not the endpoint that is vulnerable. You find both by following stored values to their later query uses and
by treating no-output sinks as blind rather than safe.

## When to use

- A value is stored on one request and later used to build a query on a different code path.
- A parameter reaches a query whose response shows no error and no returned data.
- Data read back from the database, a cache, a log, or a queue is concatenated into a later query.

## Scope check

Test SQL injection only against databases and applications you own or are authorized to assess, on
non-production data. A confirming boolean or timing probe still executes against the database and a
destructive payload must never be used, so stay inside the authorized boundary. If you can't name the
authorization, stop.

## The loop

1. **Establish every query sink, including the silent ones.** Inventory all query construction, and mark the
   sinks whose responses reveal nothing: background jobs, count-only checks, existence probes, and endpoints
   that return a fixed status regardless of the row. This is the false-positive killer in reverse: a silent
   sink is not a safe sink, it is a blind one, and it must be tested inferentially rather than dismissed.
   Name all sinks, silent ones included.

2. **Follow stored values to their later query uses.** For second-order injection, trace each value from the
   request that stores it to every later read: a profile field, a name, a comment, a cached lookup, a log
   line re-parsed by a job. At each later read, ask whether the value is concatenated into a query or bound
   as a parameter. The storing request being safe tells you nothing about the using request.

3. **Confirm parameterization at the actual sink.** For each query, blind or second-order, determine whether
   the untrusted value is a bound parameter or spliced into the SQL text. A value that was escaped for HTML
   or safe for storage is not therefore safe for a query; the only thing that matters at the sink is whether
   it is bound. Identifiers and dynamic clauses are never bound and need allowlisting.

4. **Choose the confirmation channel for blind sinks.** When the sink returns no data, confirm through a
   boolean difference (an input that makes a true condition and a false condition produce distinguishable
   responses) or a timing difference (a conditional delay). These prove the injection without extracting
   out-of-scope data, and they are the only reliable signal a blind sink gives.

5. **Check the defenses that actually stop it.** Parameterized queries at every sink, including background
   and second-order paths, and allowlisting for any identifier or dynamic clause, remove both variants. A
   sanitizer applied at input does not, because a stored value can be re-fetched and used raw later, and an
   escape for one context is wrong for another. Determine whether the actual query sink binds the value.

6. **Confirm and record.** Confirm by using the inferential channel in scope: a boolean or timing probe at a
   blind sink, or storing a benign marker payload and observing it alter a later query's meaning at the
   second-order sink, without extracting out-of-scope rows. Kill the lead if the query at the actual sink,
   later or blind, binds the value as a parameter and allowlists any identifier, so no stored or reflected
   value becomes SQL text. Record the storing path (if any), the query sink, and the inferential evidence.

## Where blind and second-order SQL injection leaks

- **A silent response is a blind sink, not a safe one.** No error and no data means test inferentially;
  dismissing it because a probe looked inert is the miss.
- **The vulnerable path is not the accepting path.** Second-order injection fires where a stored value is
  read and concatenated later, often in a job or a different endpoint than the one that stored it.
- **Safe-for-storage is not safe-for-query.** A value escaped for storage or display is still raw text to a
  query; only binding at the query sink protects it.
- **Background jobs and reports are common second-order sinks.** They read stored data and build queries away
  from request-time review, so they escape first-order testing.
- **Identifiers are never parameterized.** A stored value used as a column, table, or order term needs an
  allowlist; binding does not apply to it.

## Worked example (a confirm and a kill)

> **Confirm.** A profile endpoint stores a display name with proper escaping. A nightly analytics job reads
> display names and concatenates each into a query string to group activity. A name stored with a crafted
> payload alters that job's query; a boolean-shaped marker makes the job's derived output differ measurably,
> confirming the injection on the second path. **Confirmed** second-order SQL injection in the analytics
> job, `high`, remediation = parameterize the job's query so the stored name is bound, never concatenated,
> and allowlist any identifier the job selects from stored data.
>
> **Kill.** Every query, including the analytics job and the existence-check endpoints that return only a
> status, binds untrusted values as parameters and allowlists identifiers; a stored marker payload is bound
> inert at the later sink and a boolean probe at the silent endpoint produces no distinguishable difference.
> **Killed**, `kill_reason` = "all sinks including second-order and blind paths bind values as parameters and
> allowlist identifiers; no stored or reflected value becomes SQL text and inferential probes show no signal."

## Rationalizations to reject

- *"The probe returned nothing, so it is safe."* → A blind sink returns nothing by design; confirm with a
  boolean or timing channel before concluding it is safe.
- *"We sanitize on input."* → A stored value is re-read and used later, and an input-time escape is the wrong
  context for a query. Bind at the query sink.
- *"This endpoint uses parameterized queries."* → Check the endpoint that uses the value later, not the one
  that stores it; second-order injection lives on the second path.
- *"It only groups or counts."* → A blind sink that counts or groups still executes injected SQL; the impact
  is confirmed inferentially, not by returned rows.
- *"The value was already validated."* → Validation for format or display is not parameterization; the query
  sink is the only place that decides.

## Executing this in practice

You need every query sink including silent ones, the storage-to-use paths for values that are stored and
later queried, and whether each actual sink binds the value or allowlists an identifier. For each stored
value, follow it to every later query; for each silent sink, plan a boolean or timing confirmation. Reading
the sink shows whether the value is bound; an inferential probe or a stored marker shows whether a blind or
second-order path is live.

## Related

- `hunting-orm-and-query-builder-injection` - the raw-method and identifier escape hatches there are common
  second-order sinks when a stored value flows into them later.
- `hunting-nosql-operator-and-where-injection` - the document-store analog, where the same stored-then-used
  path applies to operator injection.
- `adjudicating-taint-paths` - use it to connect a storing request to a distant query sink across code paths
  and jobs.
- `writing-vuln-reports` - blind and second-order findings need a precise inferential-evidence writeup; use
  it to record the boolean or timing channel clearly.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the untrusted value that becomes SQL text at the
  later or blind sink, sink = that query, evidence = the boolean, timing, or stored-marker signal.
