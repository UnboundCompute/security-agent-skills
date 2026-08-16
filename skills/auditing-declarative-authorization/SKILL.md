---
name: auditing-declarative-authorization
description: >-
  Audit authorization expressed as configuration or framework convention rather than
  inline code: row-level security and policy rules, framework before-action and
  middleware filters that must be attached to every protected route, serverless and
  gateway access rules, and object-ownership checks. Covers routes that skip the
  filter, policies with a permissive default, rules that check authentication but not
  ownership, and gaps between where the rule is declared and where the data is
  accessed. Use when reviewing role- or policy-driven access control. Coverage and
  correctness are separate checks.
license: MIT
---

# Auditing declarative authorization: coverage and correctness are two audits

Modern authorization is often declarative: a policy attached to a table, a filter
registered on a controller, a rule in a gateway or serverless config. The strength of
that model is also its weakness, security depends on the rule being attached to every
path that needs it, and on the rule actually expressing the intended constraint. The
failures are gaps (a route the filter never covers) and mis-statements (a rule that
checks the wrong thing), and both are invisible if you only read the routes that are
protected.

## When to use

- You are reviewing role- or policy-driven access control.
- Authorization is enforced by row-level policies, controller filters, middleware, or
  gateway rules.
- Protection depends on a rule being attached per route, per table, or per resource.

## Scope check

Audit access control in systems you own or are authorized to test, with test accounts
across roles and tenants. If you can't name the authorization, stop.

## The loop

1. **Locate every declared rule and its scope.** Inventory the authorization rules:
   row-level policies, controller filters, middleware, gateway or serverless access
   rules, and ownership checks. For each, determine exactly which paths, tables, or
   resources it covers. The audit is about coverage and correctness, so start by
   mapping what each rule protects.

2. **Find the paths the rule does not cover.** Enumerate every route, query, or
   resource access and check which are not covered by any rule. A framework filter that
   must be added per controller is missing wherever a developer forgot it; a policy on
   one table does not protect a sibling table or a raw query that bypasses the policy
   layer. The uncovered path is the bug.

3. **Check the default of the policy layer.** When no rule matches, does access default
   to deny or allow? A row-level-security system with policies enabled but a permissive
   fallback, or a gateway that allows unmatched routes, grants access wherever a rule
   is absent. Confirm the default is deny.

4. **Check that the rule states ownership, not just presence.** Does the rule confirm
   this user may access this specific object, or only that the user is authenticated? A
   filter that checks "logged in" but not "owns this record," or a policy that scopes
   by table but not by row or tenant, lets any authenticated user reach another's data.
   Authentication is not authorization.

5. **Check the declaration-to-access gap.** Is the rule enforced at the same layer the
   data is actually accessed, or can a code path reach the data beneath the rule (a raw
   query under row-level security, a direct service call under a gateway rule, an admin
   path that skips the middleware)? A rule declared in one place and data reached in
   another is unenforced there.

6. **Confirm and record.** Confirm a gap by accessing a protected resource through the
   uncovered path or as the wrong user; confirm a mis-statement by reaching another
   user's object through the rule. Kill the lead if every path is covered, the default
   is deny, and rules enforce ownership at the access layer. Record with the exact
   uncovered path or wrong-object access.

## Where declarative authz leaks

- **Coverage is per path, correctness is per rule.** Check both; the protected routes
  tell you nothing about the unprotected ones.
- **The default when no rule matches is the whole ballgame.** Deny-by-default contains
  mistakes; allow-by-default amplifies them.
- **Authenticated is not authorized.** A filter that stops at identity, not ownership,
  is a horizontal-access bug.
- **Rules and data can drift apart.** A path that reaches the data below the rule's
  layer is unprotected no matter how the rule reads.

## Worked example (a confirm and a kill)

> **Confirm.** An app enforces access with a per-controller before-action filter. A
> newer controller exposing the same records was added without the filter. Requesting
> those records through the new controller returns any user's data with only a valid
> session. **Confirmed** missing-filter gap, `high`, remediation = enforce
> authorization globally with explicit opt-out, or add the filter and a test that every
> controller is covered.
>
> **Kill.** Row-level security is enabled with a deny-by-default fallback, every table
> has an ownership-scoped policy keyed to the tenant and user, raw queries run under the
> same policy, and no code path reaches the data beneath the policy layer. Cross-user
> and uncovered-path attempts return nothing. **Killed**, `kill_reason` =
> "deny-by-default, ownership-scoped policies on every table enforced at the data layer,
> no bypass path."

## Rationalizations to reject

- *"Authorization is handled by the framework."* → Only where the rule is attached.
  Enumerate the paths that lack it.
- *"The policy is enabled."* → Enabled with what default and what scope? A permissive
  default or table-only scope still leaks.
- *"The user is authenticated."* → Authentication is not ownership. Check that the rule
  constrains which object, not just which user.
- *"The rule is declared on the model."* → And the data is reached where? A path under
  the rule's layer is unprotected.

## Executing this in practice

You need the full set of declared rules and their scopes, a complete enumeration of
routes and data accesses to find the uncovered ones, and the ability to request
protected resources through each path and as different users. A call graph that maps
every access to the data and shows which are covered by a guard is ideal; the
coverage-and-correctness pass is the method.

## Related

- `auditing-guard-gaps` - the unguarded-peer pattern, here applied to declarative
  rules and routes.
- `finding-fail-open-flaws` - the permissive-default failure mode of a policy layer.
- `hunting-business-logic-flaws` - authorization of outcomes reached through a
  permitted path.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the uncovered path or
  wrong-user request, sink = the protected resource it reaches.
