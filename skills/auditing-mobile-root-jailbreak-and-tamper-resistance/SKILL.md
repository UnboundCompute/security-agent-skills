---
name: auditing-mobile-root-jailbreak-and-tamper-resistance
description: >-
  Audit whether a mobile app's integrity and environment checks actually enforce a security decision, where
  the app runs a root or jailbreak check, a repackaging or debugger check, or an emulator check but decides
  the result on the client, so a controlled device defeats it, or the app relies on such a check to protect a
  secret or an operation that a genuine attacker on their own device simply bypasses, because the check is
  advisory rather than a real control and the protected asset is reachable once it is defeated. Use when a
  mobile app performs environment or tamper checks and treats their outcome as a security boundary. Covers
  client-decided integrity checks, checks not bound to a server-verified attestation, and secrets or
  operations gated only by such a check. The controlled or tampered device the check cannot truly detect is
  the source, the client integrity decision is the sink, and reaching the protected asset by defeating the
  check is the bug.
license: MIT
---

# Auditing mobile root/jailbreak and tamper resistance: when a client check guards nothing

A mobile app often tries to protect itself by checking its environment: is the device rooted or jailbroken, is
the app repackaged or being debugged, is it running on an emulator. These checks are reasonable telemetry, but
they become a false boundary when the app treats their outcome as a security decision made on the client. An
attacker runs the app on a device they fully control, and any check whose verdict is computed and enforced
locally can be forced to return the answer the attacker wants, the detection routine patched, the response
hooked, the flag flipped. So the real question is not whether the app checks, but whether anything important
depends on the check, and whether the dependency is enforced somewhere the attacker cannot reach. A secret
gated only by a local root check is recovered the moment the check is defeated; an operation allowed only when
the app believes it is untampered proceeds once the app is made to believe it. The bug is a security decision
resting on a client-side integrity verdict. You audit these by finding what each check protects and whether a
server verifies the environment independently.

## When to use

- A mobile app performs root, jailbreak, repackaging, debugger, or emulator checks and acts on the result.
- A secret or a sensitive operation is gated by the app's own integrity or environment verdict.
- The app treats a client-computed integrity result as a security boundary rather than as telemetry.

## Scope check

Audit tamper resistance only on apps and devices you own or are authorized to assess, on test devices and
accounts, defeating a check on a device you control to reach only test assets rather than real user data. A
confirmed bypass reaches whatever the check was trusted to protect, so keep every probe within scope. If you
can't name the authorization, stop.

## The loop

1. **Establish what each check protects and where the decision is enforced first.** For each integrity or
   environment check, determine what asset or operation depends on its outcome and whether the decision is
   made and enforced only on the client, or whether a server independently verifies the environment through an
   attestation the client cannot forge. This is the false-positive killer: a check that is pure telemetry,
   protecting nothing, and one whose verdict a server verifies independently are both fine; the finding is a
   security decision resting on a client-computed result. Name what the check protects before defeating it.

2. **Enumerate the checks and their consequences.** List the root, jailbreak, repackaging, debugger, and
   emulator checks the app performs, and for each what happens on a positive result: a refused operation, a
   withheld secret, a blocked feature. Checks whose result gates something valuable are the candidates.

3. **Check where the verdict is decided.** Determine whether the check's outcome is computed and acted on
   entirely within the app, so patching the routine or hooking the response changes it, or whether the
   environment is attested to a server that decides independently and cannot be satisfied by a forged client
   result.

4. **Check what the protected asset really depends on.** Determine whether a secret gated by the check is
   still reachable once the check is defeated, because it is present on the device, or whether it is held so
   that defeating the check yields nothing, for example released only by a server after independent
   verification. The asset's real dependency, not the check, decides the impact.

5. **Check for server-side attestation binding.** Determine whether any server-verified attestation is bound
   to the actual operation and session, so a genuine attestation cannot be replayed or lifted to authorize a
   different action, or whether the app merely reports its own verdict to the server as a claim.

6. **Confirm and record.** Confirm by defeating the check on a controlled test device, forcing the verdict or
   removing the check, and reaching the protected test asset or operation. Kill the lead if the check protects
   nothing beyond telemetry, if the protected asset is not reachable once the check is defeated, or if a
   server verifies the environment through an attestation bound to the operation that a controlled device
   cannot forge. Record the check, what it protected, the bypass, and the asset reached, or set a
   `kill_reason`.

## Where tamper resistance leaks

- **What the check protects is the finding.** A client integrity check is telemetry until something depends
  on it; the bug is a security decision resting on a verdict the client computes. Name the dependency.
- **Client-decided verdicts are forgeable.** A check computed and enforced in the app is patched, hooked, or
  flipped on a controlled device, so its answer becomes whatever the attacker wants.
- **Gated secrets present on the device are recovered.** A secret withheld only by a local check is still on
  the device; defeating the check reaches it, so the check was never the real protection.
- **Reported verdicts are just claims.** An app that tells the server it is untampered is making a claim a
  controlled device sets freely; only an unforgeable attestation the server verifies changes that.
- **Unbound attestation is replayable.** A genuine attestation not bound to the specific operation and session
  can be lifted or replayed to authorize a different action, so the binding matters as much as the attestation.

## Worked example (a confirm and a kill)

> **Confirm.** An app withholds a stored secret when it detects a jailbroken device, but the detection and the
> decision are entirely in the app and the secret is present on the device. On a controlled test device the
> detection routine is defeated and the secret is recovered, reaching a test asset the check was trusted to
> protect. **Confirmed** a security decision resting on a client integrity verdict, `high`, remediation = do
> not gate secrets on a client-side integrity check, hold sensitive material so it is released only by a
> server after an unforgeable attestation bound to the operation, and treat client environment checks as
> telemetry only.
>
> **Kill.** The same app treats its root and debugger checks as telemetry it reports for monitoring, and the
> sensitive operation is authorized only by a server that verifies an unforgeable device attestation bound to
> the specific operation and session. Defeating the client checks changes only the telemetry; the server still
> refuses the operation on a device that cannot produce a genuine bound attestation. **Killed**, `kill_reason`
> = "no secret or operation depends on the client verdict; the server authorizes only against an unforgeable
> attestation bound to the operation, which a controlled device cannot forge."

## Rationalizations to reject

- *"We detect rooted devices."* -> Detection is not enforcement; if the app decides and acts locally, a
  controlled device forces the verdict, so the question is what depends on it and where that is enforced.
- *"The check stops casual tampering."* -> As telemetry that is fine, but a genuine attacker on their own
  device defeats it; nothing valuable should depend on a check a controlled device can flip.
- *"The secret is only released when the check passes."* -> If the secret is on the device, defeating the
  check reaches it; hold it so it is released only by a server after independent verification.
- *"The app tells the server it is safe."* -> A self-reported verdict is a client-set claim; only an
  unforgeable attestation the server verifies means anything.
- *"We use attestation."* -> Confirm the attestation is unforgeable and bound to the operation and session; an
  unbound or replayable attestation authorizes a different action than the one attested.

## Executing this in practice

You need every integrity and environment check, what asset or operation depends on each, whether the verdict
is decided on the client or verified by a server through an unforgeable attestation, and whether any such
attestation is bound to the operation. For each check that gates something valuable, decide whether defeating
it on a controlled device reaches the asset. Reading where the decision is enforced settles most leads;
defeating the check on a controlled test device and reaching the protected test asset settles the rest.

## Related

- `auditing-mobile-biometric-and-local-auth-bypass` - a local-auth gate is another client-side decision a
  controlled device defeats, sharing the client-verdict reasoning with this skill.
- `hunting-ios-keychain-and-data-protection-gaps` - the platform protection classes a secret relies on are
  weakened on a rooted or jailbroken device, the environment this skill audits.
- `auditing-webauthn-and-passkey-flows` - an unforgeable, operation-bound attestation is the server-side
  control that replaces a client integrity verdict, the binding that skill treats for authenticators.
- `auditing-ota-and-firmware-update-channel-trust` - a repackaged or tampered app is delivered through an
  update or distribution channel whose integrity that skill audits.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the controlled or tampered device the check cannot
  truly detect, sink = the client integrity decision, evidence = reaching the protected test asset by
  defeating the check on a controlled test device.
