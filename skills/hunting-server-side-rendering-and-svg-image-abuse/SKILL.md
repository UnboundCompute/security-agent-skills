---
name: hunting-server-side-rendering-and-svg-image-abuse
description: >-
  Hunt abuse of server-side renderers of user-supplied markup, such as headless-browser PDF or screenshot
  generation, SVG rasterization, thumbnailers, and chart or document renderers, where the renderer fetches
  remote or local resources, follows redirects, executes embedded script, or reads local files while
  producing output. Use when untrusted HTML, SVG, or a URL is handed to a rendering component server-side.
  Covers server-side request forgery including to cloud metadata, local file disclosure through file
  schemes or external entities, blind out-of-band interaction, and script execution inside the generated
  document. The untrusted markup or URL is the source, the resolving renderer is the sink, and the fetch,
  file read, or script execution it performs is the bug.
license: MIT
---

# Hunting server-side rendering and SVG image abuse: when generating a document fetches the attacker's URL

To turn a user's content into a PDF, a screenshot, or a thumbnail, a server hands markup or a URL to a
renderer, and a renderer is a small browser: it resolves links, loads images, follows redirects, and often
runs script and reads local files. When that content is attacker-controlled, the renderer becomes a
request engine operating from inside the network, fetching an internal address or cloud-metadata endpoint,
reading a local file through a file scheme or an external entity, calling out to a host that proves the
render happened, or executing script that reads the rendered page. The bug is not the markup; it is a
renderer left free to resolve whatever the document points at, running server-side with network and file
reach the user never should have. You find these by locating every server-side render of untrusted content
and reading what the renderer is allowed to resolve.

## When to use

- A service renders user-supplied HTML or SVG server-side into a PDF, an image, a screenshot, or a preview.
- A thumbnailer, chart generator, or document converter processes untrusted files or URLs on the server.
- A feature fetches and renders a user-supplied URL to produce a preview, an archive, or an export.

## Scope check

Test server-side rendering only against systems you own or are authorized to assess, on non-production
infrastructure, using benign local markers and a destination you control for any fetch, never pulling real
secrets or reaching third-party addresses. A confirmed case reaches internal services, so keep every probe
inside the authorized scope and prefer an isolated instance. If you can't name the authorization, stop.

## The loop

1. **Establish what the renderer is allowed to resolve first.** For each renderer, determine whether it will
   fetch remote URLs, follow redirects, resolve local-file schemes, load external entities, and execute
   script, or whether it runs sandboxed with the network disabled, external subresources blocked, and script
   off. This is the false-positive killer: a renderer that resolves nothing external cannot be driven to
   fetch a URL or read a file no matter what the document contains, so name the renderer's capabilities
   before crafting a document.

2. **Map every server-side render of untrusted content.** Trace user HTML and SVG, uploaded documents, and
   user-supplied URLs into the render call, including indirect paths where a library rasterizes SVG or a
   converter loads a page. Each render of attacker-influenced content is a candidate sink.

3. **Determine which capability the document reaches.** A remote fetch enables server-side request forgery to
   internal and cloud-metadata endpoints; a local-file scheme or an external entity enables file disclosure
   into the output; a redirect the renderer follows bypasses a naive host allowlist; embedded script enables
   reading the rendered page or exfiltrating it. Decide which of these the renderer leaves reachable.

4. **Follow whether the result returns or must be blind.** A fetched resource or a read file discloses when it
   lands in the generated document the attacker receives. When it does not, the realistic path is the blind
   one: a subresource or a callback to a host the tester controls that proves the render reached out even
   though nothing is echoed. Identify which applies here.

5. **Check the confinement that actually closes it.** The reliable controls are running the renderer with no
   network access or on an isolated egress-filtered network, blocking local-file and non-HTTP schemes,
   disabling external entity and remote-subresource loading, disabling script, and resolving and re-checking
   any user URL against an allowlist after following redirects. Determine which of these stands between the
   untrusted content and the renderer.

6. **Confirm and record.** Confirm by rendering a document that references a benign local marker or a host
   the tester controls on an isolated instance, and observing the marker in the output or the callback
   arriving. Kill the lead if the renderer resolves nothing external, blocks file schemes and external
   entities, runs with the network disabled or egress-filtered, and disables script, or if no untrusted
   content reaches a renderer. Record the input, the renderer and its capabilities, the capability reached,
   and whether disclosure was direct or out of band, or set a `kill_reason`.

## Where renderer abuse leaks

- **A renderer is a browser on the server.** It resolves links, images, redirects, and often script and file
  schemes, from inside the network, which is exactly the reach an untrusted document should not have.
- **SVG is active content.** An SVG can reference external resources, carry script, and pull in external
  entities, so accepting it as an image and rasterizing it server-side is accepting markup into a renderer.
- **Redirects bypass naive allowlists.** A URL that passes a host check can redirect to an internal address
  the renderer follows, so the allowlist must be re-checked after each redirect on the resolved address.
- **Blind is the common case.** Many renders never return the fetched bytes to the attacker, so a subresource
  or callback to a controlled host is how resolution is proven and data is exfiltrated.
- **Cloud metadata is the prize.** A renderer that fetches an arbitrary URL from inside a cloud instance can
  reach the metadata endpoint and its credentials unless egress is filtered.

## Worked example (a confirm and a kill)

> **Confirm.** An export feature renders user-supplied HTML into a PDF with a headless browser that has
> network access and follows links. A document referencing the cloud-metadata endpoint causes the renderer
> to fetch it and embed the response in the PDF the attacker downloads, on an isolated instance. **Confirmed**
> server-side request forgery through a renderer to cloud metadata, `high`, remediation = run the renderer
> with no network access or on an egress-filtered network that blocks link-local and internal ranges, block
> non-HTTP and file schemes, disable remote subresource and external-entity loading, and resolve any user URL
> against an allowlist after redirects.
>
> **Kill.** The same feature renders in a sandbox with the network disabled, file and non-HTTP schemes
> blocked, external entities and remote subresources disabled, and script off, and any user-supplied URL is
> fetched by a separate hardened client, not the renderer. A document referencing the metadata endpoint
> resolves nothing and no request leaves the sandbox. **Killed**, `kill_reason` = "renderer runs sandboxed
> with no network and no file-scheme or external-subresource resolution and script disabled; no document
> reference produces a fetch, a file read, or a callback."

## Rationalizations to reject

- *"It only accepts images."* -> An SVG is markup with external references and script; rasterizing it
  server-side hands it to a renderer exactly like HTML.
- *"We validate the URL host."* -> A validated host can redirect to an internal address the renderer follows;
  re-check the resolved address after every redirect, not just the first URL.
- *"The renderer returns only a PDF."* -> Then test the blind path: a subresource or callback to a controlled
  host proves the fetch and exfiltrates without the bytes being echoed.
- *"Script is disabled, so it is safe."* -> Disabling script closes one capability; remote fetches, file
  schemes, and external entities remain unless they are each disabled too.
- *"It runs in our cluster, not the internet."* -> That is the problem: from inside the network it reaches
  internal services and cloud metadata that an internet client never could.

## Executing this in practice

You need every server-side render of untrusted content, including SVG rasterization and URL-to-document
features, and the exact capabilities of each renderer: whether it fetches remote URLs, follows redirects,
resolves file schemes and external entities, and runs script, and whether it has network access. For each
sink, decide which capability remains reachable and whether the result returns directly or only out of band.
Reading the renderer configuration settles most leads; a local marker or a controlled-host callback on an
isolated instance settles the rest.

## Related

- `exploiting-ssrf-to-cloud-metadata` - a renderer that fetches an arbitrary URL is an SSRF primitive, so the
  reach and credential-theft analysis there applies directly to what the render can touch.
- `hunting-dns-rebinding-and-ssrf-pivots` - the network reachability and allowlist-bypass analysis for the
  renderer's fetches, including redirect and rebinding tricks, lives there.
- `hunting-xxe-and-xml-parser-trust` - SVG and document renderers parse XML, so external-entity file
  disclosure through the renderer shares that skill's parser analysis.
- `hunting-path-traversal-and-file-access` - a renderer that resolves a file scheme is another route to local
  file read that shares the containment reasoning.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the untrusted markup or URL, sink = the resolving
  renderer, evidence = the internal fetch, the disclosed local file in the output, or the out-of-band
  callback observed on an isolated instance.
