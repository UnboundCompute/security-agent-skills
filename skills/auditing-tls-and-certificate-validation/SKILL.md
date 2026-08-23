---
name: auditing-tls-and-certificate-validation
description: >-
  Audit client code for transport security that is disabled or defeated, so an attacker on the network
  path can intercept a connection the client believes is protected. Covers verification switched off (a
  trust-all setting, a permissive flag, an environment override), a custom trust manager or callback
  that returns success unconditionally, a hostname check that is skipped or always passes, acceptance of
  an expired or self-signed certificate through a swallowed error, certificate pinning that is absent
  where required or falls through to accept on failure, and a silent downgrade to cleartext when the
  handshake fails. Use when reviewing code that opens outbound TLS connections, configures an HTTP or
  socket client, or installs a custom trust store. An attacker in a man-in-the-middle position is the
  source, the client accepting a forged certificate is the sink, and validation that does not fail
  closed is the bug.
license: MIT
---

# Auditing TLS and certificate validation: when the client accepts a certificate it should reject

Transport security fails closed by default, so almost every real finding here is code that went out of
its way to weaken it. A client that turns verification off, installs a trust manager that approves any
chain, skips the hostname check, or swallows an expired-certificate error will complete a handshake
against an attacker's certificate and hand plaintext to whoever sits on the network path. This is not
about which cipher or protocol version is chosen, that is a different audit; it is about whether the
client accepts a certificate it should have rejected. You audit it by finding every outbound TLS client
and asking, for each, whether verification, hostname matching, and any required pinning are enforced and
fail closed on the reachable production path. The discipline is separating a genuine production bypass
from a test-only flag pointed at localhost.

## When to use

- Code opens an outbound TLS connection, or configures an HTTP, gRPC, database, or raw socket client.
- You see a verification flag, a custom trust manager, a hostname verifier, or a pinning routine.
- You want to know whether the client would accept a forged certificate from a network attacker.

## Scope check

Test interception only against endpoints and clients you own or are authorized to assess, and stand up
any man-in-the-middle only on a network in scope, intercepting a real connection captures live
credentials. Prove the bypass against a controlled endpoint, not a production one. If you can't name the
authorization, stop.

## The loop

1. **Find the outbound TLS clients and their targets.** Inventory the places the code opens a TLS
   connection or builds a client, and for each note the target (a real remote endpoint over the public
   network, or a localhost or in-cluster peer) and whether the path ships in production. The target and
   reachability decide whether a weakness is exploitable, so establish them before judging any flag.

2. **Check whether verification is switched off.** Look for verification globally disabled: a trust-all
   or permissive verification setting, a flag that skips certificate checks, or an environment override
   that disables rejection of unauthorized certificates. On a client that talks to a real remote endpoint
   with no test guard, this is the whole bug.

3. **Check the trust manager and the hostname check.** Look for a custom trust manager or verification
   callback that returns success unconditionally or has an empty body, and for a hostname verifier that
   accepts every name or a setting that disables hostname matching. A chain checked without the hostname
   accepts a valid certificate issued for a different host, which a network attacker readily obtains.

4. **Check swallowed errors and pinning fall-through.** Look for a custom validation callback that
   catches and ignores a specific failure class, accepting expired or self-signed certificates instead of
   failing. And where pinning is required, look for a pin that is never compared, a pin whose failure path
   falls through to accept, or pinning removed with no equivalent constraint on a high-value channel.

5. **Check downgrade to cleartext.** Look for a client that falls back to a plaintext scheme when the
   handshake fails or that permits a cleartext connection in configuration, turning a rejected certificate
   into a silent unencrypted send rather than a hard failure.

6. **Confirm and record.** Confirm by presenting a forged or mismatched certificate to the client from a
   controlled man-in-the-middle position and showing the connection completes and data flows. Kill the
   lead if the weakening is confined to a test file or a development-only build flag or environment guard
   pointed at localhost or a self-signed dev endpoint, if the transport is already mutually authenticated
   or a private in-process channel where the peer is pinned by other means, if a custom trust store is
   narrow (a single documented internal authority, not trust-all), if an absent pin is an intentional
   reliance on the platform trust store with a documented rotation policy, or if the insecure client is
   unreachable dead code or a documentation sample not built into the artifact. Record the client, the
   disabled check, and the target it reaches.

## Where transport validation leaks

- **The default is safe, so a bypass is deliberate.** Real findings are code that switched verification
  off, installed an approve-everything trust manager, or swallowed a validation error; look for the
  weakening, not the absence.
- **A valid chain for the wrong host is still an attack.** Skipping the hostname check accepts a
  certificate an attacker legitimately holds for another name; chain validity without name matching is not
  protection.
- **An off switch pointed at production is the finding; pointed at localhost it is noise.** The same flag
  is a critical bug on a public-endpoint client and a non-issue on a test fixture; the target decides.
- **Pinning that falls through to accept is decorative.** A pin computed and then ignored, or a failure
  path that proceeds, protects nothing; the comparison has to gate the connection.
- **A downgrade turns a rejection into a plaintext send.** Falling back to cleartext when TLS fails hands
  the attacker exactly what verification was meant to deny.

## Worked example (a confirm and a kill)

> **Confirm.** A client that posts credentials to a remote endpoint over the public network sets its
> transport to skip certificate verification, with no test or localhost guard on the reachable path. From
> a controlled man-in-the-middle position, presenting a self-signed certificate, the client completes the
> handshake and sends the credentials in the clear-equivalent channel. **Confirmed** disabled certificate
> verification on a production client enabling interception, `high` rising to `critical` for
> credential-bearing traffic, remediation = remove the override, verify the chain and the hostname against
> the default trust store, and fail closed.
>
> **Kill.** The same permissive setting appears only in a test file, guarded by a development build flag,
> and it points at a self-signed endpoint on localhost; the production client path constructs a
> default-verifying transport with no override. A network attacker on the production path faces full
> verification. **Killed**, `kill_reason` = "verification is disabled only in a localhost test fixture
> behind a dev-only flag; the shipping client verifies the chain and hostname and fails closed."

## Rationalizations to reject

- *"We disable verification only in development."* -> Is the flag guarded and pointed at localhost, and is
  the production path verifying? An unguarded override that ships is a production bypass.
- *"The certificate chain is checked, so it is fine."* -> Is the hostname checked? A valid chain for
  another host is one the attacker already holds; without the name check it is accepted.
- *"We pin the certificate."* -> Is the pin actually compared, and does a pin failure stop the connection?
  A pin that is ignored or falls through to accept is not pinning.
- *"It falls back to HTTP if HTTPS fails."* -> That fallback is the attack: break the handshake and the
  client sends in the clear. Fail the connection, do not downgrade it.
- *"It trusts our internal authority."* -> One documented internal authority is fine; a trust-all manager
  that accepts any chain is not. Confirm the trust store is narrow, not open.

## Executing this in practice

You need the outbound TLS clients and their targets, whether each path ships in production, the
verification and hostname settings, any custom trust manager or validation callback and what it returns,
the pinning routine and whether its result gates the connection, and any cleartext fallback. For each
client, decide whether validation is enforced and fails closed on the reachable path. Reading the client
tells you which checks are disabled; presenting a forged certificate from a controlled position tells you
whether the connection still completes.

## Related

- `finding-crypto-misuse` - the adjacent audit of primitive and mode selection (weak cipher, bad padding,
  reused nonce); this skill covers validation being disabled, not the algorithm chosen.
- `auditing-webhook-authenticity-and-callback-trust` - the application-layer trust decision on a caller
  and a callback target, distinct from the TLS chain-and-hostname decision audited here.
- `auditing-session-lifecycle-and-fixation` - the downgrade-to-cleartext shape here is the transport
  precondition that makes that skill's missing-secure-attribute cookie capturable on the wire.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = an attacker on the network path, sink = the
  client accepting a forged certificate, evidence = the connection completing against a controlled
  man-in-the-middle.
