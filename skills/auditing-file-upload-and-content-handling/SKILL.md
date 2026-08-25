---
name: auditing-file-upload-and-content-handling
description: >-
  Audit a file-upload and content-handling path for an attacker-supplied file whose bytes, declared type,
  name, or embedded content reach a sink that stores it in a served or executable location or feeds it to
  a parser that acts on its content, after the type-decision layer and the serve behavior are resolved.
  Covers an extension or content-type trusted for a type decision that a second layer contradicts, an SVG,
  HTML, or XML file stored and served inline as active content, image or document parser exploitation, a
  polyglot file passing one content check yet executing in another context, an upload path writing outside
  the intended directory, and an archive expanding to a write primitive. Use when reviewing upload
  validation, storage, and serving, not the client-side DOM sink or the archive-extraction write primitive
  their own skills own. An uploaded file is the source, a serve or parse sink acting on its content is the
  sink, and a type decision one layer contradicts is the bug.
license: MIT
---

# Auditing file upload and content handling: when the type one layer trusts another layer contradicts

A file upload becomes a vulnerability when the layer that decides a file's type disagrees with the layer
that later serves or parses it. The upload validator trusts the extension or the declared content-type;
the web server sniffs the bytes, or the storage path serves it inline, or a parser acts on embedded
content, and the two disagreements meet in the attacker's favor. You audit it by resolving how the type is
decided at upload and how the file is served and parsed afterward, then finding where those two views
diverge. The strongest false-positive killers live in the serve layer: a no-execute sandboxed origin with
a forced download, or a server-side re-encode, neutralizes the content regardless of its bytes. The
client-side DOM sink and the archive-extraction write primitive have their own skills; this skill owns the
upload-to-serve-or-parse decision.

## When to use

- You are reviewing an upload endpoint, its type validation, where files are stored, and how they are served.
- A file's bytes, declared type, name, or embedded content reach a parser, a renderer, or a served location.
- You want to know whether an uploaded file executes or is parsed dangerously somewhere downstream.

## Scope check

Audit only applications you own or are authorized to assess, and upload a crafted file only to a system in
scope, a stored active file can execute against real users. Adjudicate on the validation, the storage, and
the serve path. If you can't name the authorization, stop.

## The loop

1. **Resolve the type-decision layer and the serve behavior first.** Determine how the upload decides the
   file's type (an extension check, a declared content-type, a magic-byte sniff, or a combination) and how
   the file is served afterward (inline or as an attachment, from the app origin or a separate no-execute
   origin, with what content-type). The bug is a divergence between these two, so establish both before
   judging any single check.

2. **Check the type-decision contradiction.** Look for an extension or a client-declared content-type
   trusted for the type decision while a second layer (the web server's own sniffing, a content-disposition
   header, or the storage path's handler) treats the file differently, so a double extension, a null-byte
   or case trick, or an appended-data file is stored as one type and served as another.

3. **Check active-content serving.** Look for an SVG, HTML, or XML file stored and served inline as active
   content, giving a stored cross-site-scripting or an external-entity parse, when it should be served as an
   inert attachment or sanitized.

4. **Check parser exploitation.** Look for an uploaded image or document fed to a parser or converter that
   acts on its embedded content (a delegate command, an external reference, an embedded script), turning a
   crafted file into code execution or a server-side request in the parser.

5. **Check polyglots, path, and archives.** Look for a polyglot file that passes one content check yet
   executes in another context, an upload path that writes outside the intended directory (traversal in the
   filename), and an archive uploaded and expanded to a write primitive (hand the extraction internals to
   the archive-extraction skill, keep the upload framing here).

6. **Confirm and record.** Confirm by uploading a crafted file and showing it is stored and then executed or
   parsed dangerously along the reachable path. Kill the lead if the file is served from a no-execute
   separate origin with a forced attachment disposition and a fixed non-active content-type so the execution
   path is cut regardless of the bytes, if the server re-encodes the file (an image recompressed through a
   hardened pipeline) stripping any polyglot or parser payload, if the type is decided by a magic-byte
   allowlist and the extension and the serve layer all agree, if the stored filename is randomized and the
   path is outside the web root so a direct request cannot execute it, if the parser is hardened (delegates
   and external entities disabled) so the crafted content is inert, or if an SVG is sanitized or served with
   a policy that blocks inline script. Record the upload, the type divergence, and the serve or parse sink.

## Where content handling leaks

- **The bug is a divergence, not a single check.** The upload validator and the serve or parse layer holding
  different views of the file's type is the vulnerability; one layer alone rarely tells you.
- **The serve layer is the strongest killer.** A no-execute sandboxed origin with a forced download, or a
  fixed inert content-type, neutralizes the file whatever its bytes; resolve the serve behavior first.
- **Re-encoding strips payloads.** Recompressing an image through a hardened pipeline destroys polyglot and
  parser content; a validated-then-stored-verbatim file does not.
- **Active content served inline is stored injection.** An SVG, HTML, or XML served inline runs or parses;
  the same file forced as an attachment from a sandbox origin does not.
- **The parser is a sink too.** An image or document converter that acts on embedded delegates or external
  references is code execution or a server-side request; hardening the parser closes it.

## Worked example (a confirm and a kill)

> **Confirm.** An avatar upload validates only the file extension, stores the file under the application's
> web root with its original name, and the server serves it with a sniffable content-type. An attacker
> uploads an SVG (named to pass the extension check) containing a script; fetching the stored file executes
> the script in the application origin. **Confirmed** stored cross-site-scripting through an upload validated
> by extension and served inline as active content, `high`, remediation = serve user uploads as inert
> attachments from a separate no-execute origin, decide type by magic bytes, and sanitize or re-encode.
>
> **Kill.** The same upload re-encodes the image to a fixed format through a hardened pipeline, stores it
> under a randomized name outside the web root, and serves it from a separate user-content origin with a
> forced attachment disposition and a fixed non-active content-type. The crafted bytes do not survive the
> re-encode and the file cannot execute where it is served. **Killed**, `kill_reason` = "the file is
> re-encoded and served no-execute cross-origin with a forced download, so the content-handling sink is
> neutralized regardless of the uploaded bytes."

## Rationalizations to reject

- *"We check the extension."* -> Does the serve layer agree? A double extension or a sniffed type diverges
  from the extension; the contradiction between the layers is the bug.
- *"It is just an image."* -> Served inline from the app origin, or re-encoded and served as an attachment
  from a sandbox origin? An SVG or a polyglot served inline is active content.
- *"The parser is standard."* -> Are its delegates and external entities disabled? A converter that acts on
  embedded content is a code-execution or server-side-request sink.
- *"The filename is validated."* -> Against traversal, and stored where? A traversal in the name or a
  web-root path lets a direct request reach the file.
- *"It is a zip we extract."* -> The extraction write primitive is the archive-extraction skill's; here
  confirm the upload accepts and routes it, and hand the expansion there.

## Executing this in practice

You need the upload endpoint and its type validation, whether the file is re-encoded, where it is stored
(web root or not, original or randomized name), how it is served (inline or attachment, which origin, which
content-type), and any parser or converter the content reaches. For each upload, decide whether the
type-decision layer and the serve or parse layer diverge in the attacker's favor. Reading the validator
tells you what is accepted; reading the serve path tells you what happens to it.

## Related

- `hunting-unsafe-archive-extraction` - owns the archive-expansion write primitive (path traversal and
  resource exhaustion on extract); this skill routes an uploaded archive there rather than duplicating it.
- `testing-client-side-dom-vulnerabilities` - owns the client-side DOM sink taxonomy; the SVG-as-active-
  content shape here is the server-served, stored variant, framed as content handling rather than a DOM sink.
- `reviewing-content-security-policy` - the policy that blocks inline script in served active content is a
  mitigating layer this skill checks when adjudicating an SVG or HTML upload.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = an uploaded file, sink = a serve or parse operation
  acting on its content, evidence = the type-decision layer, the serve behavior, and the divergence between
  them.
