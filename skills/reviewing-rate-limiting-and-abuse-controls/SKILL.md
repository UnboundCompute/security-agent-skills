---
name: reviewing-rate-limiting-and-abuse-controls
description: >-
  Review whether sensitive and expensive endpoints are rate-limited and whether the limit can be
  bypassed, as a coverage-and-keying problem rather than a taint flow. Covers a login, credential-reset,
  one-time-code verify, signup, token, payment, or expensive-query endpoint with no limit on any layer
  of its path, a limit keyed on a client-supplied identifier the caller can rotate, a counter held per
  process so it multiplies across instances, a throttle with no lockout or backoff that permits slow
  brute force, and an enumeration oracle left unthrottled. Use when reviewing the handlers and
  configuration behind authentication, account, payment, and costly operations. The sensitive or
  expensive endpoint is the source, the enforcement decision and the identifier it is keyed on are the
  sink, and a missing or spoofable limit is the bug.
license: MIT
---

# Reviewing rate limiting and abuse controls: coverage, keying, and the layer you cannot see

Rate limiting is not a taint flow; it is a coverage-and-keying question. For each endpoint where
repetition is the attack, guessing a password, brute-forcing a one-time code, enumerating accounts,
draining a costly query, three things have to hold: a limit exists somewhere on the path, it is keyed
on something the caller cannot rotate, and it counts globally rather than per process. The failures are
an endpoint with no limit at all, a limit keyed on a header or field the attacker changes per request
to get a fresh bucket, a counter in local memory that multiplies by the number of instances, and a
throttle that resets cleanly forever with no lockout. The discipline that makes this review honest is
checking every layer, because the enforcement often lives at a gateway the handler never mentions.

## When to use

- An endpoint's abuse is repetition: login, credential reset, one-time-code verify, signup, token issuance, payment, invite or message send.
- An endpoint is expensive per call: an unbounded search, a report or export, a costly query the caller shapes.
- You want to know whether a limit exists, what it is keyed on, and whether it holds across instances.

## Scope check

Drive repetition against endpoints only on systems you own or are authorized to test, and coordinate:
a brute-force or enumeration demonstration can lock out real accounts or load a shared service. If you
can't name the authorization, stop.

## The loop

1. **Inventory the sensitive and expensive endpoints, then look for a limit on every layer.** List the
   repetition-sensitive and per-call-expensive endpoints. For each, check for a limiter not only in the
   handler but in the middleware chain and the gateway or edge configuration. An absence in the handler is
   not a missing limit until the other layers are clear; this step is first because it kills the most
   common false positive.

2. **Check what the limit is keyed on.** Read the expression that forms the counter key. If it derives from
   a client-supplied value, a forwarded client-address header the app does not pin to a trusted proxy, or a
   username, account, or device field on an unauthenticated endpoint, the attacker rotates it to get a
   fresh bucket per request and the limit is theater. A limit is real only when keyed on something
   server-trusted.

3. **Check global versus per-instance counting.** A counter kept in process memory, a local map or
   in-process cache, enforces its limit once per instance, so with several instances the effective limit is
   multiplied. Confirm the counter is a shared store, or that the service genuinely runs as a single
   instance, before trusting the number.

4. **Check for lockout and backoff, and the right endpoint.** A fixed window that fully resets each period
   with no escalating delay, temporary lock, or step-up permits slow, indefinite brute force. And confirm
   the limit is on the endpoint that verifies or mutates, not only on the sibling that sends or reads: a
   throttle on issuing a one-time code does nothing if verifying it is unbounded.

5. **Check the enumeration oracle and window bypasses.** An endpoint that reveals whether an account exists,
   by response, status, or timing, and is unthrottled, is a mass-enumeration tool. And a limit keyed on a
   path or method is bypassable if the path normalizes to the same handler under a different key, or if the
   limiter runs after the expensive work rather than before it.

6. **Confirm and record.** Confirm by exercising the endpoint past its limit with the bypass in hand,
   rotating the key, spreading across instances, or crossing the reset window, and showing the work still
   proceeds. Kill the lead if a limit is enforced at a gateway or middleware layer the handler does not
   show, if the forwarded-address header is pinned to a trusted proxy hop, if a spoofable key is backstopped
   by a second server-trusted key and a lockout, if the counter is a shared store, if the service is
   single-instance, or if the endpoint is neither sensitive nor expensive. Record the endpoint, the missing
   or spoofable control, and the volume achieved.

## Where abuse controls leak

- **The enforcement is often at a layer the handler never names.** Before calling a limit missing, read the
  middleware chain and the gateway or edge configuration; centralized limits are invisible in the handler.
- **A spoofable key is the same as no key.** A limit keyed on a header or body field the caller sets freely
  gives the attacker a fresh bucket per request; only a server-trusted identifier binds.
- **A per-process counter multiplies by your fleet.** A local, in-memory limit is enforced once per
  instance; the effective limit is the stated one times the instance count.
- **A throttle without a lockout is a speed limit, not a stop.** A window that resets cleanly forever lets
  a patient attacker brute-force a secret a few tries at a time, indefinitely.
- **The limit belongs on the verify, not just the send.** Bounding the cheap sibling while the sensitive
  verify or mutate is open moves the attack, it does not stop it.

## Worked example (a confirm and a kill)

> **Confirm.** The one-time-code verification endpoint has no limiter in the handler, the middleware chain,
> or the gateway configuration; codes are six digits valid for ten minutes; and the only counter in the
> system is a local in-memory map on the send endpoint, across four instances. An attacker brute-forces the
> code space against a target account faster than the code expires. **Confirmed** unbounded one-time-code
> verification enabling account takeover, `critical`, remediation = enforce a shared, server-keyed attempt
> limit with lockout on the verify endpoint, bound to the target account, before any code check.
>
> **Kill.** The credential-reset endpoint shows no limiter in the handler, but the gateway configuration
> enforces a shared per-client-address limit on that path with the address resolved from the trusted proxy
> hop rather than a client header, and a separate per-account lockout escalates after a few failures.
> Rotating a forwarded header does not create a new bucket, and the counter is shared across instances.
> **Killed**, `kill_reason` = "limit enforced at the gateway on a server-trusted, proxy-pinned key with a
> shared counter and a per-account lockout; header rotation and multiple instances do not bypass it."

## Rationalizations to reject

- *"There is no limiter in the handler, so there is none."* -> The limit is often at the gateway or
  middleware the handler never references. Clear those layers before calling it missing.
- *"We rate-limit per client address."* -> From a client-set forwarded header, unpinned? Then the attacker
  rotates it. Only an address resolved through a trusted proxy hop is trustworthy.
- *"We limit to a few per minute."* -> Per instance? Across a fleet that is a few times as many, and with a
  clean reset each minute it is unlimited over a day. Count globally and add a lockout.
- *"The send endpoint is throttled."* -> And the verify endpoint? The attack is on the code check, not the
  code send; the limit has to sit where the secret is tested.
- *"No one brute-forces a six-digit code."* -> They do, in the ten minutes it lives, when nothing throttles
  the verify. Cheap for them, an account for you.

## Executing this in practice

You need the list of sensitive and expensive endpoints, the limiter present at each layer of each path,
the exact key each limit is formed on and whether it is server-trusted, whether the counter is shared or
per process, the lockout and backoff behavior, and which of a send-and-verify pair is bounded. For each
endpoint, decide whether repetition is actually stopped and whether a key rotation, an extra instance, or
a reset window bypasses it. Reading the layers tells you what exists; exceeding the limit with the bypass
in hand tells you whether it holds.

## Related

- `hunting-business-logic-flaws` - an unmetered sensitive action is a resource the logic forgot to limit;
  both hunt a missing bound on how often something can be done.
- `hunting-redos-and-complexity-dos` - an expensive endpoint with no per-caller cost limit is the
  single-request cousin of the repetition abuse this skill meters.
- `auditing-multi-tenant-isolation` - a sibling systemic-property audit whose strongest signal is also an
  asymmetry: one path enforces and a peer does not.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the sensitive or expensive endpoint, sink = the
  enforcement decision and its key, evidence = the volume achieved past the intended limit.
