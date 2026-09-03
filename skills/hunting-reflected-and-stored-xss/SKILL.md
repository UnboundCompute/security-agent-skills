---
name: hunting-reflected-and-stored-xss
description: >-
  Hunt reflected and stored cross-site scripting in server-rendered responses, where untrusted request or
  stored data is placed into an HTML response in a context whose encoding is missing or wrong, so the value
  becomes markup or script that runs in the victim's session. Covers the output context that decides the
  correct encoding, HTML body versus quoted or unquoted attribute versus inline script versus URL versus
  CSS, a template raw or safe marker that disables autoescaping, JSON embedded in a page, and the reflected
  versus stored delivery paths. Use when reviewing server-side templates or string-built HTML that include
  request or database data; DOM-based sinks are a separate skill. The untrusted data rendered into the page
  is the source, the HTML output context is the sink, and script execution in the victim's session is the
  bug.
license: MIT
---

# Hunting reflected and stored XSS: when data becomes markup in the wrong context

A cross-site scripting bug is an encoding failure with a location. The server takes untrusted data,
reflected straight from the request or read back from storage, and writes it into an HTML response. If the
value is encoded correctly for the exact place it lands, it stays inert data; if it is not, it becomes
markup or script and runs in the victim's browser with the victim's session. The subtlety is that
"correctly encoded" depends entirely on the output context: the escaping that neutralizes a value in an
HTML body does nothing for a value inside an unquoted attribute, an inline script, a URL, or a stylesheet.
So the hunt is not "is this escaped" but "is this escaped for the context it is actually in." You find
these by locating every place server-rendered output includes untrusted data and matching the encoding to
the context, then following whether delivery is reflected from the request or stored and served later.

## When to use

- A server-side template or string-built response embeds request parameters or stored data into HTML.
- The framework autoescapes but code uses a raw, safe, or unescaped marker in places.
- User-controlled data is serialized into JSON or an attribute that a page then interprets.

## Scope check

Test XSS only against applications you own or are authorized to assess, with test accounts, and use a
harmless proof such as a benign marker rather than a real payload against other users, because stored XSS
executes in every viewer's session. Clean up any stored test value you inject. If you can't name the
authorization, stop.

## The loop

1. **Establish the exact output context and its encoding first.** For each place untrusted data reaches the
   response, determine the precise context, HTML body, quoted attribute, unquoted attribute, inline script,
   URL, or CSS, and whether the encoding applied matches that context. This is the false-positive killer:
   HTML-entity encoding is correct in a body but useless inside a script block or an unquoted attribute, so
   an escaped value can still be XSS in the wrong place, and a value that never leaves a body with correct
   entity encoding is inert. Name the context and its encoding before judging.

2. **Trace the data to the sink.** Follow request parameters and stored fields into the template or the
   string that builds the response. Confirm the attacker controls the value and note whether it is reflected
   directly from the current request or was stored earlier and is served to other users.

3. **Check autoescaping and its bypasses.** Determine whether the template engine autoescapes by default,
   then find every raw, safe, unescaped, or trusted-HTML marker that turns it off. Those markers, and
   string concatenation outside the template engine, are where autoescaped applications still bleed.

4. **Check the dangerous contexts specifically.** Inline event handlers and script blocks, unquoted
   attributes, href and src URLs that may accept a script scheme, style blocks, and data attributes read by
   client code each need context-specific handling; entity encoding alone does not make them safe. Flag data
   landing in these without the right encoding.

5. **Check the defenses that actually reduce impact.** Determine whether a content security policy would
   block inline execution or restrict script sources, and whether output is built through a context-aware
   auto-escaping engine rather than manual string assembly. A strong policy lowers severity but a
   context-mismatched injection is still a finding.

6. **Confirm and record.** Confirm by injecting a benign marker that would only render as active markup if
   the context is unescaped (observing it parse as an element or execute a harmless proof), for stored XSS by
   viewing as a second account. Kill the lead if the value is encoded correctly for its context, if the
   engine autoescapes and no raw marker applies, or if the value never reaches an executable context. Record
   the sink, the context, reflected or stored, and the marker's behavior. Set `kill_reason` when killing.

## Where XSS leaks

- **The context decides safety, not the presence of escaping.** Entity encoding is right for a body and
  wrong for a script block, an unquoted attribute, a URL, or CSS; match the encoding to the place.
- **Raw and safe markers disable the default.** An autoescaping engine protects until a template uses a
  raw, safe, or unescaped filter, or code concatenates HTML outside the engine.
- **Stored XSS reaches every viewer.** A value written once and served to others runs in each of their
  sessions, so an admin view of user-supplied data is a high-value target.
- **URL and script-scheme contexts are easy to miss.** A value placed into an href or src can carry a script
  scheme; entity encoding does not stop it, only scheme validation does.
- **JSON embedded in HTML has two contexts.** Data serialized into a script block must be safe both as JSON
  and against a closing script tag; encoding for one context leaves the other open.

## Worked example (a confirm and a kill)

> **Confirm.** A search results page reflects the query into the HTML body and also into an inline script
> variable. The body is entity-encoded, but the inline-script assignment concatenates the raw query, so a
> value that closes the script string and adds a statement executes in the visitor's session. A benign
> marker runs. **Confirmed** reflected XSS in an inline-script context, `high`, remediation = stop building
> inline script from untrusted data, serialize the value with a context-correct encoder or pass it through a
> data attribute read safely, and add a content security policy that forbids inline execution.
>
> **Kill.** A comment feature stores user text and renders it through an auto-escaping template that
> entity-encodes into the HTML body with no raw marker, links are built by validating the scheme against an
> allowlist, and a content security policy forbids inline script and restricts sources. A stored value
> containing markup renders as visible text and does not execute in a second account's view. **Killed**,
> `kill_reason` = "value is entity-encoded for its body context by an auto-escaping engine with no raw
> marker, URL schemes are allowlisted, and the policy blocks inline execution, so injected markup stays
> inert."

## Rationalizations to reject

- *"We escape everything."* -> Escaping for one context does not protect another; a body-escaped value in a
  script block or unquoted attribute is still XSS. Match the encoding to the context.
- *"The framework autoescapes."* -> Until a raw, safe, or unescaped marker turns it off or code builds HTML
  by hand; find those places rather than trusting the default globally.
- *"It is only reflected, not stored."* -> Reflected XSS is delivered by a crafted link and runs in the
  clicker's session; it is a full finding, not a lesser one.
- *"A CSP protects us."* -> A policy reduces impact and may block inline script, but a context-mismatched
  injection into an allowed source or a script-less sink can still fire; confirm, do not assume.
- *"It is inside a JSON blob."* -> JSON embedded in a page must also be safe against a closing script tag and
  the HTML context around it; JSON-encoding alone leaves that open.

## Executing this in practice

You need every place server-rendered output includes untrusted data, the exact context each value lands in,
and the encoding applied there, plus every raw or unescaped marker and every manual HTML concatenation. For
each, decide whether the encoding matches the context and whether delivery is reflected or stored. Reading
the template and the surrounding markup tells you the context; injecting a benign marker that only becomes
active in an unescaped context, viewed as a second account for stored cases, tells you whether it executes.

## Related

- `testing-client-side-dom-vulnerabilities` - the client-side variant where the sink is a DOM API rather
  than a server template; the same data may reach both, so check both.
- `reviewing-content-security-policy` - a policy is the compensating control that limits XSS impact; that
  skill covers whether the policy actually blocks inline and untrusted sources.
- `hunting-content-type-and-parser-confusion` - an upload the browser MIME-sniffs as HTML is a stored-XSS
  path through file handling rather than a template; adjacent sink.
- `adjudicating-taint-paths` - use it to connect a request field or a stored record to the exact output
  context through the rendering code.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the untrusted data rendered into the page, sink =
  the HTML output context, evidence = a benign marker parsing as active markup or executing a harmless proof
  in the victim context.
