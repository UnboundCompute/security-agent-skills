---
name: hunting-expression-language-injection
description: >-
  Hunt expression-language injection where untrusted input reaches a server-side expression evaluator:
  Spring Expression Language, OGNL, MVEL, JEXL, a Jakarta or JSP EL context, or a rules engine that
  evaluates strings. Covers request data flowing into an expression compiled and evaluated at runtime,
  where the language exposes type access, method calls, or a runtime handle that reaches command execution.
  Use when the application evaluates expressions built from or influenced by untrusted input rather than
  from fixed developer-authored strings. The untrusted value that becomes part of an evaluated expression
  is the source, the expression evaluator is the sink, and the reachable path from evaluation to a runtime
  or reflection call is the bug.
license: MIT
---

# Hunting expression-language injection: when a string is evaluated as a program

Expression languages exist to let configuration, templates, and rules evaluate small snippets at runtime,
Spring Expression Language, OGNL, MVEL, JEXL, and the Jakarta and JSP EL contexts among them. Most expose
far more than arithmetic: type resolution, method invocation, and a handle to the runtime. When untrusted
input is concatenated into, or supplied as, an expression that one of these evaluators compiles, the
attacker's text is executed with the evaluator's full power, which typically reaches process execution or
reflection. This class caused some of the largest breaches on record. You find it by locating every
runtime expression evaluation and asking whether untrusted input shapes the expression rather than only
supplying a bound variable.

## When to use

- The application evaluates expressions at runtime through an EL, OGNL, MVEL, JEXL, or a rules engine.
- Untrusted input is concatenated into an expression string or supplied as an expression to evaluate.
- A framework feature (data binding, templating, routing, error pages) evaluates attacker-reachable EL.

## Scope check

Test expression evaluation only against applications you own or are authorized to assess, on
non-production infrastructure. A confirming expression runs on the target and typically reaches code
execution, so treat every proof as a live intrusion inside the authorized scope. If you can't name the
authorization, stop.

## The loop

1. **Establish the expression evaluators first.** Inventory every place a string is evaluated as an
   expression: explicit calls to an expression parser, template engines backed by an EL, framework
   features that evaluate EL (data binding, view resolution, error templates, routing rules), and rules
   engines. This is the false-positive killer: an evaluator fed only fixed developer strings with
   untrusted data passed as bound variables is not injectable. Name the evaluators first.

2. **Separate the expression from its variables.** For each evaluator, determine what is the expression
   (executed as code) and what is a variable (a bound value). Untrusted data as a bound variable is safe;
   untrusted data that becomes part of the expression text is the vulnerability. This distinction is the
   entire adjudication.

3. **Trace untrusted input into the expression text.** Follow request parameters, headers, path segments,
   and stored values into any expression that is concatenated, formatted, or supplied wholesale to the
   evaluator. Framework indirection matters here: a value bound to a field that is later evaluated as EL,
   or a message rendered through an EL template, is a real path even without an explicit parser call.

4. **Confirm the language reaches a dangerous capability.** These languages typically expose type access
   and method invocation, so an injected expression can reach a runtime exec, a reflective load, or a
   file operation. Confirm the specific evaluator and context permit type or method access rather than a
   locked-down subset, because some contexts restrict the available surface.

5. **Check the defenses that actually stop it.** The durable fix is to never build the expression from
   untrusted input: keep expressions as fixed developer strings and pass untrusted data only as bound
   variables. A restricted evaluation context, a sandboxed EL, or a parser configured without type access
   also reduces the surface. A denylist of keywords does not, because the languages have many equivalent
   forms. Determine which stands here.

6. **Confirm and record.** Confirm by supplying an in-scope expression that proves evaluation of attacker
   text, an arithmetic identity that changes the response or an out-of-band signal, without a destructive
   command. Kill the lead if untrusted input only ever reaches the evaluator as a bound variable, if the
   expression text is entirely developer-authored, or if the evaluation context is sandboxed without type
   or method access. Record the source, the evaluator, the expression-versus-variable split, and the
   capability reached.

## Where expression-language injection leaks

- **Data binding evaluates field paths as expressions.** A framework that binds request keys to object
  paths through an EL can evaluate an attacker-supplied path, not just set a value.
- **Error and message templates run EL.** A user-influenced value rendered through an EL-backed template
  is evaluated, so a reflected message can become execution.
- **The language exposes the runtime.** Type access and method calls mean an injected expression reaches
  exec or reflection; it is rarely limited to arithmetic.
- **A keyword denylist is bypassable.** These languages offer many ways to reach the same capability;
  blocking a few tokens does not hold.
- **A bound variable is safe, an expression fragment is not.** The whole risk is whether untrusted data
  becomes the code or the value; conflating the two is the common mistake.

## Worked example (a confirm and a kill)

> **Confirm.** A routing rule evaluates an expression that includes a request header to decide a
> redirect target, concatenating the header into the expression text. A crafted header supplies an
> expression that resolves a type and invokes a method, and a benign arithmetic-and-callback payload
> confirms the attacker text is evaluated with type access. **Confirmed** expression-language injection to
> remote code execution, `critical`, remediation = remove untrusted input from the expression text, keep
> the rule as a fixed expression and pass the header only as a bound variable, and evaluate in a context
> without type access.
>
> **Kill.** Every evaluator receives a fixed developer-authored expression, and all untrusted values reach
> it only as named bound variables through the evaluation context; the one templating path uses a
> sandboxed context without type or method access. No crafted input becomes expression text. **Killed**,
> `kill_reason` = "expressions are developer-authored and untrusted data enters only as bound variables in
> a sandboxed context; no attacker text is evaluated as code."

## Rationalizations to reject

- *"We only pass user data as a variable."* → Then confirm it. If any path concatenates it into the
  expression text, that path is the injection.
- *"The expression is in a config file, not user input."* → The question is whether user input reaches the
  evaluated string at runtime, through binding or templating, not where the base expression lives.
- *"We strip dangerous keywords."* → These languages have many equivalent constructs; a denylist is
  bypassed. Keep untrusted data out of the expression instead.
- *"It only evaluates arithmetic."* → Confirm the context restricts type and method access; most EL
  contexts do not, and then it reaches the runtime.
- *"It is an internal rules engine."* → If untrusted input reaches the rule text, internal placement does
  not change that it evaluates attacker code.

## Executing this in practice

You need every runtime expression evaluation, the split between expression text and bound variables at
each, the untrusted inputs that reach the expression text, and the evaluation context's capabilities. For
each evaluator, ask whether untrusted data becomes code or stays a value, and whether type and method
access are available. Reading the evaluator and its context shows intent; a benign arithmetic-and-callback
proof shows attacker text is evaluated.

## Related

- `hunting-template-injection-sandbox-escapes` - server-side template injection is the templating-engine
  cousin; both evaluate attacker text, and templates often sit on top of an EL.
- `hunting-orm-and-query-builder-injection` - the same value-versus-structure distinction applied to
  queries rather than expressions.
- `finding-fail-open-flaws` - a restricted evaluation context that silently falls back to a full one is a
  fail-open path worth checking alongside this.
- `adjudicating-taint-paths` - use it to confirm untrusted input reaches the expression text through data
  binding and templating indirection.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the untrusted value that becomes expression text,
  sink = the expression evaluator, evidence = evaluation reaching a runtime or reflection capability.
