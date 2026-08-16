---
name: finding-fail-open-flaws
description: >-
  Find security controls that grant access when they should deny it: an
  authorization check that returns allow on error or timeout, an empty or wildcard
  allowlist that matches everything, a default-allow branch when input is missing or
  unrecognized, and a caught exception that swallows a denial and continues. Use when
  reviewing authentication, authorization, or any gate whose failure path matters, or
  when a check "passes" for reasons you have not confirmed. The dangerous default is
  allow; prove every gate denies by default.
license: MIT
---

# Finding fail-open flaws: prove the gate denies by default

A control fails open when its error, empty, or default path grants access instead of
refusing it. The check looks present and even passes its happy-path tests, but on the
branch that runs when something goes wrong, when a lookup errors, a list is empty, a
value is missing, an exception is caught, it lets the request through. These flaws
hide in the paths tests rarely cover, and they turn any upstream failure into an
authorization bypass.

## When to use

- You are reviewing authentication, authorization, or any access gate.
- A check "passes" and you have not confirmed why, or what it does on failure.
- A control depends on an external service, a list, or an input that could be absent.

## Scope check

Test gates in code you own or are authorized to test, inducing failures against your
own environment. If you can't name the authorization, stop.

## The loop

1. **Enumerate the gates and their failure branches.** For each authentication or
   authorization check in scope, find not just the allow/deny decision but what
   happens when the check cannot be completed: the lookup throws, the policy service
   is unreachable, the input is null or unrecognized, the list is empty. Every gate
   has a failure branch; find it.

2. **Determine the default.** For each gate, is the default deny (access requires an
   explicit, successful allow) or allow (access proceeds unless something explicitly
   denies)? Default-allow is the root fail-open shape: anything that prevents the deny
   from firing grants access. State the default for every gate.

3. **Test the error path.** Force the check to fail (an unreachable dependency, a
   malformed input, an exception) and observe the outcome. If a caught exception, a
   timeout, or an error returns or falls through to allow, the gate fails open. A
   try/catch around a permission check that continues in the catch is the classic
   instance.

4. **Test the empty and wildcard cases.** Does an empty allowlist match nothing (safe)
   or everything (fail-open)? Does a wildcard, a missing filter, or an unset scope
   collapse to allow-all? Check what the control does with no rules and with a
   catch-all rule; both are common allow-everything defaults.

5. **Test the missing-input case.** When an identifier, role, or token is absent or
   unrecognized, does the gate treat it as unauthorized, or does a missing value skip
   the check or select a permissive default (an unknown role mapped to allow, a null
   user treated as trusted)? Absent input must deny.

6. **Confirm and record.** A fail-open flaw is confirmed by driving the gate down its
   failure path and observing access granted; name the branch and the trigger. Kill
   the lead if every failure, empty, and missing case denies. Record with the exact
   failing branch and the fix: default deny, fail closed on error, treat empty as
   match-nothing, and reject missing input.

## Where controls fail open

- **The happy path lies.** A gate that allows correctly can still allow on error; the
  tests rarely exercise the error branch.
- **Empty is ambiguous.** An empty allowlist is safe only if empty means deny. Confirm
  which it means.
- **Catch-and-continue is allow-on-error.** Swallowing a denial and proceeding is the
  most common fail-open pattern.
- **Missing is not trusted.** An absent role or token is unauthorized, not
  default-privileged.

## Worked example (a confirm and a kill)

> **Confirm.** An authorization middleware calls a policy service and wraps it in a
> try/catch; on any exception it logs and continues to the handler, reasoning that the
> service is usually up. Making the policy service time out causes every request to be
> authorized. **Confirmed** fail-open on error, `critical`, remediation = deny on any
> policy-check failure, never continue in the catch.
>
> **Kill.** A gate requires an explicit allow decision, denies on any exception or
> timeout, treats an empty allowlist as matching nothing, rejects requests with a
> missing or unrecognized role, and has no catch-all allow. Every forced failure and
> empty case results in denial. **Killed**, `kill_reason` = "default deny, fail closed
> on error, empty means match-nothing, missing input rejected; no branch grants on
> failure."

## Rationalizations to reject

- *"The check is right there."* → Presence is not enough. Read its error, empty, and
  missing branches.
- *"The dependency is reliable."* → Reliability is not a control. If its failure grants
  access, an attacker will cause the failure.
- *"An empty list means nothing is configured yet."* → Then it must match nothing. If
  empty means allow-all, it is a bypass.
- *"We catch the exception so it doesn't crash."* → Not crashing by allowing is worse
  than crashing. Fail closed.

## Executing this in practice

You need to see each gate's failure, empty, and missing branches, and to force those
conditions (kill a dependency, send malformed or absent input) while observing whether
access is granted. A call graph that shows the guard and its siblings helps you find
every gate and compare their defaults; forcing the failure path is the confirmation.

## Related

- `auditing-guard-gaps` - the unguarded-peer analysis; this is its failure-path
  complement.
- `auditing-declarative-authorization` - fail-open defaults expressed in policy and
  configuration.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the induced failure or
  empty/missing input, sink = the access the gate grants on that branch.
