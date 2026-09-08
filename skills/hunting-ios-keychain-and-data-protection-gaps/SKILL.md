---
name: hunting-ios-keychain-and-data-protection-gaps
description: >-
  Hunt at-rest exposure of secrets on iOS, where a credential, token, or sensitive value is stored with a
  keychain accessibility class or a data-protection class weaker than it needs, so it is readable when the
  device is locked, survives into a device backup, or is reachable by another app or after theft, because the
  item is marked always-accessible, is not restricted to this device, or the file protection lets it be read
  outside an unlocked session. Use when reviewing how an app stores secrets and files and which protection
  class each carries. Covers over-permissive keychain accessibility, missing this-device-only restriction,
  backup-included secrets, and weak file data-protection classes. The secret stored under a weak protection
  class is the source, the read outside the intended unlocked and on-device context is the sink, and
  recovering a usable secret in that weaker context is the bug.
license: MIT
---

# Hunting iOS keychain and data protection gaps: when a secret is readable while the device is locked

iOS gives every stored secret a protection class that decides when and where it can be read: a keychain item
has an accessibility attribute, and a file has a data-protection class, and together they say whether the
value is available only after the user unlocks the device, whether it is bound to this device alone, and
whether it leaves in a backup. Those classes are safe only when each secret carries the strictest one it can
tolerate. The gap is a secret stored under a class weaker than it needs: a token marked always-accessible that
can be read while the device is locked, a credential not restricted to this device that migrates in a backup
to another one, or a file whose protection class lets it be read outside an unlocked session. The bug is not
that the secret is stored; it is a protection class that exposes it in a context, locked, backed up, migrated,
or post-theft, where it should be unreadable. You hunt these by listing every stored secret and checking the
protection class against the context it must survive.

## When to use

- An iOS app stores credentials, tokens, or sensitive values in the keychain or in files.
- A stored item may carry an accessibility or data-protection class weaker than its sensitivity requires.
- Secrets may survive into a device backup or migrate to another device when they should not.

## Scope check

Hunt at-rest protection gaps only on apps and devices you own or are authorized to assess, on test devices
and accounts, proving a secret is recoverable in the weaker context without using it against a real account. A
recovered secret authenticates as its owner, so keep every probe within scope and treat recovered material as
sensitive. If you can't name the authorization, stop.

## The loop

1. **Establish the protection class each secret actually carries and the context it must survive first.** For
   each stored secret, determine its keychain accessibility or file data-protection class and the strongest
   context it needs to resist: locked-device read, backup inclusion, migration to another device, or
   post-theft access. This is the false-positive killer: a secret stored with the strictest class it can
   tolerate, available only when unlocked and bound to this device, is not exposed even though it exists on
   disk. Name the class and the context before claiming a gap.

2. **Enumerate stored secrets and their storage.** List the credentials, tokens, keys, and sensitive values
   the app stores, and for each whether it is in the keychain or a file, and the accessibility or
   data-protection class assigned. Secrets whose class is looser than the strictest they can tolerate are the
   candidates.

3. **Check the locked-device availability.** Determine whether a secret marked available while the device is
   locked, or with a file class readable outside an unlocked session, can be read without the user unlocking.
   A secret needed only while the app is in active use should not be available when the device is locked.

4. **Check the this-device and backup restrictions.** Determine whether a secret lacks the this-device-only
   restriction, so it migrates to another device through a backup or transfer, and whether it is included in
   backups at all. A secret bound to the hardware should never appear on another device.

5. **Check the post-theft and other-app reach.** Determine whether the weaker class lets the secret be read by
   an attacker with physical possession of a locked device, or through a backup an attacker can obtain, or by
   another app sharing an access group it should not. The context that matters is the one the attacker
   actually has.

6. **Confirm and record.** Confirm by recovering a usable secret in the weaker context on a test device, read
   while locked, extracted from a backup, or present after migration, without using it against a real account.
   Kill the lead if every secret carries the strictest class it can tolerate, if none is available while
   locked or in a backup beyond its need, and if none migrates off the device. Record the secret, its
   protection class, the context it was recovered in, and the proof, or set a `kill_reason`.

## Where at-rest protection leaks

- **The protection class is the finding.** A stored secret is expected; the gap is a class weaker than the
  context requires, exposing it while locked, in a backup, or after migration. Name the class and the context.
- **Always-accessible items read while locked.** A keychain item marked available regardless of lock state, or
  a file class readable outside an unlocked session, is exposed on a locked device.
- **Missing this-device-only migrates secrets.** A secret without the this-device restriction travels in a
  backup or device transfer to hardware the owner did not intend, defeating device binding.
- **Backups carry secrets off the device.** A secret included in a backup is exposed wherever the backup can
  be read, which may be far weaker than the device's own protection.
- **Shared access groups widen reach.** A keychain access group broader than needed lets another app read a
  secret that should be private to the storing app.

## Worked example (a confirm and a kill)

> **Confirm.** An app stores a long-lived authentication token in the keychain marked available regardless of
> lock state and without the this-device-only restriction. The token is read from a locked test device and is
> also present after restoring a backup to a second device, recovered without using it against a real account.
> **Confirmed** at-rest secret exposure through a weak keychain accessibility class, `high`, remediation = store
> the token with an accessibility class available only when the device is unlocked and restricted to this
> device, exclude it from backups, and scope the keychain access group to the app alone.
>
> **Kill.** The same token is stored with an accessibility class available only after first unlock while
> unlocked, restricted to this device, excluded from backups, and in an access group private to the app.
> It cannot be read on a locked device, does not appear after restoring a backup to another device, and no
> other app can reach it. **Killed**, `kill_reason` = "the token carries the strictest class it can tolerate,
> unlocked-only and this-device-only, is excluded from backups, and is privately scoped; no read in a weaker
> context recovers it."

## Rationalizations to reject

- *"The keychain is encrypted."* -> Encryption is governed by the accessibility class; an always-accessible
  item is decryptable while the device is locked, so the class, not the fact of encryption, decides exposure.
- *"It is only on the device."* -> Without the this-device-only restriction the secret migrates through a
  backup or transfer to another device; confirm the binding rather than assuming local means local.
- *"Backups are the user's own."* -> A backup can be read on a computer or in a cloud store far weaker than
  the device; exclude sensitive secrets from backups rather than trusting the backup's protection.
- *"Another app cannot read our keychain."* -> Only if the access group is scoped to the app; a broad or
  shared access group lets a sibling app read the item.
- *"The user has a passcode."* -> A passcode gates unlocked access, but an always-accessible item or a
  backup-included secret is reachable without unlocking or off the device entirely.

## Executing this in practice

You need every stored secret with its keychain accessibility or file data-protection class, its this-device
and backup status, and its access group, plus the strictest context each secret must survive. For each,
decide whether the class exposes it while locked, in a backup, after migration, or to another app. Reading the
assigned classes settles most leads; recovering a usable secret in the weaker context on a test device, while
locked, from a backup, or after migration, settles the rest.

## Related

- `hunting-mobile-secret-and-storage-exposure` - the cross-platform secret-at-rest hunt, of which the iOS
  keychain and data-protection classes here are the platform-specific mechanism.
- `auditing-mobile-biometric-and-local-auth-bypass` - a keychain item gated by biometric presence relies on
  the local-authentication check that skill audits, so protection class and auth gate meet on the same item.
- `auditing-ios-app-group-and-pasteboard-exposure` - a secret shared through an app group or the pasteboard
  leaves the keychain's protection, the adjacent exposure that skill covers.
- `auditing-mobile-root-jailbreak-and-tamper-resistance` - a jailbroken device weakens the platform guarantees
  these classes rely on, the environment that skill audits.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the secret stored under a weak protection class, sink
  = the read outside the intended unlocked and on-device context, evidence = a usable secret recovered while
  locked, from a backup, or after migration on a test device.
