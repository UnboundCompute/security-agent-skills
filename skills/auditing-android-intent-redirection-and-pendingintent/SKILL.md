---
name: auditing-android-intent-redirection-and-pendingintent
description: >-
  Audit Android privilege and access leaks through intent redirection and mutable pending intents, where a
  privileged component receives an intent carrying a nested intent it then launches, or hands out a pending
  intent an untrusted app can fill in, so the untrusted caller reaches a protected component or acts with the
  privileged app's identity, because the component forwards an attacker-supplied intent without constraint or
  the pending intent is mutable and under-specified. Use when an Android app forwards intents extracted from
  input or shares pending intents with other apps. Covers nested-intent redirection to protected components,
  mutable pending intents, implicit pending-intent delivery, and forwarding that carries the app's
  permissions. The attacker-supplied nested or fillable intent is the source, the privileged component
  launching or sending it is the sink, and reaching a protected target with the app's identity is the bug.
license: MIT
---

# Auditing Android intent redirection and PendingIntent: when an app relays an attacker's intent

An Android app acts on intents, and two patterns let an untrusted caller borrow the app's privilege. In intent
redirection, a component receives an intent that carries another intent inside it and then launches that
nested intent; if the component is reachable by other apps and forwards the nested intent without constraint,
an untrusted caller supplies an intent aimed at a protected component the caller could not reach directly, and
the privileged app launches it with the app's own identity. In the pending-intent case, the app creates a
token that lets another app perform an operation as the app later; if that token is mutable and
under-specified, the other app fills in the missing parts, target, action, or data, and directs the operation
where it wants, still under the app's identity. In both, the app relays an attacker's intent and lends its
privilege to the destination. The bug is a privileged component launching or sending an intent an untrusted
caller controls. You audit these by finding every forwarded and shared intent and checking what the caller can
steer.

## When to use

- An Android app forwards an intent it extracts from a received intent's extras or data.
- The app creates pending intents shared with other apps or delivered implicitly.
- A reachable component launches or sends intents on behalf of its caller.

## Scope check

Audit intent redirection only on apps and devices you own or are authorized to assess, on test devices and
accounts, driving the component with a benign test app you control rather than acting against real data. A
confirmed redirection reaches protected components with the app's privilege, so keep every probe within scope.
If you can't name the authorization, stop.

## The loop

1. **Establish whether the caller can actually steer the launched or sent intent first.** For each component
   that forwards a nested intent or shares a pending intent, determine whether the target and contents are
   fixed by the app, or whether an untrusted caller can supply or fill in the intent that is ultimately
   launched or sent. This is the false-positive killer: a component that launches only a fully specified,
   app-fixed intent, or a pending intent that is immutable and complete, cannot be steered by an attacker.
   Name the attacker-controlled part before crafting an intent.

2. **Enumerate forwarded and shared intents.** List the components reachable by other apps that extract an
   intent from input and launch it, and the pending intents the app creates and shares or delivers implicitly.
   Note the identity and permissions each launch or send carries.

3. **Check nested-intent redirection.** For a component that launches a nested intent taken from its input,
   determine whether it constrains the target, so it can only reach intended components, or forwards the
   attacker-supplied intent freely, letting the caller aim it at a protected component the app can reach but
   the caller cannot.

4. **Check pending-intent mutability and specificity.** For each shared pending intent, determine whether it
   is immutable and fully specifies its target, action, and data, or whether it is mutable or under-specified
   so the receiving app fills in the blanks and redirects the operation, and whether it is delivered
   implicitly where any app can intercept it.

5. **Check the identity the redirect carries.** Determine what privilege the launched or sent intent runs
   with, the app's own permissions, access to its protected components, or its granted data access, since the
   redirection matters only when it lends the caller something they lacked.

6. **Confirm and record.** Confirm by sending a benign test app's intent that redirects through the component
   to a protected target, or filling in a mutable pending intent to redirect it, and observing the app's
   privilege reach the target on a test device. Kill the lead if forwarded intents are constrained to intended
   targets, if shared pending intents are immutable and fully specified and not delivered implicitly, or if
   the redirect carries no privilege the caller lacked. Record the component, the attacker-controlled intent,
   the protected target, and the access observed, or set a `kill_reason`.

## Where intent redirection leaks

- **The steerable intent is the finding.** An app forwarding or sharing intents is normal; the bug is an
  untrusted caller steering where a privileged launch or send goes. Name the attacker-controlled part.
- **Unconstrained nested intents reach protected components.** A component that launches an attacker-supplied
  nested intent without limiting the target lets the caller reach components the app can but they cannot.
- **Mutable pending intents are filled in by the receiver.** A mutable or under-specified pending intent lets
  the receiving app set the target, action, or data, redirecting the operation under the app's identity.
- **Implicit pending-intent delivery invites interception.** A pending intent delivered implicitly can be
  received by any app that matches, handing an untrusted app a token that acts as the creator.
- **The borrowed identity is the impact.** Redirection matters because the launch or send runs with the app's
  permissions and access, so the caller reaches what the app can reach, not what they can.

## Worked example (a confirm and a kill)

> **Confirm.** An exported component extracts an intent from its extras and launches it without constraining
> the target. A benign test app sends an intent whose nested intent targets a protected, non-exported
> component of the same app, and the privileged app launches it, reaching the protected component with the
> app's identity on a test device. **Confirmed** intent redirection reaching a protected component with the
> app's privilege, `high`, remediation = do not launch intents extracted from untrusted input, or constrain
> the forwarded target to an explicit intended component, and make shared pending intents immutable and fully
> specified.
>
> **Kill.** The same component launches only an explicit, app-fixed intent and never forwards an
> attacker-supplied nested intent, and every shared pending intent is immutable, fully specifies its target,
> action, and data, and is delivered explicitly to a known package. An untrusted caller can steer neither the
> nested intent nor a pending intent. **Killed**, `kill_reason` = "forwarded intents are fixed to explicit
> intended targets and shared pending intents are immutable and fully specified; no untrusted caller steers a
> privileged launch or send."

## Rationalizations to reject

- *"The component only forwards intents."* -> Forwarding an attacker-supplied intent is the redirection; if
  the target is not constrained, the caller aims it at a protected component through the app.
- *"The pending intent is ours."* -> A mutable or under-specified pending intent shared with another app lets
  that app fill in the target and act as you; make it immutable and fully specified.
- *"Other apps cannot reach that component."* -> If the component is exported or otherwise reachable, another
  app reaches it; confirm its reachability rather than assuming it is private.
- *"It is delivered to a broadcast."* -> An implicit pending intent or broadcast can be intercepted by any
  matching app; deliver explicitly to a known package.
- *"The launched action is harmless."* -> The action runs with the app's identity and access; a harmless-
  looking launch can reach protected components or granted data the caller lacked.

## Executing this in practice

You need every component reachable by other apps that forwards a nested intent, every pending intent the app
shares or delivers implicitly, and for each the identity and permissions the launch or send carries, plus
whether the target and contents are app-fixed or caller-steerable. For each, decide whether an untrusted
caller can direct a privileged launch or send. Reading the forwarding and pending-intent construction settles
most leads; driving the component from a benign test app to reach a protected target on a test device settles
the rest.

## Related

- `auditing-android-component-exposure` - the exported-component reachability this redirection depends on is
  the surface that skill audits, so they meet on which components an untrusted app can reach.
- `auditing-mobile-deeplink-trust` - a deep link that carries a redirecting intent is one delivery path for
  the attacker-supplied intent this skill forwards.
- `auditing-ios-app-group-and-pasteboard-exposure` - the iOS side of cross-app data reach, a companion to the
  Android inter-app privilege reach here.
- `auditing-mobile-webview-bridge-exposure` - a redirected intent that drives a WebView to load attacker
  content chains into the bridge exposure that skill audits.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the attacker-supplied nested or fillable intent,
  sink = the privileged component launching or sending it, evidence = reaching a protected target with the
  app's identity from a benign test app on a test device.
