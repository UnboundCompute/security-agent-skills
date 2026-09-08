---
name: auditing-mobile-webview-bridge-exposure
description: >-
  Audit the trust a mobile app places in web content loaded in an embedded WebView, where a native bridge
  exposes app capabilities to JavaScript, letting web content that the app did not author, remote pages,
  loaded third-party frames, or attacker-influenced URLs, call native methods, read app data, or act with the
  app's privileges, because the bridge is exposed to untrusted origins, the loaded content is not
  origin-restricted, or the bridge methods do not check the caller. Use when a native app embeds a WebView
  that both loads remote or mixed content and exposes a native interface to script. Covers script-to-native
  bridges reachable by untrusted content, unrestricted content loading, and bridge methods that trust any
  caller. The untrusted web content reaching the bridge is the source, the native method it invokes is the
  sink, and web script driving native capability is the bug.
license: MIT
---

# Auditing mobile WebView bridge exposure: when a web page calls into the native app

A mobile app that embeds a WebView often gives the web content a way to call back into native code: a bridge
that exposes native methods to JavaScript so the page can use the camera, read files, fetch tokens, or drive
app features. That bridge is safe only when the content on the other side is trusted and confined, content the
app authored, loaded from an origin it controls, over a channel that cannot be tampered with. It becomes an
exposure when untrusted content reaches the bridge: a remote page the app loads, a third-party frame inside
it, a link the WebView follows to an attacker origin, or content injected over an insecure channel. Then web
script the app did not author invokes native methods and acts with the app's privileges. The bug is a native
capability reachable from web content the app does not fully trust, whether because the bridge is exposed to
untrusted origins, the WebView loads content it should not, or the bridge methods trust any caller. You audit
these by tracing which content can reach the bridge and what each exposed method does.

## When to use

- A native app embeds a WebView that exposes a native interface or message handler to JavaScript.
- The WebView loads remote content, third-party frames, or follows links to origins the app does not control.
- Bridge methods perform privileged actions or return app data without checking the calling origin.

## Scope check

Audit WebView bridges only on apps you own or are authorized to assess, on test devices and accounts, driving
the bridge with benign content you control rather than acting against real data. A confirmed exposure lets web
content wield native privilege, so keep every probe within scope. If you can't name the authorization, stop.

## The loop

1. **Establish which content can actually reach the bridge and whether the methods check the caller first.**
   For each exposed native interface, determine whether only trusted, app-authored content loaded from a
   controlled origin over a secure channel can reach it, or whether remote content, a third-party frame, a
   followed link, or injected content can, and whether each bridge method verifies the calling origin. This is
   the false-positive killer: a bridge reachable only by trusted confined content, or one whose methods
   authorize the caller, is not driven by an attacker even though it exposes native capability. Name the
   untrusted content path before crafting a call.

2. **Enumerate the exposed native methods.** List the methods the bridge exposes to JavaScript and what each
   does: privileged actions, data reads, capability access, or navigation. Note which are sensitive enough
   that web script invoking them would matter.

3. **Map what content can reach the bridge.** Determine what the WebView loads and whether it is confined to a
   controlled origin: does it load remote pages, allow third-party frames, follow links to arbitrary origins,
   or load content over a channel an attacker can tamper with. Any untrusted content that runs script in the
   WebView can reach an exposed bridge.

4. **Check the origin and caller restriction.** Determine whether the bridge is exposed only to trusted
   origins and whether each method verifies the caller before acting, or whether the interface is available to
   whatever content runs in the WebView and the methods trust any caller. A method that acts for any origin is
   the exposure.

5. **Check the content-loading confinement.** Determine whether the WebView restricts navigation to the app's
   own origins, blocks mixed and insecure content, and isolates any third-party frame from the bridge, or
   whether untrusted content can be loaded into the same context that holds the interface.

6. **Confirm and record.** Confirm by loading benign untrusted content you control into the WebView, through a
   followed link, a frame, or a tampered channel, and invoking a sensitive native method from that content on
   a test device. Kill the lead if only trusted confined content reaches the bridge, if each method authorizes
   the calling origin, and if navigation and framing are restricted to controlled origins. Record the method,
   the untrusted content path, the missing check, and the invocation observed, or set a `kill_reason`.

## Where the WebView bridge leaks

- **The untrusted-content path is the finding.** A bridge is expected to expose capability to trusted content;
  the bug is untrusted content reaching it. Name how attacker content gets into the WebView.
- **Remote and mixed content bring the attacker in.** A WebView that loads remote pages or allows insecure
  content lets a tampered or attacker page run script beside the interface.
- **Third-party frames inherit the bridge.** A frame from another origin loaded in the same WebView can reach
  an interface exposed to the whole context unless the bridge is confined to the top frame's trusted origin.
- **Followed links escape the trusted origin.** A WebView that follows links to arbitrary origins can navigate
  to an attacker page that then calls the bridge.
- **Methods that trust any caller are the sink.** A bridge method that performs a privileged action or returns
  data without checking the calling origin acts for whatever content invoked it.

## Worked example (a confirm and a kill)

> **Confirm.** A WebView exposes a native method that returns an authentication token and follows links to
> arbitrary origins. Navigating the WebView to a benign attacker-controlled page over a followed link lets that
> page's script invoke the method and receive the token, on a test device with a test account. **Confirmed**
> native capability reachable from untrusted web content, `high`, remediation = expose the bridge only to
> app-authored content on a controlled origin, verify the calling origin in each native method, restrict WebView
> navigation and framing to controlled origins, and block insecure and mixed content.
>
> **Kill.** The same WebView loads only app-authored content from a controlled origin over a secure channel,
> restricts navigation so links to other origins open outside the bridge context, isolates third-party frames
> from the interface, and each native method verifies the calling origin before acting. No untrusted content
> reaches the bridge. **Killed**, `kill_reason` = "the bridge is reachable only by trusted confined content on a
> controlled origin and every method authorizes the caller; untrusted web content cannot invoke a native
> method."

## Rationalizations to reject

- *"We only load our own pages."* -> Confirm the WebView cannot navigate or frame elsewhere; a followed link,
  a redirect, or an embedded frame can bring untrusted content into the same context as the bridge.
- *"The bridge only exposes harmless methods."* -> Enumerate every exposed method; one that returns a token,
  reads a file, or performs an action is enough, and harmless methods can be chained.
- *"The connection is HTTPS."* -> A secure channel to a remote origin still runs remote script beside the
  bridge; transport security does not make the loaded origin trusted.
- *"A frame cannot reach our interface."* -> If the interface is exposed to the whole WebView context, a
  third-party frame's script can reach it unless the bridge confines itself to the trusted top origin.
- *"The method just returns data."* -> Returning app data to untrusted web script is disclosure; treat a data
  read as a sensitive method and authorize the caller.

## Executing this in practice

You need every native method the bridge exposes, what each does, and whether it verifies the calling origin,
plus what the WebView loads and whether navigation, framing, and content security confine it to controlled
origins. For each exposed method, decide whether untrusted content can reach it through a remote page, a
frame, a followed link, or a tampered channel. Reading the bridge exposure and the WebView's loading policy
settles most leads; loading benign untrusted content you control and invoking a sensitive method on a test
device settles the rest.

## Related

- `auditing-mobile-deeplink-trust` - a deep link that drives the WebView to load attacker content is one way
  untrusted content reaches the bridge, joining link trust to bridge exposure.
- `testing-postmessage-and-web-message-trust` - a bridge reached through cross-frame messaging shares the
  origin-verification failure that skill treats in the pure-web setting.
- `hunting-mobile-tls-pinning-and-trust-gaps` - content loaded over a channel an attacker can tamper with
  reaches the bridge, the transport-trust gap that skill covers.
- `auditing-electron-ipc-trust` - the desktop analogue, where a renderer bridges to native capability and the
  same caller-authorization question applies.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the untrusted web content reaching the bridge, sink =
  the native method it invokes, evidence = benign attacker-controlled content invoking a sensitive native
  method on a test device.
