---
name: auditing-error-handling-and-information-exposure
description: >-
  Audit error handling and diagnostic surfaces for sensitive information a real client receives, where an
  exception path, a debug feature, or a diagnostic endpoint returns stack traces, database errors, internal
  paths, framework or version banners, configuration, secrets, or messages that differ enough to enumerate
  users. Use when reviewing how a service responds to malformed, unauthorized, or failing requests in its
  deployed configuration, and whether debug modes, source maps, or version-control metadata are exposed.
  Scoped to what production actually returns, not developer-only verbosity. The error or diagnostic path is
  the source, the response, header, or user-visible log is the sink, and disclosing detail that aids a
  further attack is the bug.
license: MIT
---

# Auditing error handling and information exposure: when a failure tells the attacker how it works

Every application fails, and how it fails is a message to whoever is probing it. A stack trace names the
framework, the file layout, and the line that broke. A database error quotes the query and confirms an
injection point. A verbose not-found versus a verbose forbidden tells an attacker which accounts exist. A
debug endpoint left on in production hands over configuration and sometimes secrets. None of this is a
memory-corruption bug or an injection; it is the system narrating its internals to an unauthenticated
client, turning blind probing into informed attack. The audit is not about suppressing all errors; it is
about deciding, for the deployed configuration, whether what a real client receives on failure reveals
security-relevant structure. You find it by driving the error paths a client can reach and reading exactly
what comes back.

## When to use

- A service returns errors on malformed, unauthorized, or failing requests and you can observe the responses.
- Debug modes, verbose error pages, or diagnostic and health endpoints may be reachable in production.
- Build or deployment artifacts (source maps, version-control metadata, backups) may be served publicly.

## Scope check

Test error and diagnostic surfaces only against systems you own or are authorized to assess, using benign
malformed input rather than live exploitation to trigger failures, and treat any disclosed secret as a real
credential to be reported and rotated, not reused. Stay within the authorized scope. If you can't name the
authorization, stop.

## The loop

1. **Establish what a real client receives in the deployed configuration first.** Determine whether the
   production error handler is active and debug mode is off, so the verbose page you see locally is not what a
   client actually gets. This is the false-positive killer: a detailed trace visible only under a developer
   flag that is disabled in production is not a finding, while the generic page that production returns is the
   real evidence. Confirm the deployed behavior before judging the detail.

2. **Enumerate the reachable failure paths.** Drive the errors a client can cause: malformed bodies, wrong
   types, oversized input, unauthorized and forbidden access, missing resources, and backend failures.
   Include diagnostic and health endpoints, and default framework error routes. Each response and its headers
   are a candidate sink.

3. **Read what the response discloses.** For each failure, judge whether the body, status, or headers reveal
   security-relevant detail: a stack trace, a database or template error, an internal file path or hostname,
   a framework and version banner, configuration values, or a secret. Cosmetic verbosity that names nothing
   useful is not the finding; structure an attacker can act on is.

4. **Check for enumeration through differentials.** Compare responses for existing versus non-existing
   accounts, valid versus invalid credentials, and authorized versus unauthorized resources, in wording,
   status, and timing. A difference that reliably distinguishes the two enumerates users or resources even
   when no single response looks verbose.

5. **Check for exposed artifacts and debug surfaces.** Look for served source maps, version-control
   directories, backup and configuration files, verbose health or debug endpoints, and default sample pages,
   which disclose source, structure, or secrets independent of any exception path.

6. **Confirm and record.** Confirm by triggering the failure as an unauthenticated or ordinary client in the
   deployed configuration and capturing the disclosed detail, or by demonstrating the reliable differential.
   Kill the lead if production returns a generic handler, if the verbose output is gated behind a disabled
   developer flag, if the disclosed detail names nothing an attacker can use, or if responses and timing do
   not distinguish existence. Record the failure path, the sink, the disclosed detail, and its use to an
   attacker, or set a `kill_reason`.

## Where information exposure leaks

- **The gap is local versus production.** Verbose errors in development are expected; the finding is whether
  the same verbosity survives into the deployed configuration a client can reach.
- **Errors confirm other bugs.** A quoted database or template error turns a blind injection probe into a
  confirmed one, so verbose backend errors are force multipliers, not just leaks.
- **Differentials enumerate quietly.** Existence disclosure rarely looks like an error; it hides in a wording,
  status, or timing difference between the present and absent case.
- **Artifacts leak without an exception.** Served source maps, version-control metadata, and backups disclose
  source and structure through ordinary requests, no failure required.
- **Headers talk too.** Framework and version banners and debug headers name the stack and its version,
  narrowing which known weaknesses an attacker tries.

## Worked example (a confirm and a kill)

> **Confirm.** A malformed request to an ordinary endpoint returns the framework's default error page with a
> full stack trace, the source file paths, and the database query that failed, to an unauthenticated client
> in production. **Confirmed** sensitive information disclosure through verbose errors, `medium`, remediation
> = enable the production error handler with generic client-facing messages, disable debug mode in the
> deployed configuration, log detail server-side only, and remove framework and version banners from
> responses.
>
> **Kill.** The same request returns a generic error with a correlation identifier and no internal detail,
> debug mode is off in the deployed configuration, the login and lookup responses are identical for present
> and absent accounts in wording, status, and timing, and no source maps or version-control metadata are
> served. Nothing in the failure names the stack, the query, or an account's existence. **Killed**,
> `kill_reason` = "production returns generic errors with detail only in server-side logs, no debug or banner
> disclosure, and existence differentials are absent; failures reveal nothing an attacker can act on."

## Rationalizations to reject

- *"That trace only shows in development."* -> Then confirm it is disabled in the deployed configuration; the
  finding is what a real client receives in production, which is where you must check.
- *"It is just an error message."* -> A quoted query or a named internal path confirms an injection point or
  maps the system; the message is intelligence, not noise.
- *"Different messages are more helpful to users."* -> A login that distinguishes unknown user from wrong
  password enumerates accounts; be equally uninformative for both cases.
- *"Source maps help us debug production."* -> Served publicly they hand your source to anyone; restrict them
  or strip them from the public build.
- *"The health endpoint is internal."* -> If it is reachable without authentication it is a client surface;
  confirm the access control, not the intent.

## Executing this in practice

You need the deployed error-handling configuration, the responses and headers for the full set of reachable
failure paths, the present-versus-absent differentials on authentication and lookup surfaces, and whether
any build or version-control artifacts are served. For each, decide whether the detail is security-relevant
and whether it reaches an unauthorized client in production. Driving the failures as an ordinary client and
reading the exact responses settles most leads; comparing wording, status, and timing settles the
enumeration ones.

## Related

- `auditing-security-logging-completeness` - the complement, where detail belongs in server-side logs rather
  than in client responses; the two together place each piece of information where it should be.
- `hunting-blind-and-second-order-sql-injection` - verbose database errors turn that skill's blind probes
  into confirmed ones, so error disclosure directly aids it.
- `auditing-session-lifecycle-and-fixation` - authentication response differentials that enumerate accounts
  are adjacent to the session and login concerns audited there.
- `reviewing-rate-limiting-and-abuse-controls` - enumeration through differentials is amplified without rate
  limits, so the two controls are assessed together on login and lookup surfaces.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the error or diagnostic path, sink = the response,
  header, or user-visible log, evidence = the disclosed detail or the reliable differential captured as an
  unauthorized client in the deployed configuration.
