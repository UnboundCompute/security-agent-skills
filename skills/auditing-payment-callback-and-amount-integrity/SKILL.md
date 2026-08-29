---
name: auditing-payment-callback-and-amount-integrity
description: >-
  Audit payment provider callbacks and settlement notifications for the trust that lets an attacker forge or
  alter a payment result: a callback whose signature is not verified so a spoofed success is accepted, an amount
  or currency taken from the callback or client rather than reconciled against the order the server created, a
  success notification not bound to a specific order so it can be replayed onto another order, and a settled
  status trusted without confirming it out of band with the provider. Covers redirect returns, server-to-server
  webhooks, and status polls where a payment processor tells the application a charge succeeded. Use when the
  application learns a payment result from an external processor and that message gates fulfillment. The forged
  or altered payment notification is the source, the order marked paid and fulfilled is the sink, and the
  unverified signature, unreconciled amount, or unbound order reference is the bug.
license: MIT
---

# Auditing payment callback and amount integrity: the success message is attacker-reachable until you verify it

When a payment finishes, the application finds out from the processor through a redirect back to the site, a
server-to-server webhook, or a status poll, and it acts on that message by marking the order paid and shipping
the goods. If the message is trusted without verification, the attacker controls the one signal that releases
value. The failures are concrete. A callback whose signature is not verified lets an attacker post a spoofed
success with no real payment. An amount or currency read from the callback or the client, rather than
reconciled against the order the server itself created, lets an attacker pay one cent for a thousand-dollar
order or switch to a cheaper currency. A success notification not bound to a specific order can be captured
from a real small purchase and replayed onto a larger order. And a status trusted without an out-of-band
confirmation to the provider takes the attacker's word for settlement. The audit treats the payment result as
untrusted input until the server verifies its authenticity, its amount, and its binding to the order. You audit
this by forging and altering the callback and confirming each is refused.

## When to use

- The application learns a payment succeeded from a processor callback, webhook, or status poll.
- The amount, currency, or status may be read from the callback or client instead of reconciled server-side.
- A success notification may lack a verified signature or a binding to the specific order it settles.

## Scope check

Test payment callbacks only against systems you own or are authorized to assess, on non-production or sandbox
payment paths. Forging and replaying callbacks drives real fulfillment on a live system, so use sandbox
processor credentials and test orders and never mark a real order paid or trigger a real shipment. If you can't
name the authorization, stop.

## The loop

1. **Establish the trusted settlement signal first.** Name exactly what the server must verify before it treats
   a payment as settled: the callback's signature against the processor's key, the amount and currency against
   the order the server created, and the notification's binding to that specific order, ideally confirmed out of
   band with the provider. This is the false-positive killer: a flow that verifies the signature, reconciles the
   amount and currency server-side, binds the notification to the order, and confirms settlement with the
   provider is correct. Name the trusted signal, then test forgery.

2. **Verify the callback signature.** Confirm the application verifies the callback or webhook signature against
   the processor's key over the exact payload it acts on, and rejects an unsigned or wrongly signed message.
   Post a spoofed success without a valid signature and confirm it is refused. An unverified callback lets an
   attacker assert a payment that never happened.

3. **Reconcile the amount and currency server-side.** Confirm the paid amount and currency are compared against
   the order the server created, not taken from the callback or client. Submit a callback whose amount is lower,
   or whose currency is cheaper, than the order and confirm the mismatch is rejected. An amount trusted from the
   message lets an attacker settle a large order for a token payment.

4. **Check order binding and replay.** Confirm the notification is bound to the specific order it settles and
   cannot be replayed onto another. Capture a valid success for a small test order and replay it against a larger
   test order; confirm it is refused because the reference, amount, and a single-use event identifier are checked.
   An unbound success is a reusable proof of payment.

5. **Confirm settlement out of band.** Confirm the server treats settlement as final only after confirming with
   the provider (a verified webhook or an API query), not on a client-delivered redirect status alone. A status
   the client carries back is attacker-controllable; the authoritative confirmation must come from the provider
   directly.

6. **Confirm and record.** Confirm by getting an order marked paid through a forged callback, an altered amount
   or currency, or a replayed success, on sandbox payment paths and without moving real money. Kill the lead if
   the server verifies the signature, reconciles amount and currency against its own order, binds the
   notification to that order, and confirms settlement with the provider. Record the forged or altered
   notification, the order-marked-paid sink, and the unverified signature, unreconciled amount, or unbound order
   reference.

## Where callback trust leaks

- **An unverified signature accepts a spoof.** A callback or webhook whose signature is not checked lets an
  attacker post a success for a payment that never occurred.
- **An amount from the message underpays.** Reading the paid amount or currency from the callback or client,
  instead of reconciling against the server's order, settles a large order for a small or cheaper payment.
- **An unbound notification replays.** A success not tied to a specific order and a single-use event id can be
  captured from one purchase and reused on another.
- **A client-carried status is trusted.** Treating a redirect-delivered status as settlement takes the
  attacker's word; settlement must be confirmed with the provider directly.
- **The wrong payload is verified.** A signature checked over something other than the acted-on fields lets the
  amount or order reference be altered after verification.

## Worked example (a confirm and a kill)

> **Confirm.** After redirect from the payment page the application marks the order paid using the amount and
> status in the return parameters, without verifying a signature or reconciling against the order it created.
> Returning with the status set to success and the amount set to one cent for a high-value order marks it paid
> and fulfills it. **Confirmed** payment forgery via unverified callback and unreconciled amount, `critical`,
> remediation = verify the processor signature over the payload, reconcile the paid amount and currency against
> the server-created order, bind the notification to that order with a single-use event id, and confirm
> settlement with the provider out of band.
>
> **Kill.** The application verifies the webhook signature against the processor key over the acted-on fields,
> reconciles the amount and currency against the order it created, binds the notification to that specific order
> and rejects a replayed event id, and marks the order paid only after confirming settlement with the provider
> directly, ignoring any client-carried status. A forged, altered, or replayed callback is refused. **Killed**,
> `kill_reason` = "signature verified, amount and currency reconciled server-side, notification bound to the
> order with a single-use id, and settlement confirmed with the provider; no forged payment result marks an
> order paid."

## Rationalizations to reject

- *"The processor called us back."* → Anyone can post to the callback URL; verify the signature over the payload
  you act on before trusting the result.
- *"The callback includes the amount."* → An amount in the message is attacker-controllable; reconcile against
  the order your server created, in the currency it set.
- *"The redirect said success."* → A client-carried status is spoofable; confirm settlement with the provider
  directly, not from the browser's return.
- *"Each payment has a transaction id."* → Confirm the id is single-use and bound to this order; an unbound id
  from a real small payment can be replayed onto a larger order.
- *"We verify the signature."* → Confirm it covers the amount and order reference you act on; a signature over
  the wrong fields lets those be altered after the check.

## Executing this in practice

You need how the application receives the payment result, whether it verifies the signature and over which
fields, whether it reconciles amount and currency against its own order, how the notification binds to the
order, and whether settlement is confirmed with the provider. On sandbox paths, post an unsigned success, an
altered-amount callback, and a replayed notification. Reading the callback handler shows the intended
verification; a forged result that marks an order paid shows whether it holds.

## Related

- `auditing-payment-state-machine-and-idempotency` - the callback sets the paid state this state machine gates
  on; a forged callback here defeats the ordering there, so audit them together.
- `auditing-webhook-authenticity-and-callback-trust` - the general webhook-signature and replay discipline; this
  skill is its payment-specific, amount-reconciling case.
- `hunting-price-and-coupon-manipulation` - altering the order total before payment is the front-end companion to
  altering the payment result here; both end in underpayment.
- `auditing-jwt-verification-and-key-trust` - signed callbacks share the verify-the-signature-and-the-claims
  discipline; the same decode-without-verify mistake appears in both.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the forged or altered payment notification, sink = the
  order marked paid and fulfilled, evidence = the unverified signature, unreconciled amount, or unbound order
  reference.
