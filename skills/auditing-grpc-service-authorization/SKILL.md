---
name: auditing-grpc-service-authorization
description: >-
  Audit a gRPC service for a method a caller can reach without the authorization the service assumes an
  interceptor enforces, after the interceptor coverage and the channel credentials are resolved. Covers
  authorization installed on the unary interceptor while the streaming chain omits it, a per-method
  authorization gap reachable at the wrong privilege, server reflection enabled in production exposing the
  full API, a plaintext channel with metadata trusted unverified, an absent message-size or recursion-depth
  limit inviting decode denial of service, and a transcoding gateway that does not apply the same auth
  filter as native gRPC. Use when reviewing service and interceptor registration, method handlers, and
  channel setup, not the certificate-validation mechanics the transport skill owns. A caller with forged or
  absent metadata is the source, a service method acting without an authorization check is the sink, and an
  interceptor that does not cover the method or stream is the bug.
license: MIT
---

# Auditing gRPC service authorization: the method the interceptor does not cover

A gRPC service usually centralizes authorization in an interceptor, and the bug is a method the
interceptor does not actually cover. The classic shape is an auth interceptor installed on the unary chain
while the streaming chain runs without it, so a streaming RPC is callable with no credentials. You audit
it by resolving which interceptors are registered and which methods and stream types they actually gate,
then checking each method the check assumes is covered. The other half is the channel: a plaintext or
insecure channel, metadata trusted without verification, missing message and recursion limits, and a
transcoding gateway that fronts the service without the same auth filter. Certificate-validation mechanics
belong to the transport skill; this skill owns whether a reachable method is authorized.

## When to use

- You are reviewing a gRPC server's service registration, interceptor chain, and method handlers.
- The service enforces authorization in an interceptor, or is fronted by a transcoding gateway.
- You want to know whether a caller can reach a method without the authorization the service assumes.

## Scope check

Audit only services you own or are authorized to assess, and call a method only against an endpoint in
scope, an unauthorized RPC drives real service state. Adjudicate on the registration and the handlers. If
you can't name the authorization, stop.

## The loop

1. **Resolve the interceptor coverage and channel credentials first.** Enumerate the registered
   interceptors and determine, for each, whether it applies to the unary chain, the streaming chain, or
   both, and read the channel setup for whether it requires transport credentials or runs insecure. A
   method's authorization lives in whatever interceptor actually wraps it, so establish that coverage map
   before judging any individual handler.

2. **Check the unary-versus-streaming gap.** Look for an authorization interceptor installed on the unary
   interceptor while the streaming interceptor chain omits it (or the reverse), so a whole class of RPCs
   runs without the check that the unary methods enforce.

3. **Check per-method authorization.** Look for a method reachable at the wrong privilege: an interceptor
   that authenticates a caller but does not check that this caller may call this method, letting a
   low-privilege identity invoke a high-privilege RPC or read another tenant's object.

4. **Check reflection and channel trust.** Look for server reflection registered in production, exposing
   the full method set to any caller for enumeration, and for a channel using insecure or plaintext
   credentials while trusting caller-supplied metadata (an identity header, a tenant id) without verifying
   it.

5. **Check decode limits and the transcoding gateway.** Look for a missing maximum-receive-message-size or
   recursion-depth bound, so a deeply nested or oversized message exhausts the decoder, and for a
   transcoding gateway (a REST-to-gRPC front) whose entry does not apply the same authentication and
   authorization filter as a native gRPC call, opening a second unguarded path to the method.

6. **Confirm and record.** Confirm by calling the method along the suspect path (a streaming RPC with no
   metadata, a low-privilege identity against a privileged method, the gateway route) and showing it acts
   without the authorization the service assumes. Kill the lead if a server interceptor enforces auth for
   both the unary and streaming chains so per-method handlers legitimately omit their own checks, if
   reflection is registered only behind an internal-only listener or a development build, if a service mesh
   enforces peer identity with mutual authentication so trusting the mesh identity is intended, if the
   maximum-message-size and depth limits are set, if metadata is re-validated by an interceptor rather than
   blindly trusted, or if the transcoding gateway applies the same auth filter with no REST-versus-gRPC
   asymmetry. Record the method, the path that reaches it unguarded, and the authorization it skips.

## Where gRPC authorization leaks

- **Coverage is per-chain, not per-service.** An interceptor on the unary chain does not gate streaming
  methods; resolve which chains each interceptor wraps before assuming a method is covered.
- **Authentication is not authorization.** An interceptor that proves who the caller is, but not that they
  may call this method, leaves a per-method gap that reads or writes across a privilege or tenant boundary.
- **Reflection in production is free enumeration.** Server reflection hands any caller the whole API surface;
  behind an internal listener or a dev build it is not exposed, in production it is a map for the attacker.
- **An insecure channel makes metadata a lie.** Trusting a caller-supplied identity or tenant header on a
  plaintext or unauthenticated channel lets the caller assert any identity; the metadata has to be verified.
- **The transcoding gateway is a second door.** A REST-to-gRPC front that skips the gRPC auth filter reaches
  the method by another path; the asymmetry, not the native call, is the bug.

## Worked example (a confirm and a kill)

> **Confirm.** A server registers an auth interceptor with the unary interceptor option only; the streaming
> interceptor chain has none. A server-streaming RPC that returns account records is callable with no
> metadata and streams the records to an unauthenticated caller. **Confirmed** a streaming method reachable
> without authorization because the interceptor covers only the unary chain, `high` rising to `critical`
> for cross-tenant data, remediation = install the auth interceptor on the streaming chain as well (or a
> single interceptor covering both) and re-check per-method privilege.
>
> **Kill.** The same server installs the auth interceptor on both the unary and streaming chains, verifies
> the caller identity and its per-method privilege, sets the maximum-message-size and depth limits, and
> registers reflection only on an internal loopback listener. The streaming RPC rejects the metadata-less
> call. **Killed**, `kill_reason` = "a single auth interceptor covers unary and streaming, per-method
> privilege is checked, and reflection is not exposed; no uncovered method or stream."

## Rationalizations to reject

- *"The interceptor handles auth."* -> On which chain? An interceptor on the unary option does not wrap
  streaming methods; confirm the coverage includes every chain and stream type.
- *"The caller is authenticated."* -> Authenticated to do what? Without a per-method privilege check, a
  low-privilege identity can call a high-privilege method.
- *"Reflection is convenient."* -> In production it enumerates the whole API for an attacker; register it
  only behind an internal listener or a development build.
- *"We read the tenant from metadata."* -> On what channel, verified how? On an insecure channel, unverified
  metadata is attacker-asserted; verify the identity, do not trust the header.
- *"The gateway is just a proxy."* -> Does it apply the same auth filter as the native call? A transcoding
  front that skips it is a second unguarded path to the method.

## Executing this in practice

You need the service and interceptor registration (which interceptors, which chains), each method handler
and whether it relies on the interceptor for authorization, the channel credential setup and any metadata
it trusts, the message-size and recursion-depth limits, the reflection registration and its listener, and
any transcoding gateway and its filter. For each method, decide whether a caller can reach it without the
authorization the service assumes. Reading the registration tells you what is covered; calling the method
tells you what is reachable.

## Related

- `auditing-websocket-connection-trust` - the sibling wire-protocol audit where trust is established once at
  the handshake; here the analogous gap is trust assumed in an interceptor that misses a method or stream.
- `auditing-tls-and-certificate-validation` - owns the certificate-validation mechanics for the channel;
  this skill covers whether the channel requires credentials and whether the method is authorized, not the
  chain-and-hostname check.
- `hunting-broken-object-level-authorization` - the object-authorization analogue; a gRPC method that reads
  an object without checking the caller owns it has the same missing-scope shape at the method boundary.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = a caller with forged or absent metadata, sink = a
  service method acting without an authorization check, evidence = the interceptor coverage map and the path
  that reaches the method unguarded.
