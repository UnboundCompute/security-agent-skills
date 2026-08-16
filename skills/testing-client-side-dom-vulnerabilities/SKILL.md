---
name: testing-client-side-dom-vulnerabilities
description: >-
  Test the vulnerabilities that live entirely in the browser, where the server is
  never the sink: DOM-based cross-site scripting from client-side sinks, DOM
  clobbering, prototype pollution that corrupts application logic, unsafe cross-window
  messaging, client-side path and open-redirect handling, and cross-origin
  information leaks. Use when reviewing a single-page app, heavy client-side
  JavaScript, a browser extension, or any code that reads attacker-influenceable
  input and writes it into the DOM, a sink, or a shared object. The taint never
  reaches the server.
license: MIT
---

# Testing client-side DOM vulnerabilities: the server never sees it

A whole class of vulnerabilities never touches the server: the tainted input is read
and the dangerous action taken entirely in the browser, so server-side review and
server logs see nothing. The source is a URL fragment, a message, a piece of markup,
or a global the attacker can influence; the sink is a script execution, a corrupted
object, or a leaked cross-origin fact. Testing them means tracing taint inside the
page, from client source to client sink.

## When to use

- You are reviewing a single-page app, heavy client-side JavaScript, or a browser
  extension.
- Code reads the URL, a message, storage, or the DOM and writes it into a sink.
- The server sees none of the payload, so server-side testing found nothing.

## Scope check

Test pages and extensions you own or are authorized to test. Use benign markers and
your own browser session. If you can't name the authorization, stop.

## The loop

1. **Enumerate client-side sources and sinks.** List where the page reads
   attacker-influenceable input in the browser (the URL and its fragment, message
   data, referrer, storage, the DOM itself) and where it writes to a dangerous sink
   (HTML insertion, script or event execution, navigation, object property
   assignment). A source reaching a sink without sanitization is the bug.

2. **Test DOM-based cross-site scripting.** Does any client source flow into a markup
   or code sink (assigning HTML, writing an attribute that executes, building a
   handler) without encoding? If a value from the fragment or a message becomes
   executable in the page, it is DOM XSS, and it fires without the server ever seeing
   the payload.

3. **Test DOM clobbering.** Can attacker-injected markup, in a context that allows
   some HTML but not script, define named elements that shadow the globals or
   properties the page's script relies on, redirecting logic or supplying a value the
   script trusts? Look for script that reads a global that markup could define.

4. **Test prototype pollution.** Can attacker-controlled keys reach a recursive merge,
   a query or path parser, or an assignment that walks into a base object's prototype,
   injecting a property every object then inherits? Trace whether a polluted property
   later reaches a gadget (a sink that reads it) to turn corruption into script
   execution or a logic bypass.

5. **Test cross-window messaging and navigation.** Does a message handler act on data
   without verifying the sender origin, or does client-side routing take a path or
   target from input that can become an open redirect or a client-side path traversal?
   An unauthenticated message handler that writes to a sink is an injection channel
   from any window.

6. **Test cross-origin leaks and record.** Can another origin infer state (whether you
   are logged in, a value) from an observable side channel the page exposes? Rate by
   impact: script execution and account actions are high; a pure information leak is
   lower. Record confirmed source-to-sink flows and inputs proven encoded or
   origin-checked (killed) in the schema.

## Why the server never sees it

- **The taint stays in the browser.** A fragment-to-sink flow leaves no server request,
  so server-side testing and logs miss it entirely.
- **Some sinks are not obvious.** An assigned property, a defined named element, or a
  merged key can be as dangerous as writing markup.
- **Origin is the client-side boundary.** A message handler without an origin check
  trusts every window.
- **Encoding is context-specific.** The right defense depends on the exact sink; the
  wrong encoder is no defense.

## Worked example (a confirm and a kill)

> **Confirm.** A single-page app reads a value from the URL fragment and assigns it
> into an element's HTML to render a section label, with no encoding. A crafted
> fragment injects a markup payload that executes in the page. The server never
> receives the fragment. **Confirmed** DOM-based XSS, `high`, remediation = write the
> value as text or encode for the HTML context, never assign untrusted input as
> markup.
>
> **Kill.** A message handler verifies the sender origin against an allowlist before
> acting, all client sources are written to sinks as text through a context-correct
> encoder, object merges reject prototype-walking keys, and routing matches paths
> against a fixed table. Injected fragments, messages, and keys reach no executable or
> logic sink. **Killed**, `kill_reason` = "origin-checked messaging, text-only encoded
> sinks, prototype-safe merges, table-matched routing; no client source reaches a
> dangerous sink."

## Rationalizations to reject

- *"The server sanitizes input."* → Irrelevant; the payload never reaches the server.
  The sink is in the browser.
- *"It's just a URL fragment."* → The fragment is attacker-controlled and readable by
  script. It is a taint source.
- *"We only merge trusted objects."* → If any key is attacker-influenceable, the merge
  can pollute the prototype. Check the keys.
- *"The message came from our own page."* → Only if you verified the origin. Without
  that check, any window can send it.

## Executing this in practice

You need to trace taint inside the page, from client sources to client sinks, and to
influence those sources (fragment, message, markup, storage) while observing the sink.
Browser tooling and client-side dataflow tracing are enough; the source-to-sink map
inside the browser is the method.

## Related

- `adjudicating-taint-paths` - the same source-to-sink discipline, applied inside the
  browser.
- `testing-llm-insecure-output-handling` - a rendered sink that trusts untrusted
  content, on the output side.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the client-side input
  (fragment, message, markup, key), sink = the browser execution or corrupted object
  it reaches.
