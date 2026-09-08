---
name: hunting-hybrid-app-bundle-and-config-exposure
description: >-
  Hunt secrets and abusable configuration shipped inside a mobile app bundle, where the installable package
  carries hardcoded API keys, backend credentials, signing or encryption material, private endpoints, or
  feature and debug flags in its code, resources, web assets, or configuration files, so anyone who unpacks
  the distributed app recovers them, because the bundle is treated as private when a distributed binary is
  fully readable. Use when reviewing what a shipped mobile app package contains, including a hybrid app's
  embedded web assets and configuration. Covers hardcoded credentials and keys, embedded private endpoints,
  bundled web-asset secrets, and shipped debug or feature flags. The secret or abusable setting shipped in the
  bundle is the source, unpacking the distributed app is the sink, and recovering usable material or a
  privileged toggle from the package is the bug.
license: MIT
---

# Hunting hybrid app bundle and config exposure: when the shipped app carries its own secrets

A mobile app is distributed as a package that lands on every user's device, and anyone can unpack it: the
code, the resources, the embedded web assets of a hybrid app, and the configuration files are all readable.
That is fine for everything a client is meant to hold, and a leak for anything that was meant to stay secret.
Developers ship secrets into the bundle anyway, a backend API key hardcoded in the code, a credential in a
resource file, signing or encryption material in the assets, a private internal endpoint in a configuration,
or a debug or feature flag left enabled that unlocks privileged behavior. Because the app is treated as though
its contents were private, these are shipped as if hidden, when in fact a distributed binary is fully
readable. The bug is not that the app has configuration; it is a secret or an abusable setting that anyone who
unpacks the package can recover and use. You hunt these by unpacking the shipped bundle and reading everything
in it as an attacker would.

## When to use

- A mobile app is distributed as a package whose code, resources, and configuration you can unpack.
- A hybrid app embeds web assets and configuration files that may carry secrets or endpoints.
- The bundle may contain hardcoded keys, credentials, private endpoints, or debug and feature flags.

## Scope check

Hunt bundle exposure only on apps you own or are authorized to assess, unpacking a package you are permitted
to analyze and proving any recovered secret is usable against test infrastructure rather than production. A
recovered key or credential authenticates or unlocks a real service, so keep every probe within scope and
treat recovered material as sensitive. If you can't name the authorization, stop.

## The loop

1. **Establish whether each recovered value is a live secret or an intended client value first.** For each
   secret-looking string, endpoint, or flag in the bundle, determine whether it is material meant to stay
   secret, a backend key, a credential, signing material, a private endpoint, or a privileged toggle, or a
   value a client is legitimately meant to hold, a public identifier or a scoped client token that grants
   nothing on its own. This is the false-positive killer: a public client identifier or a properly scoped
   token that authorizes nothing sensitive is not a finding even though it is in the bundle. Name what the
   value grants before claiming exposure.

2. **Unpack the bundle and enumerate its contents.** Extract the distributed package and list its code,
   resources, embedded web assets, and configuration files. For a hybrid app, include the bundled web assets
   and their configuration, which often carry endpoints and keys.

3. **Search for credential and key material.** Read the code, resources, and configuration for hardcoded API
   keys, backend credentials, signing or encryption keys, and tokens. Note each and what service or capability
   it would authenticate to.

4. **Search for private endpoints and settings.** Read the configuration and assets for internal or private
   endpoints, hosts, and service addresses the app talks to, and for feature or debug flags that unlock
   behavior a production user should not have.

5. **Check what each recovered value grants.** For each secret, endpoint, or flag, determine whether it is
   usable, whether the key authenticates to a live service, the endpoint is reachable, or the flag actually
   enables privileged behavior when set, so the exposure has real impact rather than being an inert artifact.

6. **Confirm and record.** Confirm by using a recovered secret or setting against test infrastructure you are
   permitted to reach, showing the key authenticates, the private endpoint responds, or the flag unlocks
   behavior, without acting against production. Kill the lead if every recovered value is a public client
   identifier or a scoped token that grants nothing sensitive, if endpoints are public, and if flags do not
   enable privileged behavior. Record the value, where in the bundle it was found, what it grants, and the
   proof, or set a `kill_reason`.

## Where bundle exposure leaks

- **What the value grants is the finding.** A distributed bundle is fully readable; the bug is a value in it
  that grants something sensitive, not merely a string that looks secret. Name what it unlocks.
- **Hardcoded backend keys are live.** An API key or credential compiled into the code or a resource
  authenticates to the backend for anyone who extracts it, so the client is not a safe place to keep it.
- **Hybrid web assets carry endpoints and keys.** A hybrid app's embedded web assets and their configuration
  often hold service endpoints and keys in plain view, unpacked with the rest of the bundle.
- **Private endpoints are disclosed by shipping them.** An internal host or service address in the
  configuration is revealed to anyone who reads the bundle, mapping the backend for an attacker.
- **Shipped debug and feature flags unlock behavior.** A debug mode or feature flag left enabled, or settable,
  in the bundle can unlock privileged behavior a production user should never reach.

## Worked example (a confirm and a kill)

> **Confirm.** A hybrid app's embedded web-asset configuration contains a backend API key and a private
> administrative endpoint. Extracting the bundle recovers both, and the key authenticates to the endpoint on
> test infrastructure, granting access a client should not have. **Confirmed** secret and private-endpoint
> exposure in the shipped bundle, `high`, remediation = remove backend keys and private endpoints from the
> bundle, have the client obtain scoped short-lived tokens from an authenticated service, and gate privileged
> endpoints server-side rather than by a shipped key.
>
> **Kill.** The same bundle contains only a public client identifier and a token scoped so it authorizes
> nothing without a server-side session, the endpoints in the configuration are the public API, and no debug
> or feature flag in the bundle enables privileged behavior. Every recovered value grants nothing sensitive on
> its own. **Killed**, `kill_reason` = "the bundle carries only a public identifier and a scoped token that
> authorizes nothing alone, public endpoints, and no privileged flag; nothing recovered from the package grants
> sensitive access."

## Rationalizations to reject

- *"The key is inside the compiled app."* -> A compiled app is unpacked and read routinely; compilation is not
  concealment, so a key in the binary is a published key.
- *"It is only used by our own app."* -> Anyone who extracts the bundle can use the same key or endpoint; the
  client cannot enforce that only the genuine app holds it.
- *"That endpoint is internal."* -> Shipping its address in the bundle discloses it, and if it is reachable
  the disclosure maps the backend; confirm reachability rather than assuming internal means hidden.
- *"The debug flag is off by default."* -> If it is settable in the bundle or the configuration, an attacker
  on a controlled device can turn it on; confirm the privileged behavior is gated server-side, not by a flag.
- *"It is just an obfuscated string."* -> Obfuscation slows recovery, it does not prevent it; treat an
  obfuscated live secret as exposed.

## Executing this in practice

You need the unpacked bundle with its code, resources, embedded web assets, and configuration, every
secret-looking value, endpoint, and flag in it, and for each what it authenticates to or unlocks. For each,
decide whether it is a live secret, a reachable private endpoint, or a privileged flag, or an inert client
value. Reading the unpacked contents settles most leads; using a recovered value against permitted test
infrastructure to show it grants sensitive access settles the rest.

## Related

- `hunting-mobile-secret-and-storage-exposure` - the runtime and at-rest secret hunt, of which a secret
  compiled into the shipped bundle is the build-time case.
- `auditing-mobile-backend-and-firebase-exposure` - a backend key or endpoint recovered from the bundle feeds
  directly into the backend exposure that skill audits.
- `hunting-firmware-secrets-and-debug-interfaces` - the embedded-device analogue, where secrets and debug
  interfaces ship inside a readable firmware image.
- `hunting-mobile-tls-pinning-and-trust-gaps` - a private endpoint disclosed by the bundle is reached over a
  connection whose transport trust that skill audits.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the secret or abusable setting shipped in the bundle,
  sink = unpacking the distributed app, evidence = a recovered secret authenticating, a private endpoint
  responding, or a flag unlocking behavior on test infrastructure.
