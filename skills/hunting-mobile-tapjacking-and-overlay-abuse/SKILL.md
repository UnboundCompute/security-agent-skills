---
name: hunting-mobile-tapjacking-and-overlay-abuse
description: >-
  Hunt mobile interface redressing, where another app draws over or intercepts a sensitive screen so the user
  acts on the attacker's terms without knowing it, approving a permission, confirming a transaction, or
  entering a secret while an overlay hides or fakes what they are really touching, because the sensitive
  screen does not detect that it is obscured, does not filter touches passed through an overlay, or does not
  bind the confirmation to what the user actually saw. Use when a mobile app presents consent, confirmation,
  or credential-entry screens that another app could overlay or intercept. Covers tap hijacking through
  overlays, obscured-touch acceptance, accessibility-driven interaction, and confirmations not bound to the
  displayed action. The overlaying or intercepting app is the source, the sensitive screen accepting the
  redirected interaction is the sink, and the user approving what they did not intend is the bug.
license: MIT
---

# Hunting mobile tapjacking and overlay abuse: when a tap lands on a screen the user cannot see

A mobile user acts by touching what they see, and that trust breaks when another app can draw over the screen
or intercept the interaction. An overlay app puts its own window on top of a sensitive screen, a permission
grant, a payment confirmation, a credential prompt, and either hides what the user is really touching or fakes
a benign interface over it, so the tap the user believes is dismissing a game lands on the real approve
button underneath. Related abuses intercept rather than draw: an accessibility-style service drives the
interaction programmatically. The sensitive screen is complicit when it accepts a touch it cannot confirm the
user saw: it does not detect that it is obscured, does not filter touches that passed through an overlay, or
confirms an action without binding the confirmation to what was actually displayed. The bug is a consequential
approval taken while the screen cannot vouch for what the user saw or touched. You hunt these by finding every
sensitive screen and checking whether it defends against being overlaid or driven.

## When to use

- A mobile app presents consent, confirmation, or credential-entry screens another app could overlay.
- A sensitive action is approved by a tap that an overlay could hide, fake, or pass through.
- An accessibility-style service or intercepting app could drive the interaction programmatically.

## Scope check

Hunt overlay and tapjacking abuse only on apps and devices you own or are authorized to assess, on test
devices and accounts, using a benign test overlay you control against a test screen rather than deceiving a
real user. A confirmed case makes a user approve an unintended sensitive action, so keep every probe within
scope. If you can't name the authorization, stop.

## The loop

1. **Establish whether the sensitive screen can vouch for what the user saw and touched first.** For each
   consequential screen, determine whether it detects when it is obscured and refuses or filters the touch,
   and binds its confirmation to the action actually displayed, or whether it accepts a tap without knowing
   whether an overlay hid or faked the interface. This is the false-positive killer: a screen that rejects
   obscured touches and confirms only the displayed action cannot be redressed by an overlay. Name the
   screen's defense before crafting an overlay.

2. **Enumerate the consequential screens.** List the screens where a single interaction has real
   consequence: granting a permission, confirming a payment or a transfer, entering a credential or a
   one-time value, or approving an irreversible action. These are the targets an overlay would redress.

3. **Check obscured-touch handling.** Determine whether each screen detects that another window is drawn over
   it and either refuses the touch or ignores touches that passed through an overlay, or whether it accepts
   the interaction regardless, so an overlay can hide or fake the interface while the real control receives
   the tap.

4. **Check the confirmation binding.** Determine whether the approval is bound to what was displayed, so a
   user confirming sees and confirms the actual action, or whether the confirmation is a generic tap that an
   overlay can steer to a different underlying action than the one the user believes they are approving.

5. **Check programmatic interception.** Determine whether an accessibility-style service or another mechanism
   could drive the interaction without a genuine user touch, approving the sensitive action programmatically,
   and whether the screen distinguishes a real user interaction from a synthesized one where it matters.

6. **Confirm and record.** Confirm by placing a benign test overlay over a test screen, or driving it
   programmatically, and showing a sensitive action is approved while the user's apparent interaction was with
   the overlay, on a test device. Kill the lead if the screen rejects obscured touches, binds confirmation to
   the displayed action, and resists programmatic approval, or if the action carries no consequence worth
   redressing. Record the screen, the overlay or interception, the approved action, and the redress observed,
   or set a `kill_reason`.

## Where overlay abuse leaks

- **The screen's ability to vouch is the finding.** A sensitive approval is safe only if the screen knows the
  user saw and touched the real control; the bug is accepting a tap it cannot vouch for. Name the missing
  defense.
- **Obscured touches redress the action.** A screen that accepts a touch while obscured lets an overlay hide
  the real control and fake a benign one, so the user taps approve believing they tapped something else.
- **Unbound confirmations approve the wrong thing.** A generic confirming tap not bound to the displayed
  action can be steered by an overlay to approve an action the user never saw.
- **Programmatic interaction skips the user.** An accessibility-style service can approve a sensitive action
  without a genuine touch, so a screen that cannot tell a real interaction from a synthesized one is driven.
- **Consequence is what makes it matter.** Redressing a screen with no real consequence gains nothing; the
  targets are permissions, payments, credentials, and irreversible actions.

## Worked example (a confirm and a kill)

> **Confirm.** A payment confirmation screen approves the transfer on a tap and does not detect when it is
> obscured. A benign test overlay draws a game-like interface over it so the user's tap on a decoy button
> lands on the real approve control, confirming a test transfer the user did not intend, on a test device.
> **Confirmed** tapjacking of a consequential confirmation, `high`, remediation = refuse or filter touches
> received while the screen is obscured by another window, bind the confirmation to the displayed transaction
> details, and require a genuine user interaction for consequential approvals.
>
> **Kill.** The same screen detects that another window overlays it and refuses the touch, ignores touches
> that passed through an overlay, binds the approval to the displayed transaction, and requires a genuine
> interaction that a programmatic service cannot synthesize. An overlay hides nothing that still approves, and
> a driven tap is rejected. **Killed**, `kill_reason` = "the screen refuses obscured and passed-through
> touches, confirms only the displayed action, and rejects synthesized interaction; no overlay or interception
> approves an unintended action."

## Rationalizations to reject

- *"The user still has to tap approve."* -> An overlay can hide the approve button under a decoy so the user
  taps it believing it is something else; the tap is real, the user's understanding is not.
- *"Overlays require a special permission."* -> Users grant overlay capability routinely, and some contexts
  allow it without a prompt; assume an overlay can be present and defend the screen rather than the platform.
- *"Our screen looks distinctive."* -> An overlay can fake or hide the whole screen; appearance does not
  protect a screen that accepts obscured touches.
- *"Accessibility services are trusted."* -> A malicious or abused accessibility-style service can drive the
  interaction; a consequential approval should require a genuine user touch it can distinguish.
- *"It is only a confirmation dialog."* -> A confirmation that approves a permission, payment, or irreversible
  action is exactly the consequential target; bind it to what was displayed.

## Executing this in practice

You need every consequential screen, whether each detects and refuses obscured or passed-through touches,
whether its confirmation is bound to the displayed action, and whether it resists programmatic approval. For
each, decide whether an overlay or an intercepting service could make the user approve an unintended action.
Reading the screen's obscured-touch and confirmation handling settles most leads; placing a benign test
overlay or driving the screen programmatically on a test device settles the rest.

## Related

- `auditing-clickjacking-and-ui-redressing` - the web counterpart, where a framed page is redressed by an
  overlay, sharing the obscured-interaction reasoning with this mobile case.
- `auditing-android-component-exposure` - a screen reachable by another app is more easily brought forward for
  redressing, the reachability that skill audits.
- `auditing-mobile-biometric-and-local-auth-bypass` - a consequential approval gated by local auth relies on
  the genuine-interaction guarantee this skill checks against overlays and interception.
- `auditing-android-intent-redirection-and-pendingintent` - an intent that brings a sensitive screen forward
  for overlay is delivered through the redirection that skill audits.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the overlaying or intercepting app, sink = the
  sensitive screen accepting the redirected interaction, evidence = a sensitive action approved while the
  user's apparent interaction was with a benign test overlay on a test device.
