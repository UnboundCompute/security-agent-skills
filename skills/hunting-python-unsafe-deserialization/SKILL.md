---
name: hunting-python-unsafe-deserialization
description: >-
  Hunt Python deserialization that executes attacker code: untrusted input reaching pickle.loads, an
  unsafe YAML load, marshal, jsonpickle, dill, or a numpy or pandas loader that unpickles, where the
  format supports arbitrary object construction through __reduce__ or a tag. Covers pickled data in
  requests, cookies, caches, message queues, and model or dataframe files, and YAML documents that
  instantiate arbitrary Python objects. Use when a service loads serialized Python objects it did not
  produce with a loader that reconstructs arbitrary types rather than parsing data only. The untrusted
  serialized blob is the source, the reconstructing loader is the sink, and the __reduce__ or object tag
  that runs a callable during load is the bug.
license: MIT
---

# Hunting Python unsafe deserialization: when loading data calls a function

Several Python serialization formats are not data formats at all; they are programs. `pickle` invokes
`__reduce__` during load, which names a callable and its arguments and runs them, so a pickled blob can
call any importable function on reconstruction. YAML's full loader instantiates arbitrary Python objects
from tags. `marshal`, `dill`, `jsonpickle`, and the model and dataframe loaders that wrap pickle inherit
the same property. The vulnerability is not a bug in these libraries; it is using them on data an
attacker controls. You find it by locating every loader that reconstructs objects rather than parsing
data, and asking whether untrusted bytes reach it.

## When to use

- A service loads serialized Python objects from a request, a cookie, a cache, a queue, or a file.
- The loader in use reconstructs arbitrary objects: pickle, unsafe YAML, marshal, dill, or jsonpickle.
- A model, dataframe, or checkpoint file from an untrusted source is loaded through a pickling loader.

## Scope check

Test deserialization only against services you own or are authorized to assess, on non-production data. A
confirming payload runs code on load, so treat every proof as a live intrusion inside the authorized
scope. If you can't name the authorization, stop.

## The loop

1. **Establish the reconstructing loaders first.** Inventory every call that reconstructs objects rather
   than parsing data: `pickle.loads`/`load`, `yaml.load` without a safe loader, `marshal.loads`, `dill`,
   `jsonpickle.decode`, and model or dataframe loaders that unpickle internally. This is the false-positive
   killer: a `json.loads` or a `yaml.safe_load` parses data and cannot construct arbitrary objects. Name
   the reconstructing sinks first.

2. **Trace untrusted bytes into each loader.** Follow request bodies, cookies, headers, cache entries,
   queue messages, and uploaded or downloaded files into the loader. A model checkpoint or a dataframe
   pulled from an untrusted registry, a shared bucket, or a user upload is attacker-influenced the moment
   its provenance is not controlled. Confirm the bytes crossing the boundary are not internal-only.

3. **Confirm the format supports construction, not just parsing.** Pickle and unsafe YAML run a callable
   during load; safe YAML and JSON do not. For a wrapped loader, confirm it reaches pickle underneath. A
   loader restricted to a safe subset, a strict YAML loader, or a signed-and-verified blob changes the
   answer, so read the exact call.

4. **Check any allowlist or unpickler restriction.** A custom `Unpickler` that overrides `find_class` to
   an allowlist, a safe YAML loader, or a signature check on the blob before load each blunt the vector.
   The absence of any restriction means `__reduce__` or a YAML tag can name any importable callable.
   Determine whether a real restriction stands at the sink.

5. **Trace the reconstruction to a callable.** Confirm the format's construction step reaches an
   attacker-chosen callable with attacker-chosen arguments: a `__reduce__` naming a system call, a YAML
   tag instantiating a class with a dangerous initializer, or a checkpoint whose custom reducer runs on
   load. The callable is the terminal sink.

6. **Confirm and record.** Confirm by supplying a benign in-scope blob whose reduce or tag performs an
   out-of-band signal, proving the loader runs the callable, without a destructive payload. Kill the lead
   if no untrusted data reaches a reconstructing loader, if the loader is a safe data parser, if an
   unpickler allowlist or a verified signature constrains the blob, or if the format cannot construct
   objects. Record the source, the loader, the construction mechanism, and the callable.

## Where Python deserialization leaks

- **pickle is a code format, not a data format.** `__reduce__` runs a callable on load; there is no safe
  way to unpickle untrusted data without an allowlisting unpickler.
- **`yaml.load` without a safe loader instantiates objects.** A YAML tag constructs arbitrary Python
  classes; only `safe_load` or a safe loader parses data alone.
- **Model and dataframe files carry pickle.** A checkpoint or serialized dataframe from an untrusted source
  unpickles on load, so a poisoned artifact is code execution disguised as a data file.
- **Signed does not mean safe unless verified before load.** A signature checked after loading is useless;
  the callable already ran.
- **A cache or queue blob is untrusted on read.** Even if the app wrote the first copy, an attacker who can
  write the store controls what comes back.

## Worked example (a confirm and a kill)

> **Confirm.** A worker loads task payloads from a shared cache with `pickle.loads` and no unpickler
> restriction. An attacker who can write a cache key supplies a pickle whose `__reduce__` names a callable
> that performs an out-of-band request. A benign marker payload confirms the callable runs on load.
> **Confirmed** untrusted deserialization to remote code execution, `critical`, remediation = replace the
> pickle transport with a data-only format such as JSON or a schema codec, and if pickle is unavoidable,
> load through an `Unpickler` whose `find_class` allowlists only the expected classes.
>
> **Kill.** The service loads task payloads only with `json.loads` and loads YAML config only with
> `yaml.safe_load`; the one model file it reads is fetched from a controlled registry over an integrity
> check and loaded through an allowlisting unpickler. No untrusted blob reaches a reconstructing loader.
> **Killed**, `kill_reason` = "untrusted inputs parsed as data only (json, safe_load); the sole pickle load
> is integrity-verified and allowlist-restricted, so no attacker callable runs on load."

## Rationalizations to reject

- *"It is just our internal task format."* → A cache, queue, or file an attacker can write is untrusted on
  read; internal use is not authentication.
- *"We use YAML, which is a config format."* → `yaml.load` without a safe loader instantiates arbitrary
  objects; only `safe_load` parses data. Check which one.
- *"It is only a model file."* → Model and checkpoint files unpickle on load; a poisoned artifact runs code.
- *"The blob is signed."* → Only safe if the signature is verified before the load call. Verify first, then
  load, or the callable runs regardless.
- *"We validate the object after loading."* → `__reduce__` and YAML construction already executed. Post-load
  validation is too late.

## Executing this in practice

You need every reconstructing loader, the untrusted inputs that reach each, the exact loader variant
(reconstructing versus data-only), and any unpickler allowlist or pre-load signature check. For each
loader, ask whether attacker bytes arrive, whether the format constructs objects, and whether a
restriction stands between. Reading the loader call shows the intent; a benign out-of-band proof shows the
callable actually runs.

## Related

- `hunting-java-deserialization-gadget-chains` - the JVM sibling; both run code on reconstruction, one via
  gadgets, one via `__reduce__`.
- `auditing-ml-model-supply-chain` - model and checkpoint files are a primary untrusted source here; that
  skill governs where the artifact comes from.
- `testing-rag-and-memory-poisoning` - another path by which an untrusted artifact enters an AI pipeline and
  is later loaded.
- `adjudicating-taint-paths` - use it to confirm untrusted bytes reach a reconstructing loader through
  wrappers and framework indirection.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the untrusted serialized blob, sink = the
  reconstructing loader, evidence = the reduce or tag running an attacker callable on load.
