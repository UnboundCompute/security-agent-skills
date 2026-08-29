---
name: auditing-container-image-provenance
description: >-
  Audit how a cluster decides which container images to trust and run: an image referenced by a mutable tag
  rather than a content digest, a workload pulling from a registry that admits unsigned or unverified images,
  a signature or attestation policy that is configured but not enforced at admission, and a base image or
  layer whose origin the pipeline never verified. Covers Kubernetes and container platforms where the image
  a workload runs is the code that runs, and where tag mutability, signing, and provenance decide whether it
  is the intended artifact. Use when workloads pull images whose signing and provenance are not enforced end
  to end. The unverified image reference is the source, the running container is the sink, and the code that
  runs without proven provenance is the bug.
license: MIT
---

# Auditing container image provenance: when the tag is not the artifact

The image a workload runs is the code it runs, and a container reference is only a promise about which bytes
those are. A mutable tag can point at different content over time, so a workload pinned to a tag runs whatever
that tag resolves to at pull time, not the artifact that was reviewed. Signing and provenance attestation
exist to close that gap, but only if they are enforced at admission rather than merely produced in the
pipeline: a signature nobody verifies is decoration, and an attestation the cluster does not require proves
nothing to the cluster. The failure is quiet because the workload starts and runs; the question is whether
what it runs is provably the artifact you intended. You audit this by tracing each workload's image reference
back to a verified digest and confirming the cluster refuses anything unproven.

## When to use

- Workloads reference images by mutable tags rather than immutable content digests.
- Image signing or provenance attestation is produced but may not be enforced at admission.
- Images are pulled from registries or base images whose origin the pipeline does not verify.

## Scope check

Audit image provenance only for clusters, registries, and pipelines you own or are authorized to assess. A
confirming test may push a benign image or retag one in a registry you control, so stay inside the authorized
registry and remove any test artifact. If you can't name the authorization, stop.

## The loop

1. **Establish the intended artifact and its proof first.** Name, for each workload, exactly which image it is
   supposed to run and how that is proven: a content digest, a signature by a trusted key, a provenance
   attestation. This is the false-positive killer: a workload pinned to a digest whose signature and provenance
   the cluster verifies at admission is running a proven artifact. Name the intended proof, then check whether
   the reference and enforcement provide it.

2. **Check tag versus digest references.** For each workload, determine whether the image is referenced by a
   mutable tag or an immutable digest. A tag can be repointed, so a tag reference runs whatever content the tag
   resolves to at pull time. A digest reference fixes the content; a tag reference does not, and is the first
   provenance gap.

3. **Confirm signing enforcement at admission.** Determine whether the cluster requires a valid signature from
   a trusted key before running an image, or merely that signatures are produced somewhere. Enforcement lives
   at admission: if unsigned or unknown-key images are admitted, signing is not a control. Confirm the trusted
   keys and that verification is mandatory, not advisory.

4. **Check provenance attestation and its enforcement.** Where build provenance or an SBOM attestation is
   expected, confirm the cluster requires it and validates it against policy (the expected builder, source,
   and materials), not just that an attestation exists. An attestation nobody checks does not constrain what
   runs.

5. **Trace base images and registry trust.** Follow where images and their base layers come from. A workload
   pulling from a registry that admits unsigned images, or built on a base image whose origin was never
   verified, inherits that registry's or base's trust. Confirm the registries in use enforce provenance and
   that base images are pinned and verified.

6. **Confirm and record.** Confirm by running, or getting admitted, an image the policy should reject, an
   unsigned image, an image at a repointed tag, or one lacking required provenance, in a non-production
   namespace against a registry you control, and removing it after. Kill the lead if every workload pins a
   digest, the cluster enforces trusted-key signatures and required provenance at admission, and base images
   and registries are verified. Record the unverified reference, the running-container sink, and the missing
   proof of provenance.

## Where image provenance leaks

- **A mutable tag is not the artifact.** A tag can be repointed after review, so a tag-referenced workload
  runs whatever the tag resolves to at pull time.
- **Signatures without enforcement prove nothing.** A signature produced in the pipeline but not verified at
  admission does not stop an unsigned or wrong-key image from running.
- **Attestations nobody requires are decoration.** Build provenance the cluster does not validate against
  policy places no constraint on what runs.
- **A permissive registry admits anything.** Pulling from a registry that does not enforce provenance inherits
  its lack of trust.
- **Unverified base images poison the top layer.** A trusted application layer on an unverified base still runs
  the base's code.

## Worked example (a confirm and a kill)

> **Confirm.** Workloads reference images by a moving tag, and admission control checks image names but does
> not verify signatures. Repointing the tag in a registry the tester controls to a benign but different image,
> then triggering a redeploy, runs the new content with no signature check. **Confirmed** unverified image
> provenance to running container, `high`, remediation = pin workloads to content digests, enforce
> trusted-key signature verification and required build provenance at admission, and pull only from registries
> that enforce provenance.
>
> **Kill.** Every workload pins an image by content digest, admission requires a valid signature from a trusted
> key and a build-provenance attestation validated against the expected builder and source, base images are
> pinned and verified, and the registries in use reject unsigned images. An image at a repointed tag or without
> a required signature is refused at admission. **Killed**, `kill_reason` = "workloads pinned to verified
> digests with trusted-key signing and provenance enforced at admission and verified base images; no unproven
> image reaches a running container."

## Rationalizations to reject

- *"We use a specific tag."* → A tag is mutable; it can be repointed after review. Pin the content digest, or
  the reference proves nothing about the bytes.
- *"Our images are signed."* → Signed is not verified; confirm the cluster rejects unsigned and wrong-key
  images at admission, or the signature is decoration.
- *"We generate SBOMs and provenance."* → Producing an attestation is not requiring one; confirm the cluster
  validates it against policy before running.
- *"It is from our registry."* → Confirm the registry enforces provenance and the base image origin is
  verified; a private registry can still hold an unverified image.
- *"The image ran fine."* → Running proves it started, not that it is the intended artifact; provenance is
  about which bytes ran, not whether they ran.

## Executing this in practice

You need each workload's image reference (tag or digest), the admission enforcement for signatures and
provenance with the trusted keys and expected builder, the registries in use and their provenance posture,
and the base-image origins. For each workload, trace the reference back to a verified digest and confirm the
cluster refuses anything unproven. Reading the workload specs and admission policy shows the intended trust;
getting an unproven image admitted against a controlled registry shows whether it holds.

## Related

- `auditing-admission-control-policy-gaps` - the enforcement mechanism this skill relies on; a provenance
  policy is only as strong as the admission gate that applies it.
- `hunting-supply-chain-risks` - the broader artifact-provenance discipline; container images are one
  credential-bearing, directly-executed case of it.
- `auditing-container-image-build-hardening` - the build side: how the image is produced and what it embeds,
  the upstream half of proving what a workload runs.
- `auditing-iac-module-and-provider-supply-chain` - the same pin-and-verify discipline for infrastructure
  code that this skill applies to images.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the unverified image reference, sink = the running
  container, evidence = the missing proof of provenance.
