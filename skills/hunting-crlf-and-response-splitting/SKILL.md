---
name: hunting-crlf-and-response-splitting
description: >-
  Hunt CRLF injection where untrusted input carrying a carriage return and line feed reaches a response
  header, a log line, or an outbound email header that does not strip or reject those characters, letting
  the attacker inject headers, split the message, forge log entries, or add email headers. Use when input
  flows into a Set-Cookie, Location, or custom response header, into a log written from request data, or
  into mail headers built from user input. Covers full response splitting into injected content or cache
  poisoning, header injection, log forging, and email header injection. The untrusted input is the source,
  the header, log, or email writer is the sink, and the injected line that the consumer treats as new
  structure is the bug.
license: MIT
---

# Hunting CRLF and response splitting: when a newline becomes a new header

Headers, log lines, and email headers are all line-oriented: a carriage return and line feed ends one
field and begins the next. When untrusted input flows into one of these unescaped, an attacker who supplies
a CRLF sequence stops writing a value and starts writing structure. A newline in a redirect location injects
a header, or a blank line followed by a body splits the response into a second one the attacker controls. A
newline in a log line forges an entry that frames another user or hides an action. A newline in a mail
header adds recipients or spoofs fields. The bug is not the input; it is a writer that concatenates
untrusted text into a line-structured format without rejecting the line terminators. You find these by
locating every header, log, and email write fed request data and checking whether CR and LF survive.

## When to use

- Request input flows into a response header value: a Set-Cookie, a Location redirect, or a custom header.
- Log lines are written from request data (paths, user agents, identifiers) without neutralizing newlines.
- Outbound email headers (recipients, subject, from, reply-to) are built from user-supplied input.

## Scope check

Test CRLF injection only against systems you own or are authorized to assess, on non-production
infrastructure, using benign markers rather than live cache-poisoning or spoofed mail when you confirm. A
confirmed split can affect other users through a shared cache, so keep every probe within the authorized
scope and prefer an isolated instance. If you can't name the authorization, stop.

## The loop

1. **Establish whether the writer rejects CR and LF first.** For each header, log, and email sink, determine
   whether the runtime or framework already forbids carriage returns and line feeds in the value, or whether
   a hand-built string or a permissive stack passes them through. This is the false-positive killer: most
   modern header APIs reject bare CR and LF, so an injected newline never reaches the wire, while a
   manually-assembled header or an older interface may not. Name the writer's behavior before crafting input.

2. **Map every line-structured write fed untrusted input.** Trace request values into response header
   assignments, into log-writing calls, and into outbound mail header fields. Note whether the value is set
   through an API that validates it or concatenated into a raw string. Each such write is a candidate sink.

3. **Distinguish header injection from full response splitting.** A single injected header (a forged
   Set-Cookie, a cache-control override) is one impact; a CRLF followed by a blank line and a body splits the
   response so the attacker controls a second message, which enables reflected content and cache poisoning.
   Decide which the sink allows based on how much of the response line structure the input reaches.

4. **Follow the consumer of the injected line.** A split response matters when a shared cache stores it or a
   browser renders the injected content; a forged cookie matters when the app trusts it; a forged log entry
   matters when an operator or a downstream parser trusts the log; an injected mail header matters when the
   mail transfer agent acts on it. Identify who consumes the injected structure and what they do with it.

5. **Check the neutralization that actually closes it.** The reliable control is to reject or strip CR and LF
   (and related separators) before the value reaches the writer, or to use an API that encodes or forbids
   them, and for logs to encode newlines in untrusted fields. Determine which stands between the input and
   each sink, and whether it covers every write rather than the obvious one.

6. **Confirm and record.** Confirm by injecting a benign marker header or a benign extra log field through
   the sink and observing it appear as structure on an isolated instance, and for splitting, a controlled
   second response or a poisoned cache entry keyed to a marker. Kill the lead if the writer rejects CR and
   LF, if the input is encoded or stripped before the write, or if no request data reaches a line-structured
   sink. Record the input, the sink, whether it is header injection or a full split, and the consumer, or set
   a `kill_reason`.

## Where CRLF injection leaks

- **The writer decides everything.** An API that rejects CR and LF makes the input inert; a raw string
  concatenation into a header or log passes them through. The sink's behavior is the finding.
- **Redirects and cookies are prime sinks.** A Location or Set-Cookie built from input is a common place a
  newline injects a header or splits the response.
- **Splitting reaches other users through caches.** A response split that a shared cache stores serves the
  attacker's injected content to everyone who requests the key, turning one request into stored impact.
- **Logs are trusted downstream.** A forged log line can frame another user, hide an action, or inject into a
  log parser or dashboard that treats each line as a record.
- **Email headers add recipients.** A newline in a user-supplied mail field can add recipients or spoof the
  from and reply-to, so contact and notification features are sinks.

## Worked example (a confirm and a kill)

> **Confirm.** A redirect endpoint copies a request parameter into the Location header by string
> concatenation on a stack that does not validate header values. A parameter containing a CRLF, a blank
> line, and a body splits the response into a second one whose content the attacker controls, and a shared
> cache stores it against the request key on an isolated instance. **Confirmed** HTTP response splitting to
> cache poisoning, `high`, remediation = set the Location through an API that rejects CR and LF, strip line
> terminators from the parameter before use, and validate the redirect target against an allowlist.
>
> **Kill.** The same endpoint sets the Location through a framework API that rejects any value containing CR
> or LF, strips line terminators from the parameter first, and the log writer encodes newlines in untrusted
> fields. A parameter carrying a CRLF is rejected before the header is written and no second response is
> produced. **Killed**, `kill_reason` = "header is set through an API that forbids CR and LF and the input is
> stripped first, and logs encode newlines; no injected line reaches a header, a split response, or a forged
> log entry."

## Rationalizations to reject

- *"Our framework builds the headers."* -> Only a finding-killer if that API rejects CR and LF; confirm it,
  because a hand-built header or a permissive interface elsewhere on the path may not.
- *"It is only a redirect parameter."* -> A newline in a Location injects a header or splits the response;
  the parameter is line-structured input into a header sink.
- *"Logs are internal."* -> A forged log line misleads operators and can inject into a downstream parser or
  dashboard; encode newlines in untrusted log fields regardless.
- *"We URL-encode the value."* -> Encoding helps only if it is applied before the write and the writer does
  not decode it again; confirm the CR and LF cannot reach the sink decoded.
- *"The email field is just a subject."* -> A newline in a subject or any user-supplied mail field can add
  headers and recipients; validate every mail field built from input.

## Executing this in practice

You need every response header, log, and outbound email write fed request data, and for each whether the
writer rejects CR and LF or concatenates a raw string, plus who consumes the resulting line. For each sink,
decide whether an injected newline survives to become structure and whether the consumer treats it as a new
header, a split response, a forged record, or an added recipient. Reading the writer settles most leads; a
benign marker header, log field, or a cache entry keyed to a marker on an isolated instance settles the rest.

## Related

- `testing-web-cache-attacks` - a split response stored by a shared cache is a poisoning primitive that skill
  covers, so response splitting feeds directly into it.
- `auditing-open-redirect-and-forced-navigation` - the Location header is a shared sink; a redirect target
  can be both an open redirect and a CRLF injection point, so audit them together.
- `auditing-session-lifecycle-and-fixation` - an injected Set-Cookie header can fix or overwrite a session,
  connecting header injection to the session concerns there.
- `testing-smtp-smuggling-and-email-spoofing` - email header injection is the application-layer cousin of the
  message-boundary confusion that skill treats at the protocol layer.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the untrusted input, sink = the header, log, or
  email writer, evidence = the injected header, split response, poisoned cache entry, forged log line, or
  added email header observed on an isolated instance.
