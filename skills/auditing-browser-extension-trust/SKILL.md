---
name: auditing-browser-extension-trust
description: >-
  Audit a browser extension (Manifest V3) for a trust boundary another web page or extension can cross to
  reach a privileged capability, after the permission scope and the message-sender checks are resolved.
  Covers an externally_connectable or onMessageExternal handler that verifies an origin but not the
  calling script, a content-script-to-background message handler with no sender validation, host
  permissions broader than the extension needs, a web_accessible_resources page that acts on URL
  parameters, an injected content script writing page-controlled data to a DOM sink, and a weak or
  eval-permitting content-security policy. Use when reviewing the manifest, background and content
  scripts, and cross-context message passing, not the web-page DOM sink taxonomy the client-side DOM skill
  owns. An untrusted web origin or another extension is the source, a privileged extension API or DOM sink
  is the sink, and a message reaching it without a sender-and-origin check is the bug.
license: MIT
---

# Auditing browser extension trust: what another page can reach through the extension

A browser extension is dangerous where a web page it does not trust, or another installed extension, can
drive one of its privileged capabilities: a host-scoped fetch that carries the user's cookies, a
`chrome`-namespace API, or a DOM write in a content script that runs with the extension's reach. The bug
is almost always a message boundary that verifies too little: an external handler that checks the calling
domain but not that a trusted script is calling, or a content-script-to-background channel that trusts any
sender. You audit it by resolving, per privileged sink, whether every path that reaches it validates the
sender identity and origin and whether the permission it uses is scoped to what the extension needs. Stay
on the extension's trust boundary; the web-page DOM sink taxonomy belongs to the client-side DOM skill.

## When to use

- You are reviewing a browser-extension manifest, its background service worker, and its content scripts.
- The extension exposes an external message handler, a cross-context channel, or web-accessible resources.
- You want to know what an untrusted page or extension can reach through the extension's privileges.

## Scope check

Audit only extensions you own or are authorized to assess, and drive a message boundary only against an
extension and profile in scope, a privileged handler acts on the real browser session. Adjudicate on the
manifest and the handlers. If you can't name the authorization, stop.

## The loop

1. **Resolve the permission scope and the sender checks first.** Read the manifest for host permissions
   (a broad all-urls match versus an activeTab or a narrow host list) and for the external-connection
   surface, and read each message handler for what it validates about the sender. A privileged sink is
   only an attack surface when an untrusted caller can reach it and the permission it uses is broad enough
   to matter, so settle both before flagging.

2. **Check the external message surface.** Look for an externally_connectable configuration or an
   onMessageExternal handler that a web page or another extension can call, and check whether it verifies
   the origin and that the calling script is one it trusts. Verifying the domain but not the script, or
   matching origins with a wildcard, lets an untrusted page drive the handler.

3. **Check the content-script-to-background channel.** Look for a background message handler that performs
   a privileged action on a message from a content script without validating the sender, so any page's
   injected script can send the message and drive the action.

4. **Check host permissions and web-accessible resources.** Look for host permissions broader than the
   feature needs (all-urls where activeTab or a single host would do), granting mass cookie and page
   access. And look for a web_accessible_resources page that reads URL parameters and performs a
   privileged action or reflects them into its DOM, making the operation website-triggerable.

5. **Check DOM sinks and the content-security policy.** Look for an injected content script writing
   page-controlled data to an innerHTML-class sink, giving a universal cross-site-scripting with the
   extension's origin, and for a content-security policy that permits eval or remote script where the
   platform would otherwise forbid it.

6. **Confirm and record.** Confirm by showing an attacker-controlled web origin or an arbitrary extension
   reaches a privileged API, a host-scoped fetch, or a DOM sink without passing a sender-and-origin check.
   Kill the lead if the handler validates the sender id and origin against an allowlist, if the exposed
   function only reads and reaches no privileged sink, if the host permission is activeTab (user-gesture
   scoped) or a narrow host rather than all-urls, if the web-accessible resource takes no parameters and
   performs nothing privileged (mere exposure is not a bug), if the DOM write uses a text or attribute sink
   or a sanitizer or writes only an extension-controlled constant, or if the flagged eval permission is not
   actually present in the emitted policy. Record the entry point, the missing check, and the sink reached.

## Where extension trust leaks

- **Verifying the origin but not the script is the classic gap.** An external handler that checks the
  calling domain yet not that a trusted script is calling lets an untrusted page on that domain drive it.
- **A content script runs in an isolated world.** Page JavaScript cannot call the extension without an
  exposed channel; an exposed function that only reads is not a sink, but one that acts on any sender is.
- **Over-broad host permission is the amplifier.** All-urls turns any reachable handler into mass cookie and
  page access; activeTab or a narrow host list keeps the blast radius small.
- **A parameterized web-accessible page is website-triggerable.** A resource that acts on URL parameters
  lets a website drive a privileged operation; one that takes no parameters and does nothing privileged is
  just exposed, not vulnerable.
- **A content-script DOM write inherits the extension origin.** Page-controlled data into an innerHTML-class
  sink is a universal cross-site-scripting; a text or attribute sink, or a sanitizer, kills it.

## Worked example (a confirm and a kill)

> **Confirm.** A background service worker handles an external message that names a URL and fetches it with
> the extension's host permissions, returning the response to the caller. The handler checks the message
> shape but not the sender, and externally_connectable matches all pages. Any web page posts the message
> and reads a cross-origin resource through the extension's privileges. **Confirmed** an unauthenticated
> external handler reaching a host-scoped fetch, `high`, remediation = restrict externally_connectable to
> trusted origins, validate the sender id, and scope the fetch to an allowlist.
>
> **Kill.** The same handler validates `sender.id` against a small allowlist and externally_connectable
> lists only the vendor's own origins; a page from any other origin cannot reach it, and the host
> permission is a single named host, not all-urls. **Killed**, `kill_reason` = "the external handler
> validates the sender id and origin against an allowlist and the fetch is scoped to one host; no untrusted
> caller reaches a privileged sink."

## Rationalizations to reject

- *"We check the origin."* -> Do you check that a trusted script is calling, not just the domain? Verifying
  the origin but not the caller lets any page on that origin drive the handler.
- *"Content scripts are isolated."* -> They are, until you expose a message channel; a handler that acts on
  any sender is the channel that defeats the isolation.
- *"We need broad host access."* -> Would activeTab or a named host do? All-urls turns every reachable
  handler into mass cookie and page access; scope it to the feature's need.
- *"It is just a web-accessible file."* -> Does it read URL parameters and do something privileged? A
  parameterized resource is website-triggerable; a passive file is only exposed.
- *"The content script writes to the page."* -> With what sink? Page-controlled data into innerHTML is a
  universal cross-site-scripting; confirm it is a text or attribute sink or is sanitized.

## Executing this in practice

You need the manifest (host permissions, externally_connectable, web_accessible_resources, and the
content-security policy), the background handlers and what each validates about the sender, the content
scripts and their DOM writes, and the map from each entry point to the privileged sink it can reach. For
each sink, decide whether an untrusted caller can reach it without a sender-and-origin check and whether
the permission it uses is scoped. Reading the manifest tells you the surface; reading the handlers tells
you what an untrusted caller can drive.

## Related

- `auditing-electron-ipc-trust` - the desktop-app sibling with the same untrusted-content-to-privileged-
  capability shape, across the Electron preload bridge rather than the extension message boundary.
- `auditing-editor-extension-workspace-trust` - the editor-extension sibling, where the untrusted source is
  a cloned repository rather than a web page or another extension.
- `testing-client-side-dom-vulnerabilities` - owns the web-page DOM sink taxonomy this skill references for
  the content-script injection shape; the trust boundary (isolated world, host permissions) is unique here.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = an untrusted web origin or another extension, sink
  = a privileged extension API or DOM sink, evidence = the reachable entry point, the missing sender-and-
  origin check, and the privilege it reaches.
