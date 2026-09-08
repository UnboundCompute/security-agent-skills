---
name: hunting-mobile-tls-pinning-and-trust-gaps
description: >-
  Hunt transport trust gaps in a mobile app, where the app accepts a network position it should reject,
  because it trusts user-added certificate authorities, allows cleartext or mixed connections, disables or
  misapplies certificate validation, or pins only some connections so an unpinned or fallback path lets a
  network attacker read or alter traffic the app treats as secure. Use when reviewing how a mobile app
  establishes and validates its network connections and whether every sensitive connection resists an
  intercepting network position. Covers user-trusted anchors, cleartext and mixed connections, disabled or
  permissive validation, and partial or bypassable pinning. The intercepting network position the app fails
  to reject is the source, the connection the app treats as trusted is the sink, and reading or altering
  supposedly secure traffic is the bug.
license: MIT
---

# Hunting mobile TLS pinning and trust gaps: when the app trusts a network it should not

A mobile app talks to its backend over connections it treats as private, and that privacy depends entirely on
the app rejecting any network position that would let someone read or alter the traffic. The app does that by
validating the server's certificate against trusted anchors, refusing cleartext, and, for its most sensitive
connections, pinning to a known certificate or key so even a trusted-but-wrong authority is rejected. The gap
is any place the app accepts a network position it should reject: it trusts certificate authorities the user
or a device profile added, it allows a cleartext or mixed connection, it disables or loosens validation for
convenience, or it pins some connections but leaves an unpinned or fallback path a network attacker can take.
The bug is not the absence of pinning everywhere; it is a sensitive connection the app treats as secure that
an intercepting network position can actually read or alter. You hunt these by listing every connection and
checking whether each resists a network attacker.

## When to use

- A mobile app makes network connections it treats as private and you can observe its trust behavior.
- The app may trust user-added anchors, allow cleartext, or loosen certificate validation.
- Some sensitive connections may be unpinned, or pinning may have a bypassable fallback.

## Scope check

Hunt transport trust gaps only on apps and networks you own or are authorized to assess, on test devices and
accounts, interposing only on a network you control and never on real users' traffic. A confirmed gap exposes
supposedly secure traffic, so keep every probe within scope. If you can't name the authorization, stop.

## The loop

1. **Establish whether each sensitive connection rejects an intercepting position first.** For each connection
   the app treats as private, determine whether it validates the certificate against trusted anchors, refuses
   cleartext, and, where required, pins to a known certificate or key, so an intercepting network position is
   rejected, or whether some trust gap lets that position succeed. This is the false-positive killer: a
   connection that validates strictly and pins where it matters cannot be read by a network attacker, so its
   presence is not a finding. Name the accepted network position before claiming interception.

2. **Enumerate the app's connections and their sensitivity.** List the connections the app makes, the data
   each carries, and which are sensitive enough that reading or altering them would matter. Note which the app
   treats as secure and relies on for authentication or private data.

3. **Check the trust anchors and validation.** Determine whether the app trusts only the system anchors it
   should, or also user-added authorities or a device profile's anchors, and whether it validates the
   certificate fully, hostname included, or disables or loosens validation. A user-trusted anchor or disabled
   validation lets an interposed certificate be accepted.

4. **Check for cleartext and mixed connections.** Determine whether any sensitive connection uses cleartext or
   falls back to it, or mixes secure and insecure content, so part of the traffic is readable regardless of
   the certificate handling.

5. **Check the pinning coverage and fallback.** For connections that pin, determine whether every sensitive
   connection is pinned or only some, and whether pinning has a fallback, a catch-all path, an unpinned
   subdomain, or a bypass, that a network attacker can steer the app onto. Partial or bypassable pinning
   leaves the unpinned path open.

6. **Confirm and record.** Confirm by interposing on a network you control and reading or altering a sensitive
   connection the app treats as secure, on a test device with a test account. Kill the lead if every sensitive
   connection validates strictly, refuses cleartext, and pins where required with no bypassable fallback, or if
   the interception requires a trust the attacker cannot obtain. Record the connection, the accepted network
   position, the trust gap, and the interception observed, or set a `kill_reason`.

## Where transport trust leaks

- **The accepted network position is the finding.** A secure connection is expected; the bug is one the app
  treats as secure that an intercepting position can read or alter. Name the position and how it is accepted.
- **User-added anchors defeat validation.** An app that trusts user-added or profile-added authorities accepts
  a certificate an intercepting proxy presents, so validation passes against the wrong anchor.
- **Cleartext and fallback expose traffic directly.** A sensitive connection that uses or falls back to
  cleartext is readable regardless of certificate handling, and mixed content leaks the insecure part.
- **Partial pinning leaves an open path.** Pinning some connections but not others, or an unpinned subdomain
  or catch-all, lets a network attacker steer the app onto the unpinned path.
- **Bypassable pinning is not pinning.** A pin with a fallback to system validation, or one that can be
  disabled by a setting or a downgrade, does not resist an interposed trusted-but-wrong certificate.

## Worked example (a confirm and a kill)

> **Confirm.** The app pins its primary API connection but makes a sensitive secondary connection to an
> unpinned host that validates only against system and user-added anchors. Interposing on a controlled network
> with a user-added anchor lets that secondary connection be read and altered, exposing data the app treats as
> secure, on a test device. **Confirmed** transport trust gap through partial pinning and user-anchor trust,
> `high`, remediation = pin every sensitive connection to a known certificate or key, trust only system
> anchors for those connections, refuse cleartext and fallback, and remove any unpinned sensitive path.
>
> **Kill.** Every sensitive connection validates fully against system anchors only, ignores user-added
> authorities, refuses cleartext with no fallback, and pins to a known key with no bypassable path. An
> interposed certificate is rejected on every sensitive connection and no traffic is readable. **Killed**,
> `kill_reason` = "all sensitive connections validate strictly against system anchors, refuse cleartext, and
> pin without a fallback; the intercepting position is rejected everywhere it matters."

## Rationalizations to reject

- *"We use HTTPS everywhere."* -> HTTPS validates against whatever anchors the app trusts; if it trusts
  user-added authorities or loosens validation, an interposed certificate still passes.
- *"The main connection is pinned."* -> Coverage is the point; a sensitive secondary or fallback connection
  left unpinned is the path a network attacker takes, so check every connection, not the primary.
- *"Only a rooted device trusts extra anchors."* -> A device profile or a user can add an anchor without root;
  confirm the app ignores user-added anchors for sensitive connections rather than assuming they cannot exist.
- *"Cleartext is only for non-sensitive calls."* -> Confirm which data each cleartext connection carries and
  that none falls back to cleartext; a fallback exposes the traffic the pin was meant to protect.
- *"Pinning can be turned off for debugging."* -> A pin with a switch or a fallback is bypassable; confirm the
  production path cannot be downgraded onto system validation.

## Executing this in practice

You need every connection the app makes with its sensitivity, the trust anchors and validation each uses,
whether any use or fall back to cleartext, and which are pinned with what fallback. For each sensitive
connection, decide whether an intercepting network position is rejected. Reading the trust configuration and
observing the app's behavior settles most leads; interposing on a controlled network and reading or altering a
sensitive connection on a test device settles the rest.

## Related

- `auditing-tls-and-certificate-validation` - the general certificate-validation audit, of which mobile
  pinning and anchor trust are the app-side application.
- `auditing-mobile-webview-bridge-exposure` - content loaded over a connection an attacker can intercept
  reaches the WebView bridge, so a transport gap feeds that exposure.
- `hunting-mobile-secret-and-storage-exposure` - a token read from intercepted traffic joins the secret
  exposure that skill pursues from the storage side.
- `auditing-mobile-backend-and-firebase-exposure` - an intercepted or unpinned backend connection reveals the
  backend endpoints and keys that skill audits.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the intercepting network position the app fails to
  reject, sink = the connection the app treats as trusted, evidence = reading or altering a sensitive
  connection on a controlled network with a test device.
