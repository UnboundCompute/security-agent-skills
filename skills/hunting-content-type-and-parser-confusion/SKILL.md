---
name: hunting-content-type-and-parser-confusion
description: >-
  Hunt content-type sniffing and parser confusion where untrusted content is served or consumed with an
  ambiguous or attacker-influenced type, so a browser sniffs a response as HTML, a backend picks a
  different interpretation than the sender intended, or two parsers on one path disagree. Use when user
  content is echoed with a missing, wrong, or user-controlled content type, when uploads are typed by
  extension or by the client, or when a request body can be parsed more than one way. Covers response
  sniffing, upload-filter bypass, polyglot files, charset-driven scripting, and multipart differentials.
  The ambiguously typed content is the source, the sniffer or parser that resolves the type is the sink,
  and the interpretation the attacker forces is the bug.
license: MIT
---

# Hunting content-type and parser confusion: when two parties disagree on what the bytes are

A stream of bytes has no inherent type; something decides whether it is HTML, JSON, an image, or a form.
When that decision is ambiguous, the attacker gets to influence it. A browser that sniffs a response
ignores a benign declared type and runs the HTML it finds inside. An upload filter that trusts the client
type or the extension accepts a script disguised as an image. Two parsers on one path, a permissive
multipart reader and a strict one, split a request differently and let content slip past the checker into
the sink. A single file that is valid as two formats passes an image check and executes as markup. The bug
is never the bytes; it is that the type is resolved by guessing rather than by an authoritative, enforced
declaration. You find these by locating where content is typed and asking who decides and whether anyone
can be made to disagree.

## When to use

- User-controlled content is returned in a response with a missing, generic, or user-influenced type.
- Uploads are accepted and typed by client-supplied content type or by file extension rather than content.
- A request body or document can be parsed as more than one format along its path through the system.

## Scope check

Test typing and parsing behavior only against applications you own or are authorized to assess, from test
accounts, using benign markers rather than live payloads when you confirm that content is sniffed or a
filter is bypassed. A confirmed stored case can execute in other users' sessions, so coordinate and stay in
scope. If you can't name the authorization, stop.

## The loop

1. **Establish the declared type and whether it is enforced first.** For each response that echoes user
   content and each parser that consumes untrusted bytes, determine what type is declared and whether it is
   authoritative: is a correct content type and a nosniff response directive set, and does the consumer honor
   the declared type instead of guessing? This is the false-positive killer. A response with the right type
   and nosniff will not be sniffed as HTML, and a parser that binds to one declared format cannot be
   confused, so name the declared type and its enforcement before crafting anything.

2. **Map where content is typed and consumed.** Trace responses that reflect or store user content, upload
   handlers and where they read the type, and every point where a body or document is parsed. Note each
   place the type is inferred (sniffed, taken from the extension, taken from the client header) rather than
   fixed by the server, because inference is where confusion enters.

3. **Check response sniffing.** For a response carrying user content, determine whether a browser could sniff
   it as HTML: a missing or generic type, a wrong charset, or content that starts with markup, without a
   nosniff directive, lets injected HTML run. A response with the correct type and nosniff, or served as a
   download with the right disposition, is not sniffable.

4. **Check upload and parser typing.** For uploads, determine whether the accepted type is decided by content
   inspection or by a trusted client value or extension; a filter that trusts the client accepts a disguised
   file. For bodies, check whether two parsers on the path could interpret the same bytes differently, so a
   value the checker reads is not the value the sink reads.

5. **Check the polyglot and charset variants.** Consider a single file valid as two formats that passes one
   parser's validation and is executed by another, and a charset the response omits that a browser infers to
   turn otherwise-inert bytes into script. Decide whether the pipeline forces a single unambiguous
   interpretation or leaves room for a second one.

6. **Confirm and record.** Confirm by serving or storing a benign marker that is inert under the intended
   type but active under the confused one, and observing the confused interpretation reached, on an isolated
   instance. Kill the lead if responses carry the correct type, charset, and nosniff, if uploads are typed by
   content and served with a safe disposition and type, if a single parser binds the format unambiguously, or
   if no consumer sniffs or re-types the content. Record the content, the deciding point, the forced
   interpretation, and its impact, or set a `kill_reason`.

## Where content-type confusion leaks

- **A missing or generic type invites sniffing.** Without an explicit correct type and nosniff, a browser
  guesses, and user content that looks like HTML runs as HTML.
- **Client-supplied type is not evidence.** An upload's declared content type and its extension are both
  attacker-controlled; only content inspection and safe serving decide what a file really is.
- **Two parsers are one too many.** When a checker and a sink parse the same bytes with different rules, the
  attacker satisfies the checker and steers the sink, smuggling content past validation.
- **Polyglots defeat single-format checks.** A file valid as an image and as markup passes an image filter
  and executes when served or rendered as a document.
- **Charset is part of the type.** An omitted or overridable charset lets a browser infer an encoding that
  reinterprets bytes as script, so charset must be pinned alongside the type.

## Worked example (a confirm and a kill)

> **Confirm.** An endpoint stores a user note and later serves it with a generic type and no nosniff
> directive. A note whose contents begin with markup is sniffed by the browser as HTML and its embedded
> script runs in another user's session on an isolated instance. **Confirmed** content-type sniffing to
> stored scripting, `high`, remediation = serve user content with an exact non-HTML content type, a pinned
> charset, and a nosniff response directive, or return it as an attachment with a safe disposition.
>
> **Kill.** The same endpoint serves stored notes with an exact text content type, a pinned charset, and a
> nosniff directive, uploads are typed by content inspection and stored under a fixed non-executable type,
> and each body is parsed by a single format-bound parser. A note beginning with markup is delivered as
> inert text and never sniffed as HTML. **Killed**, `kill_reason` = "user content is served with the correct
> type, charset, and nosniff, uploads are typed by content and served safely, and no second parser re-types
> the bytes; no consumer can be made to interpret them as HTML."

## Rationalizations to reject

- *"We set a content type."* -> Only if it is exact and paired with nosniff and a charset; a generic type
  without nosniff still lets a browser sniff the body as HTML.
- *"The upload is an image, the extension says so."* -> The extension and the client type are attacker-set;
  only content inspection and safe serving establish what the file is.
- *"Our validator checked the body."* -> If a different parser downstream reads the same bytes differently,
  the validator checked a value the sink never used.
- *"It is just a data response, not a page."* -> A browser that sniffs it as HTML makes it a page; the type
  and nosniff decide that, not your intent.
- *"The file passed the image check."* -> A polyglot passes an image check and still executes as markup when
  served or rendered; one-format validation is not a single interpretation.

## Executing this in practice

You need every response that echoes or stores user content with its declared type, charset, and nosniff
state, every upload handler with how it decides the type and how it serves the file, and every point where a
body or document is parsed, noting where two parsers could diverge. For each, decide whether the type is
authoritatively fixed or inferred, and whether an attacker can force a second interpretation. Reading the
response headers and the typing logic settles most leads; a marker that is inert under one type and active
under another, on an isolated instance, settles the rest.

## Related

- `hunting-reflected-and-stored-xss` - sniffing a response as HTML is one route to the same script execution
  that skill hunts through output context; the two meet on responses that echo user content.
- `auditing-file-upload-and-content-handling` - the upload typing and safe-serving decisions are that skill's
  core, so an upload-filter bypass through type confusion is shared ground.
- `reviewing-content-security-policy` - a strong policy limits what sniffed or injected markup can do, so a
  weak policy and a sniffable response compound.
- `hunting-http-parameter-pollution` - the sibling shape where components disagree on parameter parsing
  rather than on content type; both exploit two parties reading the same input differently.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the ambiguously typed content, sink = the sniffer or
  parser that resolves the type, evidence = the forced interpretation (script running, a disguised upload
  accepted) reached on an isolated instance.
