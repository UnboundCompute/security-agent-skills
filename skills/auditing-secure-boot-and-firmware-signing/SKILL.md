---
name: auditing-secure-boot-and-firmware-signing
description: >-
  Audit updater and bootloader code for a firmware trust boundary that lets an unsigned or downgraded
  image be flashed or booted. Covers an update image that reaches a flash write or a boot jump with no
  signature check between receipt and commit, a verification result that is ignored or inverted, an
  integrity hash mistaken for an authenticity signature, a signature checked over the wrong or partial
  bytes or over a different buffer than the one committed, a verification key kept in writable storage
  or selected by a field in the image, and anti-rollback that is missing or checked before the
  signature so a known-vulnerable version re-flashes. Use when reviewing code that receives, verifies,
  flashes, or boots a firmware image. The received image is the source, the flash write or boot jump is
  the sink, and a verified authenticity check failing to dominate that path is the bug.
license: MIT
---

# Auditing secure boot and firmware signing: does a verified signature gate every path to flash and boot

Firmware is the most durable place an attacker can land, and the whole defense is one property: an
image is verified for authenticity before any code writes it to a boot slot or jumps to it. Auditing
that property is a domination question, not a checklist. Trace the received image to every place it is
committed, to flash or to the program counter, and ask whether a correct, enforced signature check
stands on every path, over the exact bytes that reach the commit, against a key the attacker cannot
influence. The common failures are all ways that check is absent, thrown away, checking the wrong
thing, or checking the wrong bytes. You find them by reading the receipt-to-commit path, not by
trusting that a function named verify verifies.

## When to use

- Code receives a firmware image over an update channel and writes it to flash or activates a boot slot.
- A bootloader decides whether to execute an image at startup.
- You want to know whether an unsigned, tampered, or downgraded image can be flashed or booted.

## Scope check

Audit firmware trust boundaries only on devices and code you own or are authorized to assess, and
flash crafted images only to hardware in scope. A confirmed secure-boot bypass is persistent code
execution, so treat it accordingly and coordinate. If you can't name the authorization, stop.

## The loop

1. **Map the receipt-to-commit paths.** Find where an image arrives (an update channel, a port, a shared
   store) and every sink that commits it: a flash erase or write of a boot region, a call that marks a
   slot active or valid, and the branch to the image's entry point at boot. These sinks are what a
   verification must dominate, so enumerate them before judging any check.

2. **Check that verification exists and its result is enforced.** On each path, a signature check must run
   between receipt and commit. The bug is a commit with no check, a result computed and then discarded, a
   result defaulted to success and only set to failure on one branch, or an inverted test. A verify whose
   outcome does not gate the flash or the boot is not a check.

3. **Check authenticity, the bytes, and the buffer.** The check must verify an asymmetric signature, not a
   hash or checksum compared against a value carried in the image, which an attacker simply recomputes; a
   hash is integrity, not authenticity. It must cover the whole image, including the version and any
   rollback counter, not a header or a length-limited region. And it must verify the exact buffer that is
   committed, not one that is re-read or re-streamed between verification and flash.

4. **Check the trust anchor.** The verification key must live in immutable storage, read-only or
   one-time-programmable memory, not in the same writable region an update can overwrite. It must be fixed,
   not selected by an index or field taken from the image, and not a key or certificate embedded in the
   image itself. A fallback to a test or development key, or to an unsigned path, is the same failure.

5. **Check anti-rollback.** A monotonic version or counter must prevent re-flashing a known-vulnerable
   image, its value must be compared only after the signature is verified so the attacker does not control
   the compared field, and it must be stored where an attacker cannot reset it, in hardware-backed rather
   than ordinary writable storage. A rollback gate enforced by the updater but not the bootloader, or the
   reverse, is a split boundary.

6. **Confirm and record.** Confirm by flashing or presenting a crafted image, unsigned, tampered after the
   verified region, signed by a foreign key, or an older signed version, and showing it is committed or
   booted. Kill the lead if a bootloader re-verifies the same bytes against an anchored key before every
   boot so a bad flashed image will not run, if the flagged embedded key is a public verification key
   rather than a secret, if the bypass path is compiled out of the production build or gated by hardware,
   or if a weak-looking version compare is backstopped by a hardware monotonic counter. Record the sink,
   the missing or bypassable check, and the image that reached it.

## Where the firmware trust boundary leaks

- **A hash is integrity, not authenticity.** Comparing the image against a digest it carries stops
  corruption, not an attacker, who recomputes the digest. Only an asymmetric signature authenticates.
- **A verified result thrown away is no result.** The most common real finding is a verify that runs and
  whose return is ignored, defaulted true, or inverted; the check exists and protects nothing.
- **Verifying the wrong bytes or the wrong buffer is a bypass.** A signature over the header only, or over
  a buffer that is re-read before flashing, leaves the committed bytes unauthenticated.
- **The key is the anchor; a movable anchor holds nothing.** A key in writable flash, selected by the
  image, or embedded in it lets the attacker choose who signs.
- **Rollback checked before the signature trusts the attacker's number.** The version must be
  authenticated before it is compared, and stored where it cannot be reset.

## Worked example (a confirm and a kill)

> **Confirm.** The updater computes a digest of the received image, compares it to a digest field in the
> image header, and on a match erases and writes the boot slot and reboots. There is no asymmetric
> signature anywhere on the path, so an attacker ships a malicious image together with its matching digest
> and it is flashed and booted. **Confirmed** integrity check mistaken for authenticity, `critical`,
> remediation = verify an asymmetric signature over the complete image against a key anchored in immutable
> storage before any flash, and fail closed on the result.
>
> **Kill.** The updater has no verification, but the bootloader, before every jump, verifies an asymmetric
> signature over the whole image, including its version, against a key whose hash is held in
> one-time-programmable memory, and refuses to boot on failure or on a version below the stored monotonic
> counter; the key embedded in the tree is the public verification key. A flashed malicious or downgraded
> image does not boot. **Killed**, `kill_reason` = "boot-time re-verification of the committed bytes
> against an anchored key with a hardware rollback counter dominates the boot path; the embedded key is
> public, not a secret."

## Rationalizations to reject

- *"The image is hashed, so it is verified."* -> A hash proves it was not corrupted, not who made it. An
  attacker recomputes the hash. Authenticity needs a signature.
- *"Verification runs before the flash."* -> Is its result enforced, over the whole image, on the exact
  buffer that is written? A verify whose outcome is ignored or whose bytes differ is decorative.
- *"The key is in the firmware, so it is protected."* -> A key in writable flash is overwritable, and a key
  selected by the image is the attacker's choice. Only an immutable, fixed anchor holds.
- *"We block downgrades by checking the version."* -> From the unverified header, before the signature? Then
  the attacker sets the version. Compare it only after authenticating it, from hardware-backed storage.
- *"There is a debug switch to skip verify, but it is off."* -> Off in which build? If it compiles into
  production or is not hardware-gated, it is a bypass, not a convenience.

## Executing this in practice

You need the receipt path and every flash and boot sink, the verification routine and exactly what bytes
and buffer it covers and how its result is used, where the verification key is stored and how it is
selected, and the rollback counter's storage and the order of its check against verification. For each
sink, decide whether a verified authenticity check over the committed bytes dominates it. Reading the
path tells you which checks exist; flashing or presenting a crafted image on hardware in scope tells you
whether they hold.

## Related

- `hunting-firmware-secrets-and-debug-interfaces` - the companion firmware audit; an unauthenticated update
  path there plus a missing signature here is one trivial persistent-code-execution chain.
- `finding-crypto-misuse` - the signature routine can itself be misused (weak algorithm, unchecked padding,
  wrong-length compare); that skill covers the primitive this one assumes is sound.
- `finding-fail-open-flaws` - a verification result ignored, inverted, or defaulted to success is a
  fail-open control in the boot path; both hunt a check that does not stop what it should.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the received image, sink = the flash write or boot
  jump, evidence = the unsigned, tampered, or downgraded image that is committed or booted.
