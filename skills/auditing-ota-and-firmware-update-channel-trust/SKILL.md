---
name: auditing-ota-and-firmware-update-channel-trust
description: >-
  Audit an over-the-air or firmware update channel for a device that accepts an image it should reject: an
  update whose signature is not verified so an attacker installs arbitrary firmware, an update fetched over an
  unauthenticated transport an on-path attacker can swap, a rollback to an older signed image with known
  vulnerabilities because the device does not enforce version monotonicity, an update server or manifest URL the
  device trusts without authentication, and an unencrypted image that leaks secrets and eases reverse
  engineering. Covers IoT and embedded devices, routers, wearables, and any product that fetches and installs
  firmware or application updates in the field. Use when a device installs firmware it fetches and the
  verification of that image before it runs is the boundary. The unsigned, swapped, or rolled-back update is the
  source, the persistent code execution on the device is the sink, and the missing signature check, transport
  authentication, or rollback protection is the bug.
license: MIT
---

# Auditing OTA and firmware update channel trust: whatever a device installs, it runs forever

A firmware update is the most powerful thing that can happen to a device: whatever image it installs becomes
the code it runs, persistently, with full control of the hardware. So the update channel is the highest-value
boundary on the device, and the only thing standing between an attacker and permanent control is whether the
device verifies an image before it installs it. The failures are a checklist of that verification. If the
device does not verify a cryptographic signature over the image against a key it trusts, an attacker installs
arbitrary firmware. If it fetches the image over an unauthenticated transport, an on-path attacker swaps the
image in flight even if the server is honest. If it does not enforce version monotonicity, an attacker rolls
the device back to an older, still-signed image with known vulnerabilities and re-exploits it. If it trusts an
update server or manifest URL without authenticating it, an attacker who controls or spoofs that endpoint feeds
it a malicious image. And an unencrypted image leaks embedded secrets and hands a reverse engineer the code.
The audit follows an update from where the device learns of it to where it installs it and checks that the
image is signature-verified, transport-authenticated, and version-monotonic before it ever runs. You audit this
by trying to get the device to install an image it should refuse.

## When to use

- A device fetches and installs firmware or application updates in the field (IoT, embedded, router, wearable).
- The update image may not be signature-verified, or may be fetched over an unauthenticated transport.
- The device may allow rollback to older signed images, or trust an update server or manifest URL without
  authentication.

## Scope check

Test update channels only on devices and infrastructure you own or are authorized to assess, in a controlled
environment. Feeding a device a crafted image or intercepting its update traffic can brick hardware and exercise
a real install path, so use your own devices and test servers and never push firmware to or intercept updates
for a device that is not yours. If you can't name the authorization, stop.

## The loop

1. **Establish how the device verifies an image before installing it first.** Name the intended checks: a
   signature over the whole image verified against a trusted key, an authenticated transport to an
   authenticated server, a version-monotonicity rule that refuses older images, and whether the image is
   encrypted. This is the false-positive killer: a device that verifies a signature against a trusted key,
   fetches over an authenticated transport from an authenticated server, refuses rollback below its current
   version, and ships an encrypted image is protecting the channel. Name the intended verification, then try to
   defeat each check.

2. **Test signature verification.** Offer the device a modified or unsigned image and confirm it refuses to
   install. Alter a byte of a signed image and confirm the signature check fails closed. A device that installs
   an image whose signature it does not verify (or verifies against a key an attacker can supply) accepts
   arbitrary firmware, the core bug.

3. **Test the transport and server authentication.** Determine how the device fetches the image and the update
   manifest, and whether the transport authenticates the server (not just encrypts) so an on-path attacker
   cannot swap the image or redirect the fetch. An unauthenticated transport or a manifest URL trusted without
   authentication lets an on-path or spoofed server feed a malicious image even when signatures exist but are
   checked weakly.

4. **Test rollback protection.** Offer an older, still-validly-signed image with a known vulnerability and
   confirm the device refuses it because it enforces version monotonicity. A device that accepts any signed
   image regardless of version can be rolled back and re-exploited through a bug that was already fixed.

5. **Check image confidentiality and secret exposure.** Determine whether the image is encrypted and whether it
   embeds secrets (keys, credentials, tokens). An unencrypted image hands a reverse engineer the code and any
   embedded secrets, easing exploit development and revealing keys that may protect other devices.

6. **Confirm and record.** Confirm on your own device by installing a modified or unsigned image, swapping the
   image over an unauthenticated transport, or rolling back to a vulnerable signed version, in a controlled
   environment and without touching a device that is not yours. Kill the lead if the device verifies a
   signature against a trusted key, fetches over an authenticated transport from an authenticated server,
   enforces version monotonicity, and encrypts the image. Record the unsigned, swapped, or rolled-back update,
   the persistent code execution on the device, and the missing signature check, transport authentication, or
   rollback protection.

## Where update-channel trust leaks

- **No signature verification.** A device that installs an image without verifying a signature against a
  trusted key runs whatever an attacker supplies, permanently.
- **Unauthenticated transport.** Fetching the image or manifest over a transport that does not authenticate the
  server lets an on-path attacker swap the image in flight.
- **Trusted but unauthenticated server or manifest.** A device that trusts an update endpoint or manifest URL
  without authenticating it takes a malicious image from a spoofed or compromised source.
- **No rollback protection.** Accepting any signed image regardless of version lets an attacker roll back to an
  older signed build with a known vulnerability and re-exploit it.
- **Unencrypted image.** A plaintext image leaks embedded secrets and hands a reverse engineer the code, easing
  exploitation of this and sibling devices.

## Worked example (a confirm and a kill)

> **Confirm.** An IoT device fetches its firmware from a fixed URL over an unauthenticated transport and checks
> only a CRC of the image, not a cryptographic signature. On a test bench, an on-path attacker serves a modified
> image with a correct CRC; the device installs it and runs attacker-controlled firmware persistently.
> **Confirmed** arbitrary firmware install via missing signature verification and unauthenticated transport,
> `critical`, remediation = verify a cryptographic signature over the whole image against a trusted key before
> install, fetch over an authenticated transport from an authenticated server, and treat the CRC as
> integrity-only, never as authentication.
>
> **Kill.** The device verifies a cryptographic signature over the whole image against a key in its trust store
> before installing, fetches the image and manifest over an authenticated transport from an authenticated
> server, refuses any image whose version is not greater than the running one, and ships the image encrypted. A
> modified image fails the signature check, a swapped image fails transport authentication, and an older signed
> image is refused by the monotonicity rule. **Killed**, `kill_reason` = "image is signature-verified against a
> trusted key over an authenticated transport with enforced version monotonicity and encryption; no unsigned,
> swapped, or rolled-back image installs."

## Rationalizations to reject

- *"The image has a checksum."* → A CRC or hash is integrity against corruption, not authentication against an
  attacker who recomputes it; require a cryptographic signature against a trusted key.
- *"It updates over TLS."* → Confirm the transport authenticates the server (validates the certificate) and that
  the device does not fall back to plaintext; encryption without server authentication still allows a swap.
- *"It only installs signed images."* → Confirm it also enforces version monotonicity; an older signed image
  with a known bug is a valid signature and a rollback attack.
- *"The update server is ours."* → An on-path attacker or a spoofed endpoint sits between the device and your
  server; the device must authenticate the server and the image, not assume the endpoint is honest.
- *"Nobody will extract the firmware."* → Firmware is routinely dumped from flash; an unencrypted image leaks
  secrets and code, so encrypt it and never rely on the image staying private.

## Executing this in practice

You need how the device learns of and fetches an update (server, manifest, transport), how it verifies the
image (signature and trusted key, or only a checksum), whether it enforces version monotonicity, and whether the
image is encrypted. On a test bench, offer modified, unsigned, swapped, and older-signed images and observe
what installs. Reading the update client and boot verification shows the intended checks; an image that installs
when it should be refused shows which check is missing.

## Related

- `auditing-ble-and-gatt-authorization` - the live-control channel of the same class of device; GATT
  authorization and update-channel trust are the two main remote surfaces of an embedded device.
- `hunting-firmware-secrets-and-debug-interfaces` - an unencrypted image and a device that embeds keys leak the
  very secrets that should protect the update channel; the two failures compound.
- `hunting-supply-chain-risks` - a firmware image is a supply-chain artifact; signing, provenance, and version
  control are the same trust applied to what a device installs.
- `auditing-declarative-authorization` - who may push an update and to which device is an authorization decision
  the server side of this channel must enforce alongside the device-side verification.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the unsigned, swapped, or rolled-back update, sink =
  the persistent code execution on the device, evidence = the missing signature check, transport authentication,
  or rollback protection.
