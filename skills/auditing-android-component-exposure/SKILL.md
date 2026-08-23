---
name: auditing-android-component-exposure
description: >-
  Audit an Android app for components another app on the device can reach and drive, after the manifest
  export flags and permission gates are resolved. Covers an activity, service, broadcast receiver, or
  content provider exported without a permission gate, an intent filter that makes a component implicitly
  exported, a provider that grants URI access or exposes files across the app boundary, a permission
  declared with a weak protection level, and a component that trusts intent extras from an untrusted
  caller for a sensitive action. Use when reviewing the manifest and the component code that handles an
  inbound intent, not the deep-link URL trust that a WebView loads (that is the deep-link skill). A
  cross-app caller is the source, a reachable exported component acting on the intent is the sink, and a
  sensitive component another app can invoke unguarded is the bug.
license: MIT
---

# Auditing Android component exposure: what another app on the device can reach and drive

An Android component is only an attack surface if another app can reach it, so this audit is a resolution
of the export state and the permission gate, not a scan of component names. A component is exported when a
flag says so or an intent filter makes it so by default, and it is exposed only when no permission of a
strong enough protection level gates the caller. The bug is a sensitive component another installed app
can invoke, and then trust the caller's intent extras to do something it should not. You audit it by
resolving, per component, whether it is reachable across the app boundary and what it does with an
attacker's intent. Stay on component reachability and inbound-intent handling; when the exposure is a URL
the component hands to a WebView, that is the deep-link skill's seam, cross-referenced below.

## When to use

- You are reviewing an Android manifest and the activities, services, receivers, or providers it declares.
- A component sets an export flag or an intent filter, or handles intent extras for a sensitive action.
- You want to know which components another app on the device can actually reach and drive.

## Scope check

Audit only apps you own or are authorized to assess, and exercise a component only on a device or
emulator in scope, invoking an exported component drives real app state. Adjudicate on the manifest and
the handler. If you can't name the authorization, stop.

## The loop

1. **Resolve the export state and permission gate first.** For each component, determine whether it is
   exported: an explicit export flag, or an intent filter that makes it exported by default on the relevant
   platform version. Then find the permission that gates it, if any, and its protection level. A component
   is only an attack surface if it is both exported and either ungated or gated by a weak permission;
   settle both before flagging.

2. **Check activities and services.** Look for an exported activity or service with no permission gate that
   performs a sensitive action, changes state, or returns data on behalf of the caller, reachable by any
   installed app through an explicit or implicit intent.

3. **Check broadcast receivers.** Look for an exported receiver with no gate that acts on a broadcast an
   untrusted app can send, or that trusts the contents of a received broadcast for a sensitive branch. A
   receiver registered for an implicit action is reachable by any sender.

4. **Check content providers and file exposure.** Look for an exported provider with no read or write
   permission, a provider that grants URI permissions too broadly, or a file-sharing provider whose path
   configuration exposes files outside the intended directory across the app boundary.

5. **Check the permission protection level and intent trust.** Look for a custom permission declared at a
   weak protection level that any app can hold, defeating the gate it is meant to enforce, and for a
   component that trusts intent extras (a target, a redirect, a file path, a flag) from an untrusted caller
   for a sensitive action without validating them.

6. **Confirm and record.** Confirm by resolving that an unprivileged installed app can reach the component
   and drive the sensitive action. Kill the lead if the component is not actually exported (not flagged and
   no implicit filter, or explicitly not exported), if a signature-level or otherwise strong permission
   gates it so only same-signer or privileged apps qualify, if the component is exported but performs
   nothing sensitive and trusts no caller input, if the provider grants access only within its own app
   sandbox, or if the exposure is purely a URL the component forwards to a WebView (hand that to the
   deep-link skill to avoid double-reporting). Record the component, its resolved export and gate, and the
   action a caller can drive.

## Where component exposure leaks

- **Exported plus ungated is the surface, not the component.** A component is only reachable cross-app when
  it is exported and no strong permission gates it; resolve both before calling it exposed.
- **An intent filter can export by default.** A component with an intent filter is exported without an
  explicit flag on some platform versions; do not rely on the flag alone.
- **A weak custom permission is no gate.** A permission any app can be granted does not restrict the
  caller; the protection level decides whether the gate holds.
- **Trusting intent extras turns reach into action.** An exported component that acts on a caller's target,
  path, or flag without validation lets the caller redirect it; reachability plus trust is the bug.
- **The WebView URL seam belongs to the deep-link skill.** When the exposure is a URL the component loads
  into a WebView, adjudicate it there, not here, so one bug is not reported twice.

## Worked example (a confirm and a kill)

> **Confirm.** An exported activity with no permission gate accepts an intent extra naming a file path and
> returns that file's contents to the caller; any installed app can send the intent and read files from the
> app's private storage. **Confirmed** an unguarded exported activity that trusts a caller-supplied path,
> `high`, remediation = set the component not exported or gate it with a signature-level permission, and
> validate the path against an allowlist.
>
> **Kill.** A service looks sensitive but is declared not exported, and no intent filter makes it exported;
> only the app's own components can bind to it. No other app can reach it. **Killed**, `kill_reason` = "the
> component is not exported and has no implicit intent filter, so no cross-app caller can reach it."

## Rationalizations to reject

- *"The component looks internal."* -> Is it exported by flag or by an intent filter? An implicit filter can
  export it by default; resolve the export state, not the intent behind the name.
- *"It is protected by a permission."* -> At what protection level? A normal or weak custom permission any
  app can hold is not a gate; only a strong level restricts the caller.
- *"It is exported but harmless."* -> Does it act on the caller's intent extras for anything sensitive? An
  exported component that trusts caller input is drivable even if it looks passive.
- *"The provider is for our own files."* -> Does its path configuration or URI grant cross the app boundary?
  A misconfigured provider exposes files beyond the sandbox.
- *"It just opens a URL."* -> Then the trust decision is on that URL in the WebView; adjudicate it in the
  deep-link skill so the same bug is not counted twice.

## Executing this in practice

You need the manifest, each component's export flag and intent filters, the permission that gates it and
its protection level, the platform version defaults, and the handler code for each inbound intent. For each
component, resolve whether a cross-app caller can reach it and what action it can drive. Reading the
manifest tells you what is exported; reading the handler tells you what the caller can make it do.

## Related

- `auditing-mobile-deeplink-trust` - the WebView-and-URL seam: when a component forwards a URL into a
  WebView or exposes a JavaScript bridge, that skill owns the URL trust and this one owns the component reach.
- `hunting-mobile-secret-and-storage-exposure` - the sibling for secrets and data at rest in the mobile app,
  distinct from the cross-app component reach this skill audits.
- `hunting-broken-object-level-authorization` - the server-side authorization analogue; a mobile component
  that trusts a caller-supplied identifier has the same missing-scope shape.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = a cross-app caller, sink = a reachable exported
  component acting on the intent, evidence = the resolved export state, the gate, and the drivable action.
