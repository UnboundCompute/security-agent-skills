---
name: hunting-server-side-template-injection
description: >-
  Hunt server-side template injection where untrusted input becomes part of a template that the engine
  compiles and evaluates, rather than data passed into a fixed precompiled template. Covers input
  concatenated into a template string, a user-chosen template name or path, and admin or content features
  that render user-authored templates, across engines like Jinja, Twig, Freemarker, Velocity, ERB, and
  Handlebars. The impact runs from expression evaluation and data disclosure up to remote code execution
  when the engine exposes object internals or a weak sandbox. Use when a template string or name is built
  from input, or a feature lets users supply template markup. The untrusted value reaching compilation is
  the source, the template engine eval is the sink, and expression evaluation escalating to disclosure or
  code execution is the bug.
license: MIT
---

# Hunting server-side template injection: when input becomes the template, not the data

Template engines are safe when untrusted input is passed as a bound variable into a fixed, precompiled
template: the value can only ever be data. Server-side template injection happens when input instead becomes
part of the template that the engine compiles, either concatenated into a template string, supplied as the
template name or path, or authored directly by a user in a feature that renders their markup. Now the engine
evaluates the attacker's template expressions. The floor is expression evaluation and data disclosure; the
ceiling, on many engines, is remote code execution by walking object attributes to reach a runtime, a
loader, or a process API, and by escaping whatever sandbox the engine offers. The distinction that governs
everything is whether input reaches compilation or only reaches rendering as data.

## When to use

- Code builds a template by concatenating or interpolating untrusted input before handing it to the engine.
- A template name, path, or layout is selected from user input.
- A feature lets users author templates or expressions: email or report templates, themes, or rules.

## Scope check

Test template injection only against applications you own or are authorized to assess, on non-production
infrastructure, because a confirmed case often reaches code execution. Prove the class with an inert
arithmetic or string expression before attempting anything further, and stay within the authorized instance.
If you can't name the authorization, stop.

## The loop

1. **Establish that input reaches compilation first.** Determine whether the untrusted value is compiled as
   part of a template (a dynamic template string, a user-chosen template name, or user-authored markup) or
   is only bound as a variable into a fixed template. This is the false-positive killer: a value passed as
   data into a precompiled template is not template injection no matter how it renders. Confirm the value
   reaches the engine's compile or eval step before proceeding.

2. **Identify the engine and its evaluation model.** Name the template engine, because the syntax, the
   reachable internals, and the sandbox differ sharply between them. Establish whether expressions can read
   arbitrary object attributes, call methods, or reach a runtime, and whether a sandbox or a restricted mode
   is configured.

3. **Confirm evaluation with an inert probe.** Establish the class with a benign expression whose evaluated
   result differs from its literal text, for example an arithmetic or string-concatenation expression in the
   engine's syntax. Evaluation of the expression, not reflection of the literal characters, is what proves
   injection and separates it from reflected XSS.

4. **Map the reachable capability.** From a confirmed expression, determine what the engine exposes: reading
   configuration and context variables, traversing object graphs to sensitive data, invoking methods, or
   reaching a loader or process API. This is where the impact is decided, from disclosure to code execution;
   assess it by reading what the engine and the template context make reachable, not by launching payloads.

5. **Check the sandbox and the input path.** Determine whether a configured sandbox actually blocks attribute
   access and method calls or is a known-porous one, and whether the template-name path is constrained to an
   allowlist of known templates rather than an attacker-chosen file. A strong sandbox plus data-only binding
   plus a fixed template set is what closes the class.

6. **Confirm and record.** Confirm by evaluating an inert expression through the sink on an authorized
   instance and, where the engine allows, demonstrating a bounded read of a non-sensitive context value.
   Kill the lead if input is only bound as data into a fixed template, if the template name is constrained
   to an allowlist, if the engine's sandbox provably blocks attribute and method access for the reachable
   context, or if the value is escaped as output rather than compiled. Record the input, the compilation
   sink, the engine, and the reachable capability.

## Where template injection leaks

- **Data binding is safe; string building is not.** The same engine is secure with a bound variable and
  injectable when input is concatenated into the template source.
- **Template names select attacker files.** A user-controlled template name or path can load an unintended
  or attacker-supplied template even when the values inside are bound safely.
- **The engine decides the ceiling.** Some engines evaluate only simple expressions; others expose object
  internals that reach a runtime, so the same primitive is disclosure on one and code execution on another.
- **Sandboxes vary and leak.** A restricted mode that forgets one attribute or one builtin is an escape;
  treat a sandbox as a control to verify, not a guarantee.
- **It hides in author features.** Email, report, theme, and rule builders that accept template markup are
  injection by design unless the expression surface is deliberately constrained.

## Worked example (a confirm and a kill)

> **Confirm.** A notification feature lets an admin write a message template that the server renders by
> concatenating the stored string into the engine and compiling it. An inert arithmetic expression in the
> engine's syntax evaluates to its computed value in the sent message, and the template context exposes
> object attributes that reach a loader. **Confirmed** server-side template injection with a path to code
> execution, `critical`, remediation = render user content as bound data in a fixed template, never compile
> user strings, and if authoring is required use a logic-less engine or an expression allowlist with no
> attribute or method access.
>
> **Kill.** The message body is passed as a bound variable into a precompiled, logic-less template, the
> template name is selected from a fixed server-side allowlist, and the engine runs in a restricted mode
> that blocks attribute and method access. An arithmetic expression renders as literal text. **Killed**,
> `kill_reason` = "input is bound as data into a fixed template, template names are allowlisted, and the
> engine cannot evaluate attacker expressions; no input reaches compilation."

## Rationalizations to reject

- *"We escape the output, so it is safe."* -> Output escaping stops XSS in the rendered result; it does
  nothing when the input is compiled as template source and evaluated first.
- *"Only admins can edit templates."* -> An authenticated author is still an attacker for this class, and
  author features reach code execution; constrain the expression surface regardless of who edits.
- *"The sandbox blocks dangerous calls."* -> Sandboxes for these engines are frequently escaped; verify the
  exact restricted mode against the reachable context rather than trusting it.
- *"It just reflects the input."* -> Reflection is XSS; evaluation of an expression is template injection.
  Prove which with an inert expression whose result differs from its text.
- *"The template name is internal."* -> If any request influences the name or path, it can select an
  unintended or attacker-controlled template; pin it to an allowlist.

## Executing this in practice

You need every place a template is compiled or selected, which of those take untrusted input as source
versus bound data, the engine in use, and whether a sandbox or restricted mode is configured. For each
compilation sink, decide whether input reaches it and what the engine and context make reachable. Reading the
call that builds and compiles the template shows whether input is source or data; an inert expression through
the sink shows whether the engine evaluates it.

## Related

- `hunting-expression-language-injection` - the closely related class where the evaluated language is an
  expression language embedded in a framework rather than a full template engine.
- `hunting-reflected-and-stored-xss` - the lesser sibling to rule out; reflection of literal input is XSS,
  evaluation of an expression is template injection.
- `hunting-os-command-injection` - a frequent escalation target once an engine exposes a process API through
  a reachable object graph.
- `adjudicating-taint-paths` - use it to prove the input reaches a compile or eval step and not just a bound
  render.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the untrusted value reaching template compilation,
  sink = the template engine eval, evidence = an inert expression evaluating to its computed result.
