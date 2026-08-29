---
name: auditing-payment-state-machine-and-idempotency
description: >-
  Audit payment and checkout state machines for transitions an attacker can drive out of order or replay for
  value: an order marked paid before the charge is confirmed, a step that can be skipped or repeated so goods
  ship without settlement, a non-idempotent charge or fulfillment endpoint that double-processes on a retried or
  replayed request, and a refund or cancel that returns value while the underlying charge stays captured. Covers
  checkout, charge, fulfillment, and refund flows where money and goods change hands across a sequence of state
  transitions. Use when a purchase moves through ordered payment states and the transitions and their idempotency
  are the boundary. The out-of-order or replayed transition is the source, the value released without settlement
  is the sink, and the skippable step or non-idempotent handler that allows it is the bug.
license: MIT
---

# Auditing payment state machine and idempotency: value must not move faster than settlement

A checkout is a state machine, and money moves through it in a strict order: an order is placed, a charge is
authorized, the charge settles, and only then are goods fulfilled; refunds and cancels run the sequence
backward. The bugs are transitions that break the order or repeat for value. An order marked paid before the
charge is actually confirmed lets fulfillment run against a payment that never settles. A step that can be
skipped, so fulfillment triggers without a completed charge, or repeated, so one payment yields multiple
fulfillments, releases value the sequence did not authorize. A charge or fulfillment endpoint that is not
idempotent double-processes when a request is retried or deliberately replayed, so a network retry or an
attacker's repeat becomes two charges or two shipments. And a refund or cancel that returns value while the
charge stays captured pays out twice. The audit walks the state machine and tries to move value ahead of, or
more than once per, settlement. You audit this by driving transitions out of order and replaying the ones that
release value.

## When to use

- A purchase moves through ordered payment states (placed, authorized, settled, fulfilled, refunded).
- Fulfillment may trigger before a charge is confirmed, or a paid state may be set ahead of settlement.
- Charge, fulfillment, or refund endpoints may not be idempotent under retry or replay.

## Scope check

Test payment flows only against systems you own or are authorized to assess, on non-production or sandbox
payment paths. Driving transitions and replays moves real value on a live system, so use sandbox payment
credentials and test accounts and never trigger a real charge, refund, or fulfillment that is not yours. If you
can't name the authorization, stop.

## The loop

1. **Establish the intended state machine first.** Map the legal states and the only transitions allowed between
   them, and which transition each value-releasing action (fulfillment, refund, payout) requires as its
   precondition. This is the false-positive killer: a flow where fulfillment requires a confirmed settled charge,
   every transition validates its precondition server-side, and every value-releasing handler is idempotent is
   correct. Name the intended sequence, then test deviations.

2. **Check the paid-state precondition.** Confirm the state that gates fulfillment is set only after the charge
   is confirmed settled by the payment provider, not on client assertion or on authorization alone. An order
   marked paid before settlement, or on a client-supplied status, lets goods ship against a payment that may
   never complete.

3. **Test step skipping and out-of-order transitions.** Attempt to reach a value-releasing state without the
   transitions that should precede it: trigger fulfillment without a completed charge, jump past a verification
   step, or set a later state directly. Confirm each transition validates its precondition on the server and
   refuses an out-of-order jump. A state reachable without its prerequisites is a skip.

4. **Test replay and idempotency of value-releasing handlers.** Replay the charge, fulfillment, and payout
   requests, and retry them with and without any idempotency key. Confirm each is idempotent: a repeated request
   for the same logical operation processes once, not once per delivery. A non-idempotent handler turns a retry
   or a deliberate replay into a double charge or double fulfillment.

5. **Check refund and cancel symmetry.** Confirm a refund or cancel only returns value that was actually
   captured, moves the state consistently, and cannot be repeated to pay out more than once or run while the
   charge remains captured. A refund that credits without reversing the charge, or that replays, is a payout
   primitive.

6. **Confirm and record.** Confirm by releasing value without settlement, a fulfillment on an unconfirmed
   charge, a skipped step reaching a paid state, a replayed handler double-processing, or a repeated refund, all
   on sandbox payment paths and without moving real money. Kill the lead if every value-releasing action requires
   its settled precondition validated server-side and every such handler is idempotent under retry and replay.
   Record the out-of-order or replayed transition, the value-released sink, and the skippable step or
   non-idempotent handler.

## Where payment state trust leaks

- **Paid set before settlement.** Marking an order paid on authorization alone or on client-supplied status
  ships goods against a charge that may never settle.
- **A skippable step reaches fulfillment.** A value-releasing state reachable without its preceding transitions
  lets fulfillment run without a completed charge.
- **A non-idempotent handler double-processes.** A charge, fulfillment, or payout endpoint that acts once per
  request rather than once per operation doubles on retry or replay.
- **A repeatable refund pays out twice.** A refund or cancel that can be replayed, or that credits without
  reversing the capture, returns more value than was taken.
- **Client-driven state transitions.** Trusting a client to assert the next state, rather than deriving it from
  server-confirmed events, lets an attacker set the state that releases value.

## Worked example (a confirm and a kill)

> **Confirm.** A checkout marks the order paid and triggers fulfillment when the browser posts back a success
> status after redirect from the payment page, before the server confirms settlement with the provider. Posting
> the success callback directly, without any real payment, drives the order to paid and ships the goods.
> Separately, replaying the fulfillment request produces a second shipment for one order. **Confirmed** payment
> state bypass via client-asserted paid state and non-idempotent fulfillment, `critical`, remediation = set the
> paid state only from a server-side confirmation of settlement with the provider, require the settled
> precondition before fulfillment, and make fulfillment idempotent per order.
>
> **Kill.** The paid state is set only after the server confirms settlement with the payment provider out of
> band; fulfillment, charge, and payout each validate their settled precondition server-side and refuse an
> out-of-order jump; every value-releasing handler is idempotent per logical operation under retry and replay;
> and refunds return only captured value and cannot repeat. A skipped step or replayed request releases nothing
> extra. **Killed**, `kill_reason` = "value-releasing transitions gated on server-confirmed settlement and
> idempotent under replay; no out-of-order or repeated transition releases value."

## Rationalizations to reject

- *"The payment page redirected back with success."* → A client-delivered success is spoofable; confirm
  settlement server-side with the provider before setting paid.
- *"The charge was authorized."* → Authorization is not settlement; gate fulfillment on the confirmed settled
  state, not on a hold that can be released.
- *"Retries are just network noise."* → A non-idempotent handler turns any retry, benign or malicious, into a
  double charge or double shipment; make it idempotent per operation.
- *"Refunds go through our finance team."* → If the refund endpoint can be replayed or credits without reversing
  the capture, it is a payout primitive regardless of who normally calls it.
- *"The client tracks the order state."* → Client-asserted state lets an attacker set the value-releasing state;
  derive every transition from server-confirmed events.

## Executing this in practice

You need the legal states and transitions, which server-confirmed event gates each value-releasing action, and
whether each such handler is idempotent. On sandbox payment paths, try to mark paid without settlement, skip to
fulfillment, replay the charge and fulfillment and refund handlers, and repeat a refund. Reading the state-
transition code shows the intended order; a value release without settlement or a doubled handler shows whether
it holds.

## Related

- `auditing-payment-callback-and-amount-integrity` - the provider callback that should set the paid state; its
  authenticity and amount checks are the trusted signal this state machine must wait for.
- `hunting-price-and-coupon-manipulation` - manipulating the amount before this state machine runs is the paired
  attack; together they cover paying less and paying nothing.
- `auditing-webhook-authenticity-and-callback-trust` - settlement usually arrives as a webhook; that skill
  verifies the event this flow depends on is genuine.
- `hunting-broken-object-level-authorization` - driving another user's order through these transitions is an
  object-authorization failure on the order; the two meet at the order identifier.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the out-of-order or replayed transition, sink = the
  value released without settlement, evidence = the skippable step or non-idempotent handler.
