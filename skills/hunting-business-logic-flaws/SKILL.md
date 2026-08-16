---
name: hunting-business-logic-flaws
description: >-
  Hunt for vulnerabilities that live in what an application is allowed to do, not
  in how it is coded: workflow steps that can be skipped or reordered,
  quantity/price/limit values that go negative or overflow a cap, state transitions
  that should be unreachable, replay and concurrency abuse, and privileged outcomes
  reached through a sequence of individually-valid requests. Use when reviewing
  checkout, transfers, redemption, quotas, or any rule the code enforces implicitly.
  These are the flaws static analysis and scanners structurally miss.
license: MIT
---

# Hunting business-logic flaws: the bugs where every request is valid

A business-logic flaw is a gap between what the code enforces and what the business
intends. No request is malformed, no payload is injected, no signature is invalid;
the attacker just does allowed things in a disallowed order, quantity, or
combination. Scanners cannot find these, because there is no dangerous function call
to flag. The danger is the sequence and the arithmetic, so finding them means
modeling the intended rule first, then searching for a permitted path that breaks
it.

## When to use

- You are reviewing checkout, payment, transfers, refunds, or redemption.
- A multi-step flow with a required order (signup, approval, provisioning).
- Anything with a quota, a limit, a balance, a tier, or a one-time action.

## Scope check

Test flows you own or are authorized to test. Exercise sequences against your own
accounts and data; do not move real money or affect real users. If you can't name
the authorization, stop.

## The loop

1. **Recover the intended rules.** For the flow under review, write down the
   invariants the business assumes: "a coupon applies once," "you cannot ship what
   you did not pay for," "a transfer cannot exceed the balance," "step 3 requires
   step 2." These are usually unwritten. State each explicitly, because each is a
   hypothesis you will try to break.

2. **Map the state machine and its real transitions.** Diagram the states and the
   requests that move between them. Then find transitions the diagram forbids but the
   code permits: can you post the final step directly, replay an approval, revisit a
   completed state, or reach a state without its precondition? Missing server-side
   enforcement of order is step-skipping.

3. **Attack the arithmetic and the limits.** For every quantity, price, count, or
   balance the user influences, test the edges: negative, zero, very large,
   fractional and rounding boundaries, and the value that pushes a cumulative total
   past a cap. A quantity that can go negative credits the attacker; a limit checked
   before but not after a concurrent change is a limit-overrun.

4. **Attack ordering, replay, and concurrency.** Can a permitted request be replayed
   (redeem a code twice), sent out of order, or fired concurrently to beat a
   check-then-act on a shared limit (withdraw the same balance twice before either
   debits)? Logic that validates and then acts non-atomically is a race.

5. **Attack authorization of outcomes, not just endpoints.** Even if every endpoint
   checks "is this user logged in," does it check that this user is allowed this
   object, this price, this tier? Reaching a premium outcome through a permitted
   sequence (a paywall or tier bypass) is a logic flaw even when every request is
   authenticated.

6. **Confirm by executing the sequence.** A logic flaw is confirmed by walking the
   concrete steps that reach the disallowed outcome, in order, and observing the
   violated invariant: a negative charge, a doubled redemption, a skipped payment.
   Record the sequence as the repro. Kill the hypothesis if the server enforces the
   invariant you tried to break.

7. **Record.** Each confirmed flaw names the intended rule, the permitted sequence
   that broke it, and the impact. Severity is the business impact (money, access,
   data), not a generic class score.

## Why scanners can't see these

- **There is no tainted source-to-sink path.** The inputs are valid; the
  vulnerability is the semantics, and a tool has no dangerous call to match.
- **The rule lives in the business, not the code.** A tool cannot compare behavior
  against an intent it was never told.
- **The exploit is a sequence of legitimate requests**, which no signature or filter
  flags.
- **Finding them is a modeling exercise first.** You must know the intended rule to
  see the violation.

## Worked example (a confirm and a kill)

> **Confirm.** A checkout applies a discount code, then lets the cart be edited before
> payment. The intended rule is "the discount applies to the paid cart." The attacker
> applies a high-value code to a qualifying cart, removes the items after the discount
> is locked in, and pays a near-zero total. Every request is valid; the sequence
> breaks the invariant. **Confirmed** business-logic flaw (discount-then-edit),
> `high`, remediation = re-validate the discount against the final cart at payment,
> server-side.
>
> **Kill.** A funds transfer checks the balance and debits it in a single atomic
> transaction with a row lock, re-checks the limit at commit, and rejects negative and
> over-balance amounts server-side. Concurrent and replayed transfers either serialize
> or fail the re-check. **Killed**, `kill_reason` = "invariant enforced atomically at
> commit; amount bounds and limit re-checked server-side; no permitted sequence
> violates it."

## Rationalizations to reject

- *"Every request is authenticated and valid."* → That is why it is a logic flaw. The
  abuse is the sequence, not a bad request.
- *"The scanner found nothing here."* → Scanners cannot model intent. Absence of a
  flagged sink is not absence of a flaw.
- *"The UI won't let you do that."* → The UI is not the boundary. Replay the requests
  directly and see what the server enforces.
- *"The check happens on the first request."* → If state changes after the check and
  is not re-validated, the check is stale.

## Executing this in practice

You need the intended business rules (ask, or infer them from the flow), the ability
to replay and reorder the real requests outside the UI, and a way to observe
server-side state: the charged amount, the redemption count, the reached state. A
call graph helps you see where a check is and is not re-applied, but the core method
is modeling the invariant and searching for a permitted path that breaks it.

## Related

- `detecting-race-conditions` - the concurrency and check-then-act half of
  limit-overrun and replay.
- `auditing-guard-gaps` - the authorized-endpoint-but-wrong-object half of outcome
  authorization.
- `mapping-attack-surface` - enumerating the multi-step flows worth modeling.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the permitted request
  sequence, sink = the violated business invariant.
