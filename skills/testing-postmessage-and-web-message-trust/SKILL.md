---
name: testing-postmessage-and-web-message-trust
description: >-
  Test cross-document messaging trust, where a browser message handler acts on data whose origin or content
  an attacker can influence by opening or framing the window, or where code sends sensitive data to another
  window with a wildcard target. Use when reviewing client code that registers a message event listener and
  routes the data into the DOM, an evaluation, navigation, or storage, or that posts secrets across
  windows. Covers a missing, wildcard, or substring origin check, a missing source-window check, a wildcard
  target that leaks data, and a deserialized message driving a sink. The message event is the source, the
  handler sink or the outbound post is the sink, and acting without an exact origin and source check, or
  leaking to a wildcard target, is the bug.
license: MIT
---

# Testing postMessage and web message trust: when any window can drive the handler

Cross-document messaging lets one browsing context send data to another, across origins, and the receiving
page decides what to do with it. That decision is a trust boundary: the message arrives with an origin the
browser stamps on it, but nothing forces the handler to check that origin, or to confirm the message came
from the window it expected. A handler that reads the data straight into the page markup, an evaluation, a
navigation, or storage, without an exact origin and source check, can be driven by any site that opens or
frames the window. The mirror image is a page that posts sensitive data to another window with a wildcard
target, spraying it to whatever origin happens to be there. The bug is trusting a message because it
arrived, rather than because it came from a known origin and source. You find these by reading every
message listener and every outbound post and asking who is allowed to send or receive.

## When to use

- Client code registers a cross-document message event listener and acts on the message data.
- A page embeds or is embedded by other origins, or opens child windows it exchanges messages with.
- Code sends data to another window and you need to confirm the target origin is not a wildcard.

## Scope check

Test cross-document messaging only against applications you own or are authorized to assess, from a test
origin and test accounts, using benign markers to show a handler acts on an unexpected message or that a
secret reaches a wildcard target. A confirmed case runs in a real user's session, so coordinate and stay in
scope. If you can't name the authorization, stop.

## The loop

1. **Establish whether the handler checks the origin and source first.** For each message listener, read
   whether it validates the event origin by exact match against an expected value, and, where relevant,
   confirms the source window is the one it expected, before using the data. This is the false-positive
   killer: a handler that acts only on an exact-origin, expected-source message cannot be driven by an
   arbitrary site, while one with a missing, wildcard, or substring origin check can. Name the check before
   judging the sink.

2. **Map every listener and every outbound post.** Inventory each registered message handler and what it does
   with the data, and each call that sends a message to another window along with the target origin it uses.
   These are the sources and sinks the rest of the loop examines.

3. **Judge the origin check.** A missing check accepts every origin; a substring or prefix check accepts a
   lookalike origin that contains the expected string; a check that compares against the wrong property, or
   only logs a mismatch without stopping, is no check. Only exact equality against a fixed expected origin is
   a real gate. Decide which the handler uses.

4. **Follow the data into the sink.** Determine whether the message data reaches a dangerous sink: written
   into the page markup or an element that executes it, passed to an evaluation, used as a navigation target,
   or stored where it is later trusted. A handler that reads only inert fields into non-executing state is
   low impact even without a perfect origin check; a handler that reaches an executing sink is the finding.

5. **Check the outbound posts and the source window.** For each send, confirm the target origin is an exact
   value and not a wildcard, because a wildcard delivers the data to whatever origin currently occupies the
   target window. For receivers, confirm the source-window check where the protocol expects a specific
   partner, so a third frame cannot impersonate it.

6. **Confirm and record.** Confirm by posting a benign marker from an unexpected test origin and observing the
   handler reach its sink, or by showing a secret delivered to a wildcard target, on an isolated instance.
   Kill the lead if the handler checks the origin by exact match and the source where relevant and only then
   uses the data, if the data reaches no executing or trusted sink, or if every outbound post uses an exact
   target origin. Record the handler or post, the trusted value, the sink, and the impact, or set a
   `kill_reason`.

## Where web message trust leaks

- **The origin is checked or it is not.** A handler without an exact-origin gate acts for every site that can
  reach the window; the browser supplies the origin, but only the handler can enforce it.
- **Substring checks accept lookalikes.** An origin test that uses contains or ends-with passes an attacker
  origin that embeds the expected string; only exact equality holds.
- **The sink decides the severity.** A handler that writes the data into executing markup or an evaluation is
  a DOM scripting sink; one that reads an inert field into non-executing state is not, even with a weak check.
- **Wildcard targets leak outbound.** Posting sensitive data with a wildcard target origin delivers it to
  whatever origin holds the target window, so secrets require an exact target.
- **Source matters when the partner is fixed.** When a protocol expects one specific window, skipping the
  source-window check lets a third frame that can post to the page impersonate the expected partner.

## Worked example (a confirm and a kill)

> **Confirm.** A page registers a message handler that writes a field from the message data into an element
> that renders it as markup, with no origin check. A page on an attacker origin frames the target and posts a
> message whose field contains markup, and it executes in the victim's session on an isolated instance.
> **Confirmed** cross-document message handling to DOM scripting, `high`, remediation = validate the event
> origin by exact match against the expected origin (and the source window where the partner is fixed) before
> using the data, and write the data through a safe, non-executing API rather than as markup.
>
> **Kill.** The same handler compares the event origin by exact equality against a single expected origin,
> confirms the source is the expected child window, and writes the data through a text API that does not
> execute markup, and every outbound post uses an exact target origin rather than a wildcard. A message from a
> test origin is ignored and never reaches the sink. **Killed**, `kill_reason` = "handler acts only on an
> exact-origin, expected-source message and writes through a non-executing API, and outbound posts use exact
> targets; an arbitrary origin cannot drive the sink or receive the data."

## Rationalizations to reject

- *"The browser tells us the origin."* -> It does, but only the handler enforces it; without an exact-match
  gate the handler acts on every origin the browser reports.
- *"We check the origin contains our domain."* -> A contains or ends-with check passes a lookalike origin that
  embeds your domain; only exact equality is a check.
- *"It is just data we read."* -> If that data is written as markup, evaluated, or used as a navigation
  target, it is a sink; trace where it goes before calling it inert.
- *"We post with a wildcard for convenience."* -> A wildcard target delivers the message to whatever origin
  holds the window; if the data is sensitive, the target origin must be exact.
- *"Only our own frame talks to us."* -> Any origin that can open or frame the window can post to it; confirm
  the origin and, where the partner is fixed, the source window, rather than assuming the sender.

## Executing this in practice

You need every registered message listener with its origin and source checks and the sink each feeds, and
every outbound post with the target origin it uses. For each listener, decide whether an arbitrary origin can
drive a dangerous sink; for each post, whether a secret can reach a wildcard target. Reading the handler and
its checks settles most leads; posting a benign marker from an unexpected test origin, or observing a secret
delivered to a wildcard, on an isolated instance settles the rest.

## Related

- `testing-client-side-dom-vulnerabilities` - the message data reaching an executing sink is a DOM sink that
  skill covers in depth; this one focuses on the cross-window trust boundary that feeds it.
- `auditing-cors-and-cross-origin-trust` - the server-side mirror of the same origin-trust question, and it
  also treats cross-window message handlers, so the two audits share the boundary.
- `reviewing-content-security-policy` - a strong policy limits what a driven handler can execute, so a weak
  policy and a missing origin check compound.
- `hunting-reflected-and-stored-xss` - a message handler that writes data as markup is another route to the
  script execution that skill hunts through server output.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the message event, sink = the handler sink or the
  outbound post, evidence = the handler reaching an executing sink from an unexpected origin, or a secret
  delivered to a wildcard target, on an isolated instance.
