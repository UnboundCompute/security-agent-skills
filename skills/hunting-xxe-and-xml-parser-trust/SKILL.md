---
name: hunting-xxe-and-xml-parser-trust
description: >-
  Hunt XML external entity injection and unsafe XML parser features where untrusted XML is parsed with
  document type definitions, external general or parameter entities, or XInclude enabled. Covers in-band
  file disclosure and server-side request forgery through external entities, blind out-of-band exfiltration
  through parameter entities, and denial of service through entity expansion. The XML often hides inside
  other formats: office documents, SVG images, SOAP calls, and identity assertions all carry a parser that
  may resolve entities. The single fact that decides the class is the parser configuration on the specific
  library and version in use. Use when a service parses XML from a request body, an upload, or a federated
  message. The untrusted XML is the source, the entity-resolving XML parser is the sink, and file
  disclosure, request forgery, or expansion denial of service is the bug.
license: MIT
---

# Hunting XXE and XML parser trust: when the parser fetches what the document asks for

An XML parser is dangerous when it honors the parts of the XML specification that let a document reach
outside itself: a document type definition can declare external entities that the parser resolves by reading
a local file or fetching a URL, parameter entities can drive that resolution even when the result is never
reflected, and XInclude can pull in external content directly. Untrusted XML that reaches such a parser turns
into local file disclosure, server-side request forgery including to internal and cloud-metadata endpoints,
blind out-of-band exfiltration, and denial of service by recursive entity expansion. The XML is frequently
not obvious: office documents, SVG images, SOAP envelopes, and federated identity assertions all carry XML
into a parser. The one fact that governs everything is how the specific parser is configured. You find XXE by
locating every XML parse of untrusted input and reading whether DTDs and external entities are enabled.

## When to use

- A service parses XML from request bodies, file uploads, or federated messages.
- Uploads accept formats that contain XML, such as office documents or SVG images.
- A parser is constructed without an explicit secure configuration, or a default-insecure library is used.

## Scope check

Test XXE only against services you own or are authorized to assess, on non-production infrastructure,
because a confirmed case reads server files or reaches internal endpoints. Use inert markers and a
destination you control for any out-of-band probe, and stay within the authorized scope. If you can't name
the authorization, stop.

## The loop

1. **Establish the parser configuration first.** For each XML parse of untrusted input, read how the parser
   is constructed: whether DTD processing, external general entities, external parameter entities, and
   XInclude are enabled or disabled, on the exact library and version in use. This is the false-positive
   killer: a parser with secure processing on and external entities off is not vulnerable no matter what the
   document declares. Name the configuration before judging the document.

2. **Find every XML entry point, including hidden ones.** Inventory request bodies, upload handlers, and
   federated message endpoints, and note the formats that carry XML inside them: office documents, SVG,
   SOAP, and identity assertions. Each is a path to a parser and must be checked, not just the endpoints
   that declare an XML content type.

3. **Determine in-band versus blind reachability.** Decide whether the parsed result is reflected back to the
   caller, enabling direct in-band file or URL disclosure, or whether nothing is returned, in which case a
   parameter-entity, out-of-band channel to a destination you control is the confirmation path. The channel
   changes how you prove it, not whether it exists.

4. **Assess the reachable impact.** From an entity-resolving parser, determine what is reachable: local file
   reads, requests to internal services and cloud metadata, out-of-band exfiltration, and expansion-based
   denial of service. Assess this from the parser capability and the network position, using inert probes
   rather than pulling sensitive files.

5. **Read the neutralization that would hold.** Determine whether the parser disables DTDs entirely (the
   strongest control), or disables external general and parameter entities and XInclude, or runs a secure
   processing mode that does so. Check that the setting is applied to every parser instance on the path,
   including ones inside format-specific loaders, not just the primary one.

6. **Confirm and record.** Confirm by parsing a document with an inert external entity that reads a
   non-sensitive local marker or contacts a destination you control on an authorized instance, observing the
   read or the callback. Kill the lead if the parser disables DTDs or external entities and XInclude, if a
   secure processing mode is applied to every instance on the path, or if no untrusted XML reaches an
   entity-resolving parser. Record the entry point, the parser and its configuration, the channel (in-band
   or out-of-band), and the reachable impact.

## Where XXE leaks

- **The configuration is the whole bug.** A document can declare any entity; whether it resolves is decided
  entirely by how the parser is built.
- **Blind XXE needs parameter entities.** When nothing is reflected, an external parameter entity that drives
  an out-of-band request is the confirmation, so a parser is not safe just because it returns no content.
- **The XML hides in other formats.** Office documents, SVG, SOAP, and identity assertions each carry a
  parser; a loader for one of those formats can be vulnerable while the obvious XML endpoint is hardened.
- **One instance is enough.** A single parser on the path built without the secure setting reopens the class
  even when others are hardened.
- **Expansion is a separate impact.** Recursive entity expansion is a denial of service that needs no
  external fetch, so entity limits matter even when external access is blocked.

## Worked example (a confirm and a kill)

> **Confirm.** An import endpoint accepts an office document and parses its embedded XML with a parser built
> without a secure configuration, DTDs and external entities enabled. A document carrying an external entity
> that reads a non-sensitive local marker returns that marker in the response, and a parameter-entity variant
> reaches a destination the tester controls. **Confirmed** XXE with file disclosure and server-side request
> forgery, `high`, remediation = disable DTD processing on every XML parser on the path, or at minimum
> disable external general and parameter entities and XInclude, and apply the setting inside the
> document-format loader as well.
>
> **Kill.** The endpoint constructs every parser, including the one inside the document loader, with DTD
> processing disabled and secure processing on, so external entities and XInclude do not resolve. A document
> declaring an external entity parses with the entity unresolved and no request leaves the host. **Killed**,
> `kill_reason` = "every parser on the path disables DTDs and external entities with secure processing on;
> no declared entity resolves to a file read or a network fetch."

## Rationalizations to reject

- *"We do not accept XML."* -> Office documents, SVG, SOAP, and identity assertions carry XML into a parser;
  inventory the formats, not just the declared content type.
- *"Nothing is reflected, so it is safe."* -> Blind XXE uses parameter entities and an out-of-band channel;
  a silent response does not mean no resolution.
- *"The library is safe by default."* -> Defaults vary by library and version and change over time; read the
  actual construction of the parser instance in use.
- *"We hardened the main parser."* -> A single unhardened instance, often inside a format loader, is enough;
  verify every parser on the path.
- *"We block external access at the firewall."* -> Local file reads and entity-expansion denial of service
  need no outbound access; the parser setting is still required.

## Executing this in practice

You need every place untrusted XML is parsed, the formats that carry XML into those parsers, the exact parser
construction and version, and whether DTDs, external entities, and XInclude are enabled. For each, decide
whether the configuration resolves entities and what a resolved entity reaches. Reading the parser
construction shows the configuration; parsing a document with an inert entity that reads a marker or contacts
a controlled destination shows whether it resolves.

## Related

- `hunting-xpath-and-xml-query-injection` - the other XML-surface class, where the bug is a query built by
  concatenation rather than an entity-resolving parser.
- `exploiting-ssrf-to-cloud-metadata` - the escalation path when an external entity reaches an internal or
  metadata endpoint.
- `auditing-file-upload-and-content-handling` - the upload angle, where a format that carries XML reaches a
  parser through an upload handler.
- `hunting-dns-rebinding-and-ssrf-pivots` - a related outbound-request pivot useful when the entity fetch
  reaches internal services.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the untrusted XML, sink = the entity-resolving XML
  parser, evidence = an inert entity reading a local marker or reaching a controlled destination.
