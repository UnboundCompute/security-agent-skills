---
name: auditing-ios-app-group-and-pasteboard-exposure
description: >-
  Audit sensitive data leaving an iOS app's protection through shared containers and system-wide channels,
  where a secret or private value is written to a shared app-group container, a shared keychain access group,
  the general pasteboard, or app state that lands in snapshots and extensions, so another app, an app
  extension, or any process reading the pasteboard can recover it, because the shared surface is broader than
  the data's sensitivity requires. Use when reviewing what an iOS app shares with its own extensions or other
  apps and what it copies or exposes system-wide. Covers app-group container leakage, over-broad shared
  keychain groups, general-pasteboard secrets, and snapshot or extension data exposure. The sensitive value
  placed on a shared surface is the source, the other app or process reading that surface is the sink, and
  recovering the value outside the app's boundary is the bug.
license: MIT
---

# Auditing iOS app group and pasteboard exposure: when a secret leaves the app through a shared door

An iOS app is meant to be a sealed container, but it has several legitimate doors through which data leaves:
an app-group container it shares with its own extensions, a shared keychain access group, the general
pasteboard any app can read, and the state the system captures in snapshots or hands to extensions. Each door
is fine for data whose sensitivity tolerates the audience on the other side, and a leak when a secret goes
through a door wider than it should. A token written to an app-group container is readable by every extension
and app in that group; a secret in an over-broad shared keychain group is reachable by apps that should not
have it; a value copied to the general pasteboard is readable by any app, including ones the user never
launched deliberately; and sensitive data left on screen or in memory lands in a snapshot or an extension
context. The bug is a sensitive value on a shared surface whose audience exceeds the app's own boundary. You
audit these by listing what the app writes to each shared surface and who can read it.

## When to use

- An iOS app shares data with its own extensions through an app group or a shared keychain access group.
- The app copies or exposes values through the general pasteboard or hands data to an extension.
- Sensitive data may appear in a snapshot, a shared container, or a channel other apps can read.

## Scope check

Audit shared-surface exposure only on apps and devices you own or are authorized to assess, on test devices
and accounts, recovering only test data from the shared surface rather than a real user's. A confirmed leak
lets another app or process read the app's private data, so keep every probe within scope. If you can't name
the authorization, stop.

## The loop

1. **Establish the audience of each shared surface and the sensitivity of what is written to it first.** For
   each shared surface the app uses, an app-group container, a shared keychain group, the general pasteboard,
   a snapshot, or an extension handoff, determine who can read it and whether the data written there tolerates
   that audience. This is the false-positive killer: a shared surface carrying only data whose sensitivity
   matches its audience is not a leak, and a sensitive value never written to a shared surface is not exposed.
   Name the surface, its audience, and the sensitive value before claiming a leak.

2. **Enumerate the shared surfaces and their contents.** List the app-group containers, shared keychain access
   groups, pasteboard uses, and extension or snapshot data paths, and for each what the app writes there.
   Note which carry secrets, tokens, or private data.

3. **Check the app-group and shared-keychain scope.** Determine whether the app-group container or shared
   keychain access group is scoped only to the app and its own extensions, or is broad enough that another
   app can join or read it, and whether the data placed there needs to be shared at all.

4. **Check the pasteboard and snapshot exposure.** Determine whether sensitive values reach the general
   pasteboard, readable by any app, rather than a scoped or expiring pasteboard, and whether sensitive data
   left visible or in memory is captured in a snapshot or exposed to an extension that should not see it.

5. **Check whether the sharing is necessary.** Determine whether each sensitive value genuinely needs the
   shared surface, or whether it could stay in the app's private container or keychain scope. Data that need
   not be shared should not be on a shared surface at all.

6. **Confirm and record.** Confirm by recovering a sensitive test value from the shared surface as another app,
   an extension, or a pasteboard reader on a test device, or observing it in a snapshot. Kill the lead if
   every shared surface carries only data whose sensitivity matches its audience, if no secret reaches the
   general pasteboard or a snapshot, and if shared scopes are limited to the app and its own extensions.
   Record the surface, its audience, the leaked value, and the recovery, or set a `kill_reason`.

## Where shared-surface exposure leaks

- **The audience of the surface is the finding.** A shared door is expected; the leak is a sensitive value
  whose audience on the far side exceeds the app's boundary. Name the surface and who reads it.
- **App-group containers are readable by the whole group.** A token or secret written to an app-group
  container is available to every extension and app in that group, not just the one that wrote it.
- **Over-broad shared keychain groups widen reach.** A shared keychain access group larger than the app and
  its own extensions lets apps that should not hold a secret read it.
- **The general pasteboard is world-readable.** A value copied to the general pasteboard can be read by any
  app, so sensitive data belongs on a scoped or expiring pasteboard, or not copied at all.
- **Snapshots and extensions capture on-screen state.** Sensitive data left visible or in memory is captured
  in a snapshot or handed to an extension, leaking it outside the deliberate flow.

## Worked example (a confirm and a kill)

> **Confirm.** An app writes an authentication token to an app-group container so its extension can use it,
> but the group is broad and the token is stored in the clear. A separate app in the group reads the token
> from the shared container on a test device, recovering a usable test credential. **Confirmed** private data
> exposure through an over-broad shared container, `high`, remediation = scope the app group and shared
> keychain access group to the app and its own extensions only, store shared secrets bound to a key the
> reader must hold, and keep values that need not be shared in the app's private container.
>
> **Kill.** The same app shares only a non-sensitive value through an app group scoped to the app and its own
> extensions, keeps the token in its private keychain scope, never copies secrets to the general pasteboard,
> and marks sensitive screens so they are excluded from snapshots. No other app or process recovers a
> sensitive value. **Killed**, `kill_reason` = "shared surfaces carry only data whose audience matches its
> sensitivity, the token stays in a private scope, and no secret reaches the pasteboard or a snapshot; nothing
> sensitive leaves the app boundary."

## Rationalizations to reject

- *"The app group is only for our own extension."* -> Confirm the group and shared keychain scope actually
  exclude other apps; a broad group or access group lets an app the user did not intend read the data.
- *"The pasteboard is convenient for the user."* -> The general pasteboard is readable by any app; use a
  scoped or expiring pasteboard for sensitive values, or do not copy them at all.
- *"That container is inside the sandbox."* -> A shared container is shared by design; its audience is the
  whole group, so sandbox membership does not make it private to one app.
- *"Snapshots are internal to the OS."* -> A snapshot of a sensitive screen can be read from the device or a
  backup; exclude sensitive screens from capture rather than assuming the snapshot is safe.
- *"The value has to be shared."* -> Confirm it does; much shared data need not be, and the fix is often to
  keep it in the private container rather than to widen the audience.

## Executing this in practice

You need every shared surface the app uses, app-group containers, shared keychain access groups, pasteboard
uses, snapshot and extension data paths, with the audience of each and what the app writes there. For each
sensitive value, decide whether its surface's audience exceeds the app's boundary and whether it needs to be
shared at all. Reading what is written and the scope of each surface settles most leads; recovering a sensitive
test value as another app, an extension, or a pasteboard reader on a test device settles the rest.

## Related

- `hunting-ios-keychain-and-data-protection-gaps` - a secret that leaves the keychain onto a shared surface
  loses the protection class that skill audits, so storage class and sharing scope are adjacent.
- `hunting-mobile-secret-and-storage-exposure` - the broader secret-at-rest hunt, of which a shared-container
  or pasteboard leak is one exposure channel.
- `auditing-android-component-exposure` - the Android analogue, where a content provider or exported component
  is the shared door another app reads through.
- `auditing-mobile-webview-bridge-exposure` - a WebView that reads a shared surface can hand its contents to
  web content, chaining a shared-data leak into the bridge that skill audits.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the sensitive value placed on a shared surface, sink
  = the other app or process reading that surface, evidence = recovering the value outside the app's boundary
  on a test device.
