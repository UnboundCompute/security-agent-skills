---
name: auditing-container-image-build-hardening
description: >-
  Audit container image build definitions (Dockerfile, containerfile, and the compose or run config that
  sets runtime flags) for an image that ships over-privileged or carrying a secret, after multi-stage
  discards and deploy-time overrides are accounted for. Covers an image that runs as root, a secret baked
  into a layer, remote content pulled unpinned or unverified, a mutable or untagged base, an over-broad
  copy that pulls in local secrets and history, and a dangerous runtime request such as privileged mode or
  a sensitive host mount. Use when reviewing the image build plane, not the deploy-time security context
  or the cloud resource definition. The build definition is the source, the shipped image or run config is
  the sink, and privilege or a secret that survives into the final image is the bug.
license: MIT
---

# Auditing container image build hardening: what survives into the shipped image

A secret used in a build stage that a multi-stage build discards is not in the shipped image, and a root
Dockerfile deployed under a non-root security context does not run as root. So the bug here is not an
instruction in isolation; it is what survives into the final image layer and the runtime request that
ships with it. This audit reads the build definition and asks, per concern, whether the image that ships
runs unprivileged, carries no secret in any layer, pulls only pinned and verified content, and requests
no dangerous runtime privilege. The checklist competitors flag the instruction; the finding that survives
is the one a later build stage or a deploy-time override does not neutralize. Stay on the image build
plane, and hand deploy-time privilege to the Kubernetes skill and cloud resources to the infrastructure
skill.

## When to use

- You have a Dockerfile or containerfile, or a compose or run config that sets container runtime flags.
- An instruction sets the user, copies files, fetches remote content, pins a base, or requests a privilege.
- You want to know what survives into the shipped image, not which instruction looks wrong in isolation.

## Scope check

Audit only build definitions for images you own or are authorized to assess, and never push or run a
built image to test a finding against shared infrastructure, adjudicate on the layers and the config. If
you can't name the authorization, stop.

## The loop

1. **Trace what reaches the final stage first.** Follow the build stages and note what the shipping stage
   actually contains: which stage the final image is built from, what a copy-from carries forward, and
   which runtime fields come from the deploy manifest rather than the build. A secret or a root default
   only matters if it survives into the shipped image or is not overridden at deploy; settle that first.

2. **Check the run-as user.** Look for no user directive, or a user set to root, so the final image
   defaults to the root account, then check whether the deploy-time security context or a non-root user in
   the base image overrides it. This is the image default; the Kubernetes skill owns the deploy-time context.

3. **Check for a baked secret.** Look for an environment or build argument carrying a token or password, a
   copy of a key, credentials, or an environment file, or a secret used in a run step that persists in that
   layer, then confirm the final stage carries it rather than a multi-stage build discarding it. Report only
   a secret present in the shipped layers; hand its liveness and blast-radius to the secret-reachability skill.

4. **Check remote content and the base pin.** Look for remote content fetched at build with no checksum or
   digest verification (a fetch-and-run, an add from a URL), and a base image on a mutable or untagged
   reference rather than a digest, then check whether a later step verifies the content or the deployment
   re-pins the image by digest.

5. **Check the copy scope and runtime request.** Look for an over-broad copy of the whole context with no
   ignore file, pulling in version-control history, local secrets, and build junk, and for a compose or run
   config requesting privileged mode, added capabilities, or a sensitive host mount. Resolve the host-mount
   and capability overlap by layer: the build or compose config is this skill, the pod spec is the Kubernetes skill.

6. **Confirm and record.** Confirm by showing the concern survives into the shipped image or the production
   run config. Kill the lead if a root default is overridden by a deploy-time non-root security context or a
   non-root base user, if a secret lives only in a discarded build stage the final copy-from does not carry,
   if remote content is verified by digest or checksum later, if a mutable tag is re-pinned by digest at
   deployment, if an over-broad copy is neutralized by an ignore file that excludes the sensitive paths, or
   if a privileged or host-mount request is confined to a local-development config not the production
   manifest. Record the instruction, what survives into the image, and the layer that carries it.

## Where container builds leak

- **The finding is what survives, not the instruction.** A discarded build stage or a deploy-time override
  can neutralize it; adjudicate the shipped image, not the line.
- **A root Dockerfile can be fixed at deploy.** A non-root security context in the manifest overrides the
  image default; check the runtime layer before calling a missing user a finding.
- **A multi-stage build discards intermediate secrets.** A secret in a build stage the final copy-from does
  not carry never ships; the final stage is what matters.
- **A mutable tag can be re-pinned downstream.** A latest or untagged base is a drift and integrity risk
  unless the deployment re-pins by digest; check where the image is actually resolved.
- **An over-broad copy is only a leak without an ignore file.** A copy of the whole context is neutralized
  when an ignore file excludes version control, secrets, and junk.

## Worked example (a confirm and a kill)

> **Confirm.** The final build stage sets a build argument holding an API key into an environment variable,
> and no deploy override removes it, so the shipped image layer exposes a live key in its history. A pull of
> the image reveals the key. **Confirmed** a secret baked into the shipped image layer, `high` rising with
> the key's privilege, remediation = pass the secret at runtime from a secret store, never bake it into a
> layer, and rotate the exposed key.
>
> **Kill.** A key is set in a build stage, but the final stage is built from a separate base and the
> copy-from carries only the compiled artifact, not the key, so the shipped image has no secret layer.
> **Killed**, `kill_reason` = "the secret lives only in a discarded build stage; the final copy-from does not
> carry it, so it is not in the shipped image."

## Rationalizations to reject

- *"The image has no user directive, so it runs as root."* -> Does the deploy security context or the base
  image set a non-root user? The image default can be overridden at runtime.
- *"There is a secret in a build argument."* -> Does the final stage carry it, or does a multi-stage build
  discard it? Only a secret in the shipped layers is the finding.
- *"The base is on the latest tag."* -> Does the deployment re-pin by digest? A mutable tag re-pinned at
  deploy is resolved to a fixed image.
- *"It copies the whole build context."* -> Is there an ignore file excluding secrets and version control?
  A scoped ignore neutralizes an over-broad copy.
- *"The config requests privileged mode."* -> Is that the production manifest or a local-development
  compose file? A dev-only privilege request is not what ships.

## Executing this in practice

You need the build definition and its stages, what the final stage is built from and what a copy-from
carries, the deploy-time fields that override image defaults, the remote-content fetches and base pins, the
copy scope and any ignore file, and the production run config. For each concern, decide what survives into
the shipped image or the production runtime. Reading the instruction tells you what the build does; tracing
the final stage and the deploy overrides tells you what actually ships.

## Related

- `auditing-kubernetes-workload-and-rbac-hardening` - the mutual FP-killer: a root image here is killed by a
  locked-down deploy-time security context there, and that context is aggravated by a root image here.
- `auditing-infrastructure-as-code-exposures` - the sibling for cloud provider resource state; distinct from
  the image build plane this skill audits.
- `hunting-non-human-identity-and-secret-reachability` - takes the liveness and blast-radius of a secret this
  skill finds baked into an image layer.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the build definition, sink = the shipped image or run
  config, evidence = the privilege or secret that survives into the final image layer.
