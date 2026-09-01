---
name: auditing-ml-inference-endpoint-abuse
description: >-
  Audit a hosted model inference endpoint for abuse that costs money or steals the asset: an unauthenticated or
  weakly keyed endpoint anyone can call, no per-caller rate or spend limit so a caller runs up unbounded
  inference cost, model extraction where systematic queries reconstruct the model or its decision boundary,
  membership and training-data inference that recovers whether a record was in the training set, and a response
  that returns full probabilities or embeddings that make extraction and inversion easier. Covers deployed
  prediction and embedding endpoints for classifiers, recommenders, and other served models, distinct from
  loading an untrusted model or serving a chat assistant. Use when a model is exposed as a callable endpoint and
  its cost, confidentiality, and integrity are the boundary. The unbounded or systematic query stream is the
  source, the run-up cost or reconstructed model or training data is the sink, and the missing auth, rate/spend
  bound, or over-informative response is the bug.
license: MIT
---

# Auditing ML inference endpoint abuse: a served model is an asset and a meter, both attackable

A model exposed as an inference endpoint is two things worth attacking at once: a metered resource that costs
money per call, and a confidential asset that queries can reconstruct. The abuses follow from that. If the
endpoint is unauthenticated or weakly keyed, anyone can call it. If there is no per-caller rate or spend limit,
a caller runs up unbounded inference cost, a denial-of-wallet against expensive model serving. Beyond cost, the
model itself leaks to a determined querent: systematic queries reconstruct the model or its decision boundary
(model extraction), and carefully chosen queries recover whether a specific record was in the training set or
reconstruct sensitive training data (membership and inversion inference). Over-informative responses, full
class probabilities, raw embeddings, confidence vectors, make both extraction and inversion far easier. The
audit treats the endpoint as a cost meter and a confidential asset and checks the controls on both. You audit
this by calling the endpoint as an attacker would: unauthenticated, at volume, and systematically.

## When to use

- A model is deployed as a callable prediction or embedding endpoint (classifier, recommender, scorer).
- The endpoint may be unauthenticated, weakly keyed, or lack per-caller rate and spend limits.
- Responses may return full probabilities or embeddings, and the model or its training data is confidential.

## Scope check

Test inference endpoints only on models and services you own or are authorized to assess, on non-production
deployments. Volume and systematic querying incur real cost and exercise a real confidentiality boundary, so
use a test deployment and never run up spend or extract from a model that is not yours. If you can't name the
authorization, stop.

## The loop

1. **Establish the intended access, cost, and confidentiality bounds first.** Name who may call the endpoint,
   the per-caller rate and spend limit, and what the response should reveal. This is the false-positive killer:
   an endpoint that authenticates callers, enforces per-caller rate and spend limits, and returns only the
   minimal answer (a label, a top-k, a coarse score) is behaving correctly. Name the intended bounds, then test
   each.

2. **Check authentication and per-caller identity.** Confirm the endpoint requires authentication and attributes
   each call to a caller, rather than being open or sharing one key. An unauthenticated or shared-key endpoint
   has no one to rate-limit or bill, so every abuse below is unbounded.

3. **Test rate and spend bounds (denial-of-wallet).** Call the endpoint at volume and confirm a per-caller rate
   limit and a spend or quota cap stop it. Where inference is expensive (large models, GPU, per-token billing),
   confirm the cost of a single caller's traffic is bounded. No per-caller ceiling means one caller can run up
   arbitrary cost.

4. **Test model-extraction exposure.** Assess whether systematic queries can reconstruct the model or its
   decision boundary: whether query volume is bounded, whether responses are informative enough to train a
   surrogate, and whether extraction-shaped query patterns are detected. An endpoint that answers unlimited
   fine-grained queries hands over the model over time.

5. **Test membership, inversion, and response verbosity.** Check whether the response returns more than needed,
   full probability vectors, raw embeddings, exact confidences, and whether that enables membership inference
   (was this record trained on) or model inversion (reconstructing sensitive inputs). Confirm the response is
   minimized to the decision the caller needs.

6. **Confirm and record.** Confirm by abusing the endpoint on a test deployment: calling it unauthenticated,
   running unbounded cost, training a surrogate from its answers, or recovering membership from verbose
   responses, without running up real spend or extracting a real model. Kill the lead if the endpoint
   authenticates callers, enforces per-caller rate and spend limits, and returns minimized responses that resist
   extraction and inversion. Record the abusive query stream, the cost or reconstruction sink, and the missing
   auth, bound, or over-informative response.

## Where the endpoint leaks

- **Open or shared-key access.** An unauthenticated endpoint or one shared key means no per-caller identity to
  limit or bill, so cost and extraction are unbounded.
- **No per-caller cost ceiling.** Missing rate and spend limits let one caller run up arbitrary inference cost,
  denial-of-wallet against expensive serving.
- **Unlimited fine-grained queries.** An endpoint that answers systematic queries without volume bounds or
  pattern detection lets a caller reconstruct the model.
- **Over-informative responses.** Full probability vectors, raw embeddings, and exact confidences make
  extraction, membership inference, and inversion far easier than a minimal answer would.
- **No abuse detection.** Extraction and inference attacks have distinctive query patterns; an endpoint that
  never looks for them cannot slow them down.

## Worked example (a confirm and a kill)

> **Confirm.** A classification endpoint requires only a shared API key embedded in the web client and returns
> the full probability vector for every class. A single extracted key drives high-volume systematic queries with
> no per-caller limit, running up substantial GPU inference cost and, from the fine-grained probabilities,
> training a surrogate that reproduces the model's decisions. **Confirmed** denial-of-wallet plus model
> extraction via shared key and verbose responses, `high`, remediation = authenticate and attribute each caller,
> enforce per-caller rate and spend limits, and return only the top label or a coarse score rather than full
> probabilities.
>
> **Kill.** The endpoint authenticates each caller and attributes calls to them, enforces a per-caller rate and
> spend limit that bounds cost, returns only the minimal decision (top label or coarse score, no raw embeddings
> or full vector), and flags extraction-shaped query patterns. High-volume, systematic, and verbose-response
> abuse are all bounded. **Killed**, `kill_reason` = "per-caller auth, rate and spend limits, and minimized
> responses; neither unbounded cost nor model or training-data reconstruction is reachable."

## Rationalizations to reject

- *"The key is in the client, so callers are known."* → A client-embedded key is extractable and shared; treat
  the endpoint as open unless each caller authenticates and is attributed.
- *"Inference is cheap."* → Per call, maybe; at unbounded volume on GPU or per-token billing it is not. Confirm a
  per-caller spend ceiling exists.
- *"You cannot steal a model through an API."* → Systematic queries reconstruct the decision boundary over time;
  bound query volume and minimize responses to make it impractical.
- *"We return probabilities because clients want them."* → Full vectors and embeddings enable extraction and
  inversion; return the minimal answer the client actually needs.
- *"No one is attacking it."* → Extraction and membership inference have distinctive patterns; without detection
  you would not know, and the cost accrues regardless.

## Executing this in practice

You need the endpoint's authentication, its per-caller rate and spend limits, the verbosity of its responses,
and any abuse detection. Call it unauthenticated, at volume, and with systematic and membership-probing query
sets on a test deployment, watching cost and what the responses reveal. Reading the serving configuration shows
the intended bounds; unbounded cost or a working surrogate shows whether they hold.

## Related

- `auditing-ml-model-supply-chain` - the load-time trust of the model artifact; this skill is the run-time abuse
  of the same model once it is served.
- `reviewing-rate-limiting-and-abuse-controls` - the general rate and abuse-control discipline this applies to
  the specific economics of model inference.
- `auditing-ai-agent-permissions` - denial-of-wallet through an agent's tool use is the agent-side version of
  the cost abuse here.
- `hunting-code-interpreter-and-tool-sandbox-escape` - the execution side of the AI surface; serving abuse and
  sandbox escape are the two halves of an exposed model runtime.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the unbounded or systematic query stream, sink = the
  run-up cost or reconstructed model or training data, evidence = the missing auth, bound, or over-informative
  response.
