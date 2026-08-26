---
name: hunting-ldap-injection-and-bind-trust
description: >-
  Hunt LDAP injection and bind-trust flaws where untrusted input reaches a directory query or an
  authentication bind: a request value spliced into a search filter or a distinguished name without
  escaping, letting the caller alter the filter logic or the search base, and authentication flows that
  bind with attacker-influenced credentials in ways that permit anonymous or unauthenticated bind to pass
  as success. Covers filter metacharacter injection, DN injection that changes the subtree searched, and
  bind logic that treats an empty-password or anonymous bind as a valid login. Use when an application
  builds LDAP filters or DNs from input, or authenticates by binding to a directory. The untrusted value
  that becomes filter or DN structure is the source, the search or bind call is the sink, and the missing
  escape or the accepted anonymous bind is the bug.
license: MIT
---

# Hunting LDAP injection and bind trust: when a filter or a bind decides who you are

LDAP sits under a great deal of enterprise authentication and lookup, and it has two distinct
injection-shaped failures. The first is classic filter and distinguished-name injection: when a request
value is spliced into a search filter or a DN without escaping the directory's metacharacters, the
attacker rewrites the filter's logic (turning a scoped match into a match-anything clause) or redirects the
search to a different subtree. The second is a bind-trust failure: LDAP permits anonymous and
unauthenticated binds, and a login flow that binds with an empty password, or treats an anonymous bind as
success, authenticates an attacker who supplied no valid credential. You find these by tracing untrusted
input into filter and DN construction, and by reading the bind logic for the empty-credential and
anonymous cases.

## When to use

- An application builds LDAP search filters or distinguished names from request input.
- Authentication is performed by binding to a directory with user-supplied credentials.
- A directory lookup uses a search base or scope influenced by untrusted input.

## Scope check

Test LDAP queries and binds only against directories and applications you own or are authorized to assess,
on non-production data. A confirming filter can enumerate directory entries outside the intended scope and
a bind test touches authentication, so stay inside the authorized boundary. If you can't name the
authorization, stop.

## The loop

1. **Establish the filter, DN, and bind sinks first.** Inventory every LDAP search (its filter and base
   DN) and every authentication bind. This is the false-positive killer: a filter assembled entirely from
   fixed structure with properly escaped values, and a bind that rejects empty credentials, is not
   vulnerable. Name the search and bind sinks first.

2. **Trace untrusted input into filter construction.** Follow request values into the search filter. If a
   value is concatenated into the filter string without escaping the LDAP special characters, the attacker
   can inject a clause: closing the intended predicate and adding a match-anything condition, or altering
   the boolean logic. Confirm whether each interpolated value is escaped for the filter context.

3. **Trace untrusted input into distinguished names.** A DN built from input, a base DN or a bind DN, can be
   injected to change the subtree searched or the identity bound. DN escaping differs from filter escaping;
   confirm the correct one is applied where the value lands, because a value safe for a filter may still be
   unsafe in a DN.

4. **Read the bind logic for anonymous and empty-credential acceptance.** LDAP treats a bind with a valid
   DN and an empty password as an unauthenticated bind that can still return success, and it permits
   anonymous binds. A login flow that binds and checks only for the absence of an error, without rejecting
   empty passwords and without distinguishing an authenticated bind, accepts a no-credential login. Confirm
   the flow requires a non-empty password and treats anonymous or unauthenticated binds as failure.

5. **Check the defenses that actually stop it.** Context-correct escaping of every interpolated value
   (filter escaping for filters, DN escaping for DNs), parameterized or builder-based filter construction,
   an explicit non-empty-password check before binding, and rejecting anonymous binds in the auth path each
   remove a vector. A generic input sanitizer that does not match the LDAP context does not. Determine which
   stands at each sink.

6. **Confirm and record.** Confirm by supplying an in-scope input that changes the filter's meaning (a value
   that makes a scoped search match more than intended) or, for bind trust, an empty or anonymous credential
   that the flow accepts as a login, without enumerating out-of-scope entries. Kill the lead if every
   interpolated value is context-correctly escaped or built through a safe API, and the bind path rejects
   empty passwords and anonymous binds. Record the input, the sink, and the change in filter meaning or the
   accepted bind.

## Where LDAP injection and bind trust leak

- **Filter escaping and DN escaping are different.** A value escaped for a filter can still inject in a DN;
  the escaping must match where the value is placed.
- **A match-anything clause bypasses a scoped lookup.** Injecting a wildcard or an always-true predicate
  turns a search meant to find one entry into one that matches many.
- **An empty password can be a valid bind.** Directories accept a bind with a DN and no password as
  unauthenticated; a login that only checks for an error accepts it as success.
- **Anonymous bind returns success without a credential.** If the auth path does not reject anonymous binds,
  an attacker authenticates as nobody and is treated as somebody.
- **The search base is an injection target too.** Untrusted input in the base DN redirects the query to a
  subtree the caller was never meant to read.

## Worked example (a confirm and a kill)

> **Confirm.** A user-lookup endpoint builds a filter by concatenating the `username` parameter into the
> filter string without escaping. A crafted username closes the name predicate and adds a match-anything
> clause, returning directory entries the endpoint was scoped to hide. The response confirms filter logic
> injection. **Confirmed** LDAP filter injection to directory enumeration, `high`, remediation = escape
> every interpolated value with the LDAP filter-escaping rules or build the filter through a parameterizing
> API, and never concatenate raw input into a filter.
>
> **Kill.** Filters are built through a builder that escapes each value for the filter context and DNs
> through the DN-escaping API, the login flow rejects empty passwords before binding and treats anonymous or
> unauthenticated binds as authentication failure. A crafted filter value is escaped inert and an empty
> credential is refused. **Killed**, `kill_reason` = "filters and DNs are context-correctly escaped through
> safe APIs and the bind path rejects empty-password and anonymous binds; no untrusted value alters the
> query and no credential-less bind passes as login."

## Rationalizations to reject

- *"We escape the input."* → Confirm it is the LDAP context escaping and matches where the value lands;
  filter escaping does not make a value safe inside a DN.
- *"The directory returned an entry, so the login worked."* → An unauthenticated or anonymous bind can return
  success; the flow must require a non-empty password and reject anonymous binds.
- *"The filter is mostly fixed."* → One unescaped interpolated value is enough to inject a clause; the fixed
  parts around it do not protect it.
- *"Only internal users hit this."* → An injected filter or an accepted anonymous bind is an authorization
  break regardless of who reaches the endpoint.
- *"We validate the username format."* → A format check is not filter escaping; confirm the value is escaped
  for the exact context it enters.

## Executing this in practice

You need every LDAP search with its filter and base DN, every authentication bind, the untrusted inputs
that reach each, the escaping applied at each interpolation, and the bind path's handling of empty and
anonymous credentials. For each search sink, ask whether an interpolated value is context-correctly
escaped; for each bind, whether an empty or anonymous bind can pass as a login. Reading the construction and
the bind logic shows the intent; supplying a match-anything value or an empty credential shows whether the
boundary holds.

## Related

- `hunting-orm-and-query-builder-injection` - the same value-versus-structure distinction in relational
  queries; LDAP filters need the same escape-at-the-context discipline.
- `auditing-saml-and-oidc-flows` - directory binds often sit behind federation; a bind-trust gap and a
  federation gap can both authenticate the wrong principal.
- `finding-fail-open-flaws` - an auth bind that treats an anonymous or errored bind as success is a
  fail-open decision worth checking alongside this.
- `adjudicating-taint-paths` - use it to confirm untrusted input reaches a filter, a DN, or a bind through
  framework indirection.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the untrusted value that becomes filter or DN
  structure (or the missing credential), sink = the search or bind call, evidence = the altered filter
  meaning or the accepted anonymous bind.
