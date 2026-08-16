---
name: auditing-ml-model-supply-chain
description: >-
  Audit the machine-learning models you load as untrusted code, not just data.
  Covers deserialization RCE from unsafe checkpoint formats (a model file that runs
  code on load), poisoned or backdoored weights, tampered or trojaned models pulled
  from a public hub, name and version confusion for model artifacts, and skipped
  integrity verification. Use when adding a model, checkpoint, or weights file to a
  pipeline, reviewing where models are loaded, or vetting a third-party model. A
  model file is executable input until you prove otherwise.
license: MIT
---

# Auditing the ML model supply chain: a model file is code you run

A model file is usually treated as inert data, a bag of weights. Many formats are
not: loading one can execute arbitrary code embedded in the file, and even a
pure-weights model can carry a backdoor that changes behavior on a trigger. The
moment your pipeline loads a model someone else produced, that model is untrusted
code and untrusted logic entering your system, on the training host, the inference
server, or a developer's laptop.

## When to use

- You are adding a model, checkpoint, or weights file to a training or inference
  pipeline.
- You are reviewing where and how models are loaded, and from where.
- You are vetting a third-party or publicly-hosted model before you trust it.

## Scope check

Audit models and pipelines you own or are authorized to test. Do not load or execute
untrusted model files outside a contained environment. If you can't name the
authorization, stop.

## The loop

1. **Inventory every model load path and its format.** List where the system loads a
   model, checkpoint, or weights file, who produced each one, and the serialization
   format. Formats that can reconstruct arbitrary objects execute code on load;
   formats that carry only tensors are safer. Mark each load site by format risk.

2. **Check for code execution on load (the RCE leg).** For any load path using a
   format that can rebuild arbitrary objects, a malicious file runs code the instant
   it is loaded, before any inference. Confirm whether untrusted files reach that
   path, and whether a tensor-only loader or a restricted deserializer is enforced.
   An unrestricted load of an externally-sourced file is remote code execution.

3. **Trace provenance and integrity.** Where did each model come from, and is it
   verified? Check for a pinned hash or signature, a known publisher, and a fixed
   version. A model pulled by mutable name from a public hub without hash pinning is
   a rug-pull and name-confusion channel: the artifact can be swapped, or a lookalike
   name served.

4. **Test for name and dependency confusion.** Is the model referenced by a bare or
   ambiguous name a public source could satisfy, or does loading it pull auxiliary
   code (custom loading scripts, a remote-code flag, tokenizer or pipeline scripts
   that run)? A model that ships and runs its own loading code is code execution
   disguised as a download.

5. **Consider weight-level tampering (backdoors).** Even a safe-format,
   integrity-verified model can be trojaned: trained to misbehave on a trigger input
   while scoring normally on benchmarks. You cannot fully verify this by loading, but
   you can bound exposure through provenance (trusted producer, reproducible
   training) and by treating unverified models as untrusted logic behind a boundary.

6. **Rate and record.** Code execution on load from an untrusted file is critical. A
   poisoned-weights model in a low-trust decision is lower but real. Record confirmed
   unsafe load paths and paths that enforce safe formats plus pinned, verified
   provenance (killed) in the schema.

## Reading model risk honestly

- **A load is an execution for many formats.** "We just load the weights" can mean
  "we run whatever the file says." Check the format first.
- **Popularity is not provenance.** A widely-downloaded artifact can still be swapped
  or lookalike-named. Pin and verify.
- **Remote-code load flags are opt-in RCE.** A convenience flag that lets a model run
  custom code on load is the whole vulnerability.
- **Weight backdoors survive integrity checks.** A hash proves the file is unchanged,
  not that it is honest. Provenance is the control.

## Worked example (a confirm and a kill)

> **Confirm.** An inference service downloads a model by mutable name from a public
> hub with a flag that permits the model to execute its own loading code, no hash
> pinning. An attacker publishes a trojaned version (or a lookalike name); the next
> deploy runs the attacker's code on the inference host on load. **Confirmed**
> model-supply-chain RCE, `critical`, remediation = pin by hash, disable remote-code
> execution on load, restrict to a tensor-only format, and vet the publisher.
>
> **Kill.** A pipeline loads only tensor-only-format weights (no code execution
> possible on load), by pinned hash, from an internal registry, with remote-code
> flags disabled and the loader restricted. A swapped file fails the hash and no code
> can run on load. **Killed**, `kill_reason` = "tensor-only load with no code
> execution, hash-pinned from a trusted registry, remote-code flags off."

## Rationalizations to reject

- *"It's just a weights file."* → Many formats execute code on load. Check the format
  before you call it data.
- *"It's from a popular repo."* → Popular is not pinned. The artifact behind a name
  can change or be impersonated.
- *"We need the convenience flag to load it."* → That flag is opt-in remote code
  execution. Find a safe-format alternative.
- *"The hash matches, so it's safe."* → The hash proves it is unchanged, not that it
  is not backdoored. Provenance still matters.

## Executing this in practice

You need every model load site and its format, the provenance and pinning of each
artifact, and whether the loader can execute embedded or remote code. Any environment
where you can inspect the load configuration and the artifact source works; the
format-risk classification and the provenance check are the method.

## Related

- `hunting-supply-chain-risks` - the same confusion, pinning, and rug-pull discipline
  for code dependencies.
- `auditing-ai-agent-permissions` - bounding what a loaded model or its code can reach
  at run time.
- `adjudicating-taint-paths` - tracing whether an untrusted file reaches a
  code-executing load.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the untrusted model
  artifact, sink = the load call that executes it.
