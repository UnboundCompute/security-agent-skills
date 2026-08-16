---
name: reviewing-ai-generated-code
description: >-
  Security-review discipline for code a language model wrote or completed: the
  failure patterns that show up more often in generated code and the review method
  that catches them. Covers hallucinated and confusable dependencies, insecure
  defaults and missing validation carried from training data, propagated vulnerable
  patterns, over-broad or fabricated permissions, and plausible-looking code that
  does not do what it claims. Use when reviewing an AI-authored change, an
  assistant's suggestion, or a large generated diff. Fluent is not correct.
license: MIT
---

# Reviewing AI-generated code: fluent is not correct

Model-written code reads well, which is exactly the risk: it is optimized for
plausibility, and a reviewer's guard drops when the code is clean and confident. The
security failures cluster in predictable places, dependencies that may not exist or
may be attacker-registered, defaults copied from insecure examples, validation
quietly omitted, and logic that looks right but is not. Reviewing it means aiming at
those clusters, not skimming for style.

## When to use

- You are reviewing an AI-authored change, an assistant's suggestion, or a large
  generated diff.
- Generated infrastructure, config, or access-control code is entering the codebase.
- You are setting a review bar for machine-assisted contributions.

## Scope check

Review code for projects you own or contribute to with authorization. If you can't
name the authorization, stop.

## The loop

1. **Verify every dependency the code introduces.** For each package the change adds,
   confirm it exists, is the established package (not a lookalike or a name the model
   may have invented), and is the one you intend. A hallucinated package name an
   attacker later registers turns "the model suggested it" into installed attacker
   code. Do not let a plausible import in unverified.

2. **Check the defaults and the omissions.** Generated code tends to reproduce the
   most common pattern, which is often the insecure-by-default one: permissive
   cross-origin rules, disabled verification, a broad permission, a missing auth
   check, a hardcoded or example secret, an unparameterized query. Read for what
   should be there and is not, not only for what is.

3. **Test the claim against the behavior.** Model code often states an intent in a
   comment or a name the code does not fulfill: a function named validate that does
   not reject, a check that is computed and then ignored, error handling that
   swallows and continues. Confirm the security-relevant logic actually does what its
   surface promises.

4. **Look for propagated vulnerable patterns.** If the prompt or the surrounding code
   contained an insecure idiom, the model likely extended it consistently. One
   confirmed weak pattern is a reason to sweep the whole diff for its siblings, the
   same way you would hunt variants of any bug.

5. **Check the permissions and scope the code assigns.** Generated infrastructure,
   config, and access code frequently over-grants: a wildcard role, a public bucket,
   an all-origins rule, a token with more scope than used. Treat every permission the
   code sets as a claim to minimize, not to accept.

6. **Adjudicate and record.** Confirm each issue against the real behavior and the
   real dependency, exactly as you would a hand-written finding; fluency is not
   evidence. Record confirmed issues with the minimal fix, and note patterns worth a
   broader sweep, in the schema.

## Where generated code fails

- **It optimizes for plausible, not correct.** The failure mode is confident
  wrongness, which defeats a skim.
- **Dependencies are unverified input.** A suggested package name is a claim until
  you confirm it resolves to the real thing.
- **Insecure defaults are the training-data mean.** The most common example online
  becomes the generated default.
- **Names lie more than usual.** A reassuring function name is not evidence the
  function is safe.

## Worked example (a confirm and a kill)

> **Confirm.** A generated data-access change adds an import for a package whose name
> is a near-match to a popular one but is not the established package, and configures a
> client with certificate verification disabled "for simplicity." The import is
> unverified and the default is insecure. **Confirmed**, `high`, remediation = replace
> with the verified real dependency, enable verification, and sweep the diff for the
> same disabled-verification pattern.
>
> **Kill.** A generated function adds input parsing; review confirms every dependency
> it uses is the intended, established package, the query is parameterized, validation
> rejects the documented bad inputs (tested), and no permission is widened. It does
> what its name claims. **Killed**, `kill_reason` = "dependencies verified, validation
> confirmed by test, no insecure default or over-grant introduced."

## Rationalizations to reject

- *"The model is good, the code looks clean."* → Clean is the risk. It is optimized
  for looking right. Verify behavior.
- *"It imported a package, so the package exists."* → Not necessarily, and not
  necessarily the real one. Confirm the name.
- *"The comment says it validates."* → The comment is generated too. Test that the
  code rejects what it should.
- *"It compiled and the tests passed."* → Existing tests rarely cover the security
  default or the missing check. Read for the omission.

## Executing this in practice

You need the generated diff, the ability to resolve and verify each new dependency
against a known-good source, and a way to test that the security-relevant logic
behaves as named. A call graph over the change helps you find every site of a
propagated pattern; the dependency verification and the claim-versus-behavior check
are the method.

## Related

- `hunting-supply-chain-risks` - the hallucinated-dependency and slopsquatting risk
  this review catches at the source.
- `hunting-bug-variants` - sweeping the diff for siblings of one confirmed weak
  pattern.
- `adjudicating-taint-paths` - confirming whether a missing validation is actually
  reachable by untrusted input.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the generated construct
  (dependency, default, missing check), sink = the behavior it enables.
