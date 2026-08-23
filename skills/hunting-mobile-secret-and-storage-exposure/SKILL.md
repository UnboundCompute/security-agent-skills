---
name: hunting-mobile-secret-and-storage-exposure
description: >-
  Hunt a mobile app for a real credential shipped in the binary or written to storage another party can
  read, scoped strictly to mobile-specific sinks. Covers a live secret embedded in the app package or its
  resources, sensitive data written to world-or-sandbox-readable storage without encryption, a secret
  placed outside the platform keystore where a weaker guard protects it, data cached or logged where
  another app or a device-level reader reaches it, and a backup or debug path that carries sensitive data
  off the device. Use when reviewing the app package, its storage writes, and its logging, distinguishing a
  public identifier from a credential and judging whether the platform sandbox already contains the data.
  The embedded or stored secret is the source, a party that can read it is the sink, and a real credential
  exposed beyond its intended reader is the bug.
license: MIT
---

# Hunting mobile secret and storage exposure: a real credential a real reader can reach

Not every string that looks like a key is a secret, and not every file an app writes is exposed, so this
hunt turns on two questions the noisy scanners skip: is this value actually a credential, and can a party
who should not read it actually reach it. A public app identifier shipped in the binary is meant to be
public; a value inside the platform sandbox is readable only by the app itself on a non-compromised device.
The bug is a live credential in the package or in storage that a reachable party (another app, a
device-level reader, a backup, a log sink) can read. You hunt it by finding the mobile-specific sinks and
adjudicating both the value's sensitivity and the reader's reach. Stay on mobile-specific storage and
packaging; server-side secret handling is a different skill.

## When to use

- You have a mobile app package, its embedded resources, and the code that writes storage and logs.
- You see an embedded key or token, a file written to storage, a cache, a log line, or a backup rule.
- You want to know which values are real credentials and which of those a party can actually read.

## Scope check

Audit only apps you own or are authorized to assess, and extract secrets only from a package or device in
scope, a real embedded credential is a live secret. Treat any credential you surface as sensitive and do
not use it. If you can't name the authorization, stop.

## The loop

1. **Separate credential from public identifier first.** For each candidate value, decide whether it is a
   real secret (a private key, an API secret, a token, a password) or a value meant to be public (a public
   app or client identifier, a public key, a non-secret configuration constant). Many embedded strings are
   public by design; only a real credential is a finding, so classify before chasing reach.

2. **Check the package for embedded secrets.** Look through the app binary, its resources, and its
   configuration for a live credential compiled or bundled in. Anything shipped in the package is readable
   by anyone who has the app, so an embedded credential is exposed to every installer by definition.

3. **Check storage writes and their reach.** Look for sensitive data written to storage and determine who
   can read it: shared or world-readable storage any app or a connected reader reaches, versus the app's
   private sandbox, readable only by the app on a non-compromised device. Then check whether it is
   encrypted where the storage location warrants it.

4. **Check keystore placement.** Look for a secret or key material kept outside the platform keystore or
   secure hardware, protected instead by a weaker guard (a hardcoded key, an obfuscation, plain
   preferences), where the platform provides a stronger store the app declined to use.

5. **Check caches, logs, and backups.** Look for sensitive data written to a cache another party reaches, a
   log line that records a credential or personal data where a device-level or aggregated log reader sees
   it, and a backup or debug-export path that carries sensitive data off the device to a reader outside the
   sandbox.

6. **Confirm and record.** Confirm by showing a real credential is present and a party who should not read
   it can reach it. Kill the lead if the value is a public identifier or public key, not a secret, if the
   data sits only in the app's private sandbox with no cross-app or backup path reaching it, if it is
   already encrypted or held in the platform keystore, if a backup flag is set but the sandbox and platform
   already prevent the data from leaving, or if the exposed value is a short-lived, low-value token already
   scoped and rotated. Note that a backup flag alone is usually not the finding; the reachable secret is.
   Record the value, why it is a credential, and the reader that reaches it.

## Where mobile secrets leak

- **A string that looks like a key is often public.** A public app or client identifier is meant to ship;
  classify the value as credential or public before treating it as exposed.
- **Anything in the package is readable by every installer.** An embedded live credential is exposed by
  definition; there is no sandbox around a value compiled into the app.
- **The sandbox contains most storage writes.** Data in the app's private storage is readable only by the
  app on a non-compromised device; exposure needs a cross-app, backup, or device-reader path.
- **The keystore exists for a reason.** A secret kept outside the platform keystore under a weaker guard is
  weaker than it needs to be, even when no other party reaches it yet.
- **A backup flag alone is usually not the bug.** The finding is a reachable secret; a backup setting
  matters only when it actually carries a real credential off the device.

## Worked example (a confirm and a kill)

> **Confirm.** The app package embeds a private signing key used to authenticate to a backend; anyone who
> extracts the package obtains the key and can impersonate the app to the backend. **Confirmed** a live
> credential embedded in the shipped package, `critical`, remediation = remove the key from the package,
> move authentication to a server-mediated flow, and rotate the exposed key.
>
> **Kill.** A scanner flags a long string in the resources as a hardcoded key, but it is the app's public
> client identifier, documented as public and required to be shipped; it authenticates nothing on its own.
> **Killed**, `kill_reason` = "the flagged value is a public client identifier, not a credential, so its
> presence in the package exposes no secret."

## Rationalizations to reject

- *"There is a key hardcoded in the app."* -> Is it a credential or a public identifier? A public client
  identifier is meant to ship and authenticates nothing on its own.
- *"The app writes the token to a file."* -> Which storage, and who can read it? Data in the private sandbox
  is not exposed without a cross-app or backup path.
- *"It is not in the keystore, so it is exposed."* -> Keystore placement is a strength question; it is a
  finding when a reachable party can read the secret, and a hardening note otherwise.
- *"Backup is enabled, so the data leaks."* -> Does the backup actually carry a real credential off the
  sandbox? A backup flag without a reachable secret is usually not the bug.
- *"The log line has a token."* -> Is it a live credential and does a real log reader see it? A scoped,
  rotated, low-value token in a sandboxed log is weak, not the finding an embedded key is.

## Executing this in practice

You need the app package and its resources, the storage writes and the storage location each targets, the
keystore usage, the caches and log lines, and the backup and debug-export configuration. For each candidate,
decide whether it is a real credential and whether a party who should not read it can reach it. Reading the
value tells you whether it is a secret; reading the storage location and the backup and cross-app paths
tells you who can read it.

## Related

- `auditing-android-component-exposure` - the cross-app component reach that can become the reader for a
  file this skill finds; distinct from the value-and-storage question here.
- `auditing-mobile-deeplink-trust` - the URL-trust seam whose routed action or bridge might read the storage
  this skill audits; distinct from the data-at-rest question here.
- `hunting-non-human-identity-and-secret-reachability` - the server-side analogue for a credential's liveness
  and blast-radius once this skill establishes the secret is exposed.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the embedded or stored secret, sink = a party that
  can read it, evidence = the value shown to be a credential and the reader's reach.
