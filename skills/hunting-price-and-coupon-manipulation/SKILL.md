---
name: hunting-price-and-coupon-manipulation
description: >-
  Hunt for ways a buyer can control the price the server charges: a price, quantity, or line total taken from
  the client request instead of recomputed server-side from a trusted catalog, a negative or overflowing
  quantity that drives the total down or wraps it, a discount or coupon that stacks, reuses past its limit, or
  applies to items it should not, and a total computed on the client and trusted at checkout. Covers e-commerce
  carts, checkout totals, and promotion engines where the amount charged is derived from item prices, quantities,
  and discounts. Use when a buyer influences cart contents or promotions and the charged total is the boundary.
  The client-supplied price, quantity, or coupon is the source, the discounted or negative charged total is the
  sink, and the client-trusted amount or unenforced coupon rule that produces it is the bug.
license: MIT
---

# Hunting price and coupon manipulation: whoever computes the total decides what you pay

The amount a customer is charged is computed from item prices, quantities, and discounts, and the only safe
place to do that computation is the server, from data the buyer cannot change. When any input to the total is
taken from the client, the buyer sets the price. The failures are direct. A price, quantity, or line total sent
in the request and trusted, rather than recomputed from a server-side catalog, lets a buyer name their own
price. A negative quantity subtracts from the total, and a quantity large enough to overflow the arithmetic
wraps it to a small or negative number. A discount or coupon that stacks with itself or others, is reused past
its intended limit, or applies to items it was never meant for, cuts the total below its floor. And a total
computed in the browser and accepted at checkout hands the whole calculation to the attacker. The hunt is to
find any input to the charged total that the buyer controls. You hunt this by altering prices, quantities, and
coupons in the request and seeing what the server charges.

## When to use

- A buyer influences cart contents, quantities, or promotions and the server computes a charged total.
- Prices, quantities, or line totals may be taken from the client request rather than a trusted catalog.
- Coupons or discounts may stack, be reused past their limit, or apply to items outside their scope.

## Scope check

Test pricing and promotions only against applications you own or are authorized to assess, on non-production or
sandbox checkout paths. Manipulating totals and placing orders exercises real commerce, so use test accounts and
sandbox payment and never complete a real underpriced purchase or consume real promotions. If you can't name the
authorization, stop.

## The loop

1. **Establish the server-side price of truth first.** Confirm the total should be recomputed on the server from
   a trusted catalog of prices, with server-validated quantities and server-enforced coupon rules, and that no
   price, line total, or discount from the client is trusted. This is the false-positive killer: a checkout that
   recomputes every amount server-side from the catalog, bounds quantities, and enforces coupon scope, stacking,
   and reuse limits is correct. Name the price of truth, then test each input.

2. **Test client-supplied prices and line totals.** Alter the unit price, line total, or item price in the cart
   or checkout request and confirm the server ignores it and recomputes from the catalog. A checkout that
   charges the price sent in the request lets the buyer set any amount, the most direct form of the bug.

3. **Test quantity manipulation.** Submit a negative quantity, a zero, and a very large quantity, and confirm the
   server validates the range and recomputes the total safely. A negative quantity that subtracts from the total,
   or a large one that overflows and wraps the arithmetic, drives the charge down or negative.

4. **Test coupon scope, stacking, and reuse.** Apply a coupon to items it should not cover, stack it with itself
   or with other discounts, and redeem it more times than allowed. Confirm the server enforces the coupon's item
   scope, its stacking rules, its per-account and global reuse limits, and its validity window. A coupon that
   stacks or reuses past its limit cuts the total below its intended floor.

5. **Check where the total is computed and trusted.** Determine whether the final charged total is computed
   server-side at checkout or taken from a client-supplied total. Confirm the amount charged is the server's
   recomputation, and that a total or discount posted from the browser is discarded. A client-computed total
   accepted at checkout is a full bypass of pricing.

6. **Confirm and record.** Confirm by completing a sandbox checkout at a manipulated total, a client-set price, a
   negative or overflowing quantity, an out-of-scope or stacked coupon, or a client-supplied total, without
   completing a real underpriced order. Kill the lead if the server recomputes every amount from the trusted
   catalog, bounds quantities, and enforces coupon scope, stacking, and reuse. Record the client-supplied price,
   quantity, or coupon, the discounted or negative charged total, and the client-trusted amount or unenforced
   coupon rule.

## Where pricing trust leaks

- **A client-supplied price is charged.** Trusting a unit price, line total, or item price from the request lets
  the buyer name their own price; the server must recompute from the catalog.
- **A negative or overflowing quantity lowers the total.** A quantity that goes negative subtracts from the
  charge, and one large enough to overflow wraps the arithmetic to a small or negative amount.
- **A coupon exceeds its scope or limit.** A discount that applies to ineligible items, stacks against its rules,
  or is reused past its limit cuts the total below its floor.
- **A client-computed total is trusted.** Accepting a total or discount calculated in the browser hands the
  entire pricing computation to the attacker.
- **Currency or rounding is client-influenced.** Letting the client pick a currency or rounding that lowers the
  effective charge is a quieter form of setting the price.

## Worked example (a confirm and a kill)

> **Confirm.** The checkout request includes each line item's unit price, and the server sums the submitted
> prices to compute the charge. Editing the unit price of a high-value item to a token amount before submitting
> results in an order charged at that amount, because the server trusts the client-supplied price rather than
> recomputing from the catalog. **Confirmed** price manipulation via client-supplied unit price, `high`,
> remediation = recompute every line total and the order total server-side from the trusted catalog, ignore any
> price or total in the request, and validate quantities.
>
> **Kill.** The server recomputes each line total and the order total from the catalog price for the item id,
> ignoring any price, line total, or total in the request; it bounds quantities to a valid positive range with
> overflow-safe arithmetic; and it enforces each coupon's item scope, stacking rules, per-account and global
> reuse limits, and validity window. An altered price, a negative or huge quantity, and a stacked or out-of-scope
> coupon all fail to lower the charge. **Killed**, `kill_reason` = "all amounts recomputed server-side from the
> trusted catalog with bounded quantities and enforced coupon rules; no client-supplied price, quantity, or
> coupon changes the charged total."

## Rationalizations to reject

- *"The cart already has the prices."* → Prices in the cart request are client-controllable; recompute the total
  from the server catalog by item id, never from the submitted prices.
- *"Quantities are always positive."* → Confirm the server rejects negative and zero and bounds large quantities;
  a negative or overflowing quantity is a classic total-lowering primitive.
- *"Coupons are validated when issued."* → Validate at redemption too: item scope, stacking, per-account and
  global reuse, and expiry; an issued coupon can still be stacked or reused past its limit.
- *"The frontend calculates the total."* → A client-computed total is attacker-controllable; the server must
  recompute the charged amount and discard any client total.
- *"It is only a small discount."* → Any client-controlled input to the price is the bug regardless of size; the
  same path that shaves a little can zero the total.

## Executing this in practice

You need every input to the charged total, item prices, quantities, line totals, coupons, and the final total,
and whether each is recomputed server-side from a trusted catalog or taken from the request. On sandbox checkout
paths, alter each price and line total, submit negative and overflowing quantities, and stack, misscope, and
reuse coupons. Reading the pricing code shows where the total comes from; a sandbox order at a manipulated total
shows whether the server owns the computation.

## Related

- `auditing-payment-state-machine-and-idempotency` - manipulating the total here and driving the payment state
  there are the two halves of paying less or nothing; audit them together.
- `auditing-payment-callback-and-amount-integrity` - the callback reconciles the paid amount against the order;
  this skill is where that order total is set, so a manipulated total undermines that reconciliation.
- `hunting-broken-object-level-authorization` - applying another account's coupon or cart is an
  object-authorization failure on the promotion or cart; the two meet at the identifier.
- `hunting-mass-assignment-and-property-authz` - a client that can set a price or discount field it should not
  is a mass-assignment of the pricing inputs; the same over-trust of request fields underlies both.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the client-supplied price, quantity, or coupon, sink =
  the discounted or negative charged total, evidence = the client-trusted amount or unenforced coupon rule.
