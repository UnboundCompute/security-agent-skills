---
name: auditing-mobile-deeplink-trust
description: >-
  Audit how a mobile app handles a deep link, app link, or custom-scheme URL, so an attacker-supplied URL
  cannot drive a sensitive action or reach a trusted WebView context. Covers a custom scheme any app can
  register and hijack, an app link whose domain association is unverified so the link is not exclusively
  the app's, a deep-link parameter that flows unvalidated into a sensitive action, an attacker-controlled
  URL loaded into a WebView, and a JavaScript bridge exposed to a WebView that can load untrusted content.
  Use when reviewing deep-link routing, URL handling, and WebView configuration, not the manifest export
  state of the component that receives the link (that is the component-exposure skill). The
  attacker-supplied URL is the source, a sensitive action or a trusted WebView bridge acting on it is the
  sink, and a link parameter trusted without validation is the bug.
license: MIT
---

# Auditing mobile deep-link trust: when an attacker's URL drives a trusted action

A deep link is attacker-reachable input: a custom scheme any app can claim, or a link another app or a web
page can fire, carrying parameters the app routes into an action. The bug is trusting that URL, letting a
parameter reach a sensitive operation, or handing an attacker-controlled URL to a WebView that exposes a
JavaScript bridge back into the app. This audit reads the deep-link routing, the URL handling, and the
WebView configuration and asks, per link, whether an attacker-supplied URL can drive something it should
not. It pairs with the component-exposure skill: that one owns whether the receiving component is reachable
across the app boundary, this one owns whether the URL it carries is trusted. Keep the seam clean so a
single flaw is reported once.

## When to use

- The app registers a custom scheme, an app link, or a universal link, and routes the URL to an action.
- A deep-link parameter flows into navigation, a WebView load, an authentication step, or a state change.
- You want to know whether an attacker-supplied URL can drive a sensitive action or reach a trusted WebView.

## Scope check

Audit only apps you own or are authorized to assess, and fire a deep link only at a device or emulator in
scope, a crafted link drives real app state and can complete real actions. Adjudicate on the routing and
the WebView config. If you can't name the authorization, stop.

## The loop

1. **Establish which links are attacker-controllable first.** Determine how each link is registered: a
   custom scheme (any app can also register it and intercept or forge it), an app or universal link with a
   verified domain association (only the app resolves it), or one whose association is unverified (not
   exclusive). The registration decides whether an attacker can send or hijack the link; settle it before
   judging the handler.

2. **Check scheme hijack and association.** Look for a custom scheme carrying sensitive data or actions,
   which a malicious app can register to intercept or forge, and for an app link whose domain association is
   missing or unverified, so the link is not exclusively the app's and can be claimed or spoofed.

3. **Check parameter flow into sensitive actions.** Follow the deep-link parameters into the app and look
   for one that reaches a sensitive sink without validation: a navigation target, an authentication or
   token parameter, an account or object identifier, a redirect, or a file path. A link parameter trusted
   as if it were internal is the core flaw.

4. **Check the URL into a WebView.** Look for a deep-link or app-supplied URL loaded into a WebView without
   an allowlist of trusted origins, letting an attacker point the trusted WebView at content they control.
   This is the WebView URL seam the component-exposure skill hands here.

5. **Check the JavaScript bridge.** Where a WebView exposes a native bridge (a JavaScript interface or a
   message handler) to page content, look for that WebView being able to load untrusted or attacker-chosen
   content, so a hostile page calls native methods through the bridge. An exposed bridge on a WebView that
   loads only pinned first-party content is far weaker than one reachable by an attacker URL.

6. **Confirm and record.** Confirm by showing an attacker-supplied URL drives the sensitive action or
   reaches the bridge. Kill the lead if the link is an app or universal link with a verified domain
   association so an attacker cannot forge it, if the parameter is validated or canonicalized before the
   sink, if the WebView loads only a pinned first-party origin with no attacker-controllable URL, if the
   bridge is exposed only to content the app fully controls, or if the routed action is not sensitive and
   changes no state. Record the link, its registration, the parameter path, and the action or bridge reached.

## Where deep-link trust leaks

- **A custom scheme is not exclusive.** Any app can register the same scheme and intercept or forge the
  link; a scheme carrying sensitive data or actions cannot be trusted as the app's alone.
- **An unverified app link is claimable.** Without a verified domain association the link is not
  exclusively the app's; verify the association before trusting the link's origin.
- **A link parameter is attacker input.** A target, token, identifier, or path from a deep link is
  externally supplied; trusting it as internal is the flaw, wherever it lands.
- **A WebView URL from a link is an open redirect into a trusted context.** Loading an attacker-chosen URL
  into the app's WebView hands the attacker the WebView's trust; allowlist the origin.
- **A bridge is only as safe as what the WebView loads.** A native bridge exposed to a WebView that can
  load untrusted content lets a hostile page call native code; the load allowlist is the control.

## Worked example (a confirm and a kill)

> **Confirm.** A WebView exposes a native bridge that reads app storage, and the URL it loads comes from a
> deep-link parameter with no origin allowlist; an attacker sends a link that points the WebView at their
> page, which calls the bridge and exfiltrates stored data. **Confirmed** an attacker-controlled URL into a
> bridge-exposing WebView, `critical`, remediation = allowlist the WebView origin to pinned first-party
> content and remove or gate the bridge.
>
> **Kill.** A deep link carries a next-screen parameter, but the router resolves it against a fixed
> allowlist of in-app destinations and rejects anything else, and no WebView or bridge is involved. An
> attacker's value is discarded. **Killed**, `kill_reason` = "the deep-link parameter is validated against a
> fixed in-app destination allowlist before use, so an attacker-supplied value cannot drive navigation."

## Rationalizations to reject

- *"It uses our custom scheme."* -> Any app can register that scheme and intercept or forge the link; a
  custom scheme is not proof the link came from a trusted source.
- *"It is an app link, so it is verified."* -> Is the domain association actually verified? An unverified
  association makes the link claimable, not exclusive.
- *"The parameter just picks a screen."* -> Does it reach navigation, auth, or a path without validation? An
  unvalidated link parameter is attacker input at the sink.
- *"The WebView is part of our app."* -> Does it load a URL an attacker can choose? A trusted WebView
  pointed at hostile content lends that content the app's trust.
- *"The bridge is only for our pages."* -> Can the WebView be made to load an attacker page? A bridge is
  only safe if the WebView cannot load untrusted content.

## Executing this in practice

You need the registered schemes and app or universal links and their domain associations, the deep-link
routing and where each parameter flows, the WebView configuration and the URLs it loads, and any native
bridge exposed to a WebView. For each link, decide whether an attacker-supplied URL can drive a sensitive
action or reach the bridge. Reading the registration tells you whether an attacker can send the link;
reading the handler and the WebView config tells you what the URL can drive.

## Related

- `auditing-android-component-exposure` - the component-reach seam: that skill owns whether the component
  receiving the link is exported and reachable, this one owns whether the URL it carries is trusted.
- `testing-client-side-dom-vulnerabilities` - the web-page analogue of the WebView content this skill loads;
  the same untrusted-URL-into-a-trusted-context shape in a browser.
- `hunting-mobile-secret-and-storage-exposure` - the sibling for data at rest that a bridge or a routed
  action might read; distinct from the URL-trust decision here.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the attacker-supplied URL, sink = a sensitive action
  or a trusted WebView bridge acting on it, evidence = the registration, the parameter path, and the sink reached.
