---
name: auditing-mobile-biometric-and-local-auth-bypass
description: >-
  Audit local authentication on a mobile app, where a biometric or device-passcode gate protects sensitive
  access but can be bypassed because the app trusts the result of the prompt rather than a key released by it,
  gates only the user interface while the protected data or action remains reachable, does not invalidate the
  key when enrolled biometrics change, or accepts a callback an attacker can forge on a controlled device. Use
  when a mobile app gates sensitive data or actions behind a biometric or local-auth prompt. Covers
  result-only trust without a bound key, interface-only gating, missing invalidation on enrollment change, and
  forgeable success callbacks. The local-auth prompt the app relies on is the source, the sensitive data or
  action it is meant to protect is the sink, and reaching that data or action without a genuine local
  authentication is the bug.
license: MIT
---

# Auditing mobile biometric and local auth bypass: when the prompt says yes but proves nothing

A mobile app often guards a sensitive action or a stored secret behind a local authentication prompt: a
fingerprint, a face, or the device passcode. That gate is meaningful only when passing it actually releases
something the protected path needs, a key that decrypts the data, held so the data cannot be reached without a
genuine authentication. It is theater when the app merely asks for the prompt and then trusts its own record
of the answer: a success flag the app sets, a callback it believes, or a screen it hides. On a device the
attacker controls, a boolean result can be forced, a callback can be forged, and a hidden screen's data is
still in memory or on disk. The gate also fails when the released key is not invalidated after the enrolled
biometrics change, so a newly added fingerprint unlocks the old secret, or when only the interface is gated
while the protected data or action remains directly reachable. The bug is reaching the protected data or
action without a genuine local authentication. You audit these by finding what the prompt actually releases
and whether the protected path needs it.

## When to use

- A mobile app gates sensitive data or an action behind a biometric or device-passcode prompt.
- The app may trust a success result or callback rather than a key the authentication releases.
- The protected data or action may be reachable without passing the prompt on a controlled device.

## Scope check

Audit local authentication only on apps and devices you own or are authorized to assess, on test devices and
accounts, demonstrating a bypass against test data rather than a real user's. A confirmed bypass reaches data
or actions meant to require the user's presence, so keep every probe within scope. If you can't name the
authorization, stop.

## The loop

1. **Establish what the prompt actually releases and whether the protected path needs it first.** For each
   local-auth gate, determine whether passing it releases a key that the protected data or action genuinely
   requires, so the path cannot complete without a real authentication, or whether the app trusts a result, a
   callback, or a hidden interface while the data or action remains reachable. This is the false-positive
   killer: a gate that binds the sensitive data to a key released only by a genuine authentication cannot be
   bypassed by forcing a result. Name what the prompt releases before crafting a bypass.

2. **Enumerate the gated data and actions.** List the sensitive secrets, data, and actions the app protects
   behind a local-auth prompt, and for each what reaching it would expose or allow. These are the targets a
   bypass would reach.

3. **Check for result-only trust.** Determine whether the app decides access from a boolean success, a
   callback it trusts, or a flag it sets, rather than from a key the authentication released. A gate that
   trusts its own record of success can be satisfied by forcing that record on a controlled device.

4. **Check the key binding and invalidation.** Determine whether the protected secret is bound to a key that
   is released only by a genuine local authentication and is invalidated when the enrolled biometrics change,
   or whether the key survives an enrollment change so a newly added biometric unlocks the old secret.

5. **Check for interface-only gating.** Determine whether the gate only hides or locks a screen while the
   underlying data remains in memory, on disk, or reachable through another entry point, so the action or data
   is available without ever passing the prompt.

6. **Confirm and record.** Confirm by reaching a gated secret or action without a genuine authentication on a
   controlled test device, by forcing the result, forging the callback, reading the unbound data, or using a
   post-enrollment key, against test data. Kill the lead if the protected path requires a key released only by
   a genuine authentication and invalidated on enrollment change, and if no data or action is reachable
   without the prompt. Record the gate, the target, the bypass mechanism, and the access observed, or set a
   `kill_reason`.

## Where local authentication leaks

- **What the prompt releases is the finding.** A gate matters only if passing it releases something the
  protected path needs; a gate that trusts a result protects nothing on a controlled device.
- **Result-only trust is forgeable.** A boolean success, a trusted callback, or a set flag can be forced on a
  device the attacker controls, so access decided from the result is access without authentication.
- **Unbound data is reachable anyway.** A secret not bound to a key the authentication releases is readable
  from storage or memory regardless of whether the prompt was ever shown.
- **Missing invalidation survives enrollment change.** A key not invalidated when the enrolled biometrics
  change lets a newly added fingerprint or face unlock the old secret, defeating the presence guarantee.
- **Interface-only gating hides nothing underneath.** A locked screen whose data is still in memory, on disk,
  or reachable through another entry point is gated only visually.

## Worked example (a confirm and a kill)

> **Confirm.** An app reveals a stored secret after a biometric prompt, but decides access from a success
> callback it trusts and stores the secret without binding it to a key the authentication releases. On a
> controlled device the callback is forged and the secret is also read directly from storage without any
> prompt, against test data. **Confirmed** local-auth bypass reaching a protected secret, `high`, remediation
> = bind the secret to a key released only by a genuine local authentication, invalidate that key when the
> enrolled biometrics change, and never decide access from a result the app records itself.
>
> **Kill.** The same app stores the secret encrypted under a key held so that only a genuine local
> authentication releases it, invalidates the key when the enrolled biometrics change, and exposes no copy of
> the data outside that key. Forcing the result or forging the callback releases no key, and the storage holds
> only ciphertext the attacker cannot decrypt. **Killed**, `kill_reason` = "the secret is bound to a key
> released only by a genuine authentication and invalidated on enrollment change; forcing the result yields no
> usable data and nothing is reachable without the prompt."

## Rationalizations to reject

- *"The biometric prompt is enforced by the OS."* -> The OS enforces the prompt, but if the app trusts the
  result rather than a key the prompt releases, a controlled device can force the result; bind data to the key.
- *"Access requires a successful callback."* -> A callback the app trusts can be forged on a controlled
  device; the authentication must release a key the data needs, not just signal success.
- *"The secret is only shown after auth."* -> If the secret is stored unbound to the auth-released key, it is
  readable from storage without the prompt; gating the display does not gate the data.
- *"A new fingerprint is still the user's."* -> An attacker who can enroll a biometric on a controlled device
  gains one too; the key must invalidate on enrollment change so an added biometric cannot unlock the old
  secret.
- *"We lock the screen on background."* -> Locking the interface leaves the data in memory or on disk; confirm
  the data itself is protected, not just the screen hidden.

## Executing this in practice

You need every gated secret and action, what passing the prompt actually releases, whether the protected data
is bound to that key and invalidated on enrollment change, and whether the data or action is reachable through
storage, memory, or another entry point without the prompt. For each gate, decide whether a genuine
authentication is truly required. Reading what the prompt releases settles most leads; reaching a gated target
without a genuine authentication on a controlled test device settles the rest.

## Related

- `hunting-ios-keychain-and-data-protection-gaps` - a keychain item gated by biometric presence relies on the
  key binding this skill audits, so protection class and local-auth gate meet on the same item.
- `auditing-webauthn-and-passkey-flows` - the server-side counterpart, where an authenticator's presence
  proof must be verified rather than a client result trusted.
- `auditing-mobile-root-jailbreak-and-tamper-resistance` - a controlled or tampered device is exactly where a
  result-only gate is bypassed, the environment that skill audits.
- `hunting-mobile-secret-and-storage-exposure` - a secret reachable without the prompt is the storage exposure
  that skill pursues from the at-rest side.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the local-auth prompt the app relies on, sink = the
  sensitive data or action it protects, evidence = reaching that data or action without a genuine local
  authentication on a controlled test device.
