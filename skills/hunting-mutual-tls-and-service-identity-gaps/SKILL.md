---
name: hunting-mutual-tls-and-service-identity-gaps
description: >-
  Hunt for gaps in how a service establishes and verifies the identity of the peer calling it: a mutual-TLS
  endpoint that requests a client certificate but does not require or verify it, verification that checks the
  certificate chains to a trusted authority but never checks which identity it names, a trust anchor broad
  enough that any certificate it issued is accepted as any service, and an identity derived from a spoofable
  attribute (a header, a source IP) instead of the verified certificate. Covers service-to-service calls where
  mutual TLS or a certificate is meant to prove which service is calling. Use when a service authorizes callers
  by their identity and mutual TLS or a client certificate is the proof. The unverified or misbound peer is the
  source, the called service it authenticates to is the sink, and the missing certificate requirement, identity
  check, or trust-anchor scoping that admits it is the bug.
license: MIT
---

# Hunting mutual TLS and service identity gaps: proving which service is on the other end

Service-to-service authentication with mutual TLS rests on a chain of checks, and skipping any one of them
leaves the caller's identity unproven while the connection still looks encrypted and authenticated. The
endpoint has to actually require a client certificate, not merely request one and proceed without it. It has
to verify the certificate is valid and chains to a trusted authority. Crucially, it then has to check which
identity the certificate names and that this identity is authorized for the call, verifying the chain proves
the certificate is legitimate, not that it belongs to the right service. And the trust anchor matters: if the
service accepts any certificate issued by a broad authority, then any peer that authority ever issued a
certificate to can authenticate as any service. When identity is instead taken from a header or source IP, it
is spoofable and the certificate is decorative. The hunt is for the missing link in that chain. You hunt this
by presenting the wrong certificate, no certificate, and a valid-but-wrong-identity certificate and seeing
what authenticates.

## When to use

- Services authenticate each other with mutual TLS or client certificates to prove which service is calling.
- An endpoint may request a client certificate without requiring it, or verify the chain but not the identity.
- Service identity may be derived from a header or source IP rather than the verified certificate.

## Scope check

Test service identity only on systems you own or are authorized to assess, on non-production endpoints.
Presenting certificates and connecting attempts real authentication between services, so use non-production
identities and never authenticate into a real service path. If you can't name the authorization, stop.

## The loop

1. **Establish the intended peer identity first.** Name, for each endpoint, exactly which service identities may
   call it and how that identity is proven: a client certificate naming the specific service, chaining to a
   narrowly scoped trust anchor. This is the false-positive killer: an endpoint that requires a client
   certificate, verifies its chain to a scoped anchor, checks the named identity against an allowlist, and
   authorizes on that verified identity is correct. Name the intended identity, then test each check.

2. **Test whether the certificate is required.** Determine whether the endpoint requires a valid client
   certificate or merely requests one and proceeds if none is presented. Connect without a client certificate
   and confirm the call is refused. An endpoint that requests but does not require a certificate authenticates
   callers who present none.

3. **Test chain verification.** Present a certificate that does not chain to the trusted authority, an expired
   one, and one with a broken chain, and confirm each is rejected. An endpoint that accepts an untrusted or
   invalid chain is not verifying the certificate at all, so any self-issued certificate authenticates.

4. **Test identity checking beyond the chain.** Present a certificate that is valid and chains to the trusted
   authority but names a different service than the one authorized for the call. Confirm the endpoint checks the
   named identity, not just chain validity. An endpoint that accepts any trusted-authority certificate treats
   chain validity as identity, so any legitimately issued certificate authenticates as any service.

5. **Check trust-anchor scope and identity source.** Determine how broad the trust anchor is: if the service
   accepts any certificate issued by a wide public or shared authority, the anchor is too broad and any peer
   that authority issued to can authenticate. Also confirm identity is derived from the verified certificate,
   not from a header or source IP a caller can set. A spoofable identity source makes the certificate
   decorative.

6. **Confirm and record.** Confirm by authenticating with no certificate, an untrusted certificate, or a valid
   certificate naming the wrong service, or by spoofing the header or IP the identity is read from, on
   non-production endpoints and without entering a real service path. Kill the lead if the endpoint requires a
   certificate, verifies the chain to a scoped anchor, checks the named identity against an allowlist, and
   authorizes on the verified certificate. Record the unverified or misbound peer, the called-service sink, and
   the missing certificate requirement, identity check, or trust-anchor scoping.

## Where service identity leaks

- **Requested but not required certificates.** An endpoint that proceeds when no client certificate is presented
  authenticates callers with no proof at all.
- **Chain validity mistaken for identity.** Accepting any certificate that chains to the trusted authority,
  without checking which identity it names, lets any issued certificate be any service.
- **An over-broad trust anchor.** A wide or shared authority means any peer it ever issued a certificate to can
  authenticate; the anchor must be scoped to the intended issuers.
- **Identity from a spoofable attribute.** Deriving the caller's identity from a header or source IP instead of
  the verified certificate makes the certificate decorative and the identity forgeable.
- **Encrypted is not authenticated.** A TLS connection can be established and encrypted while the peer's
  identity is never actually verified.

## Worked example (a confirm and a kill)

> **Confirm.** An internal endpoint uses mutual TLS and verifies that the client certificate chains to the
> organization's shared certificate authority, but it does not check which service the certificate names. A
> service holding a legitimately issued certificate for an unrelated, low-privilege workload connects and is
> authenticated as an authorized caller because its certificate chains to the trusted authority. **Confirmed**
> service-identity bypass via chain-validity-as-identity, `high`, remediation = check the specific identity the
> certificate names against an allowlist of authorized services, scope the trust anchor to the intended issuers,
> and authorize on the verified certificate identity rather than mere chain validity.
>
> **Kill.** The endpoint requires a client certificate, rejects connections without one, verifies the chain to a
> narrowly scoped trust anchor, checks the identity the certificate names against the allowlist of services
> authorized for the call, and derives the caller identity solely from the verified certificate rather than any
> header or IP. A missing, untrusted, or wrong-identity certificate is refused. **Killed**, `kill_reason` =
> "certificate required, chain verified to a scoped anchor, named identity checked against an allowlist, and
> identity taken from the verified certificate; no unverified or misbound peer authenticates."

## Rationalizations to reject

- *"We use mutual TLS."* → Confirm the certificate is required and its named identity checked; requesting a
  certificate or verifying only the chain is not proving which service called.
- *"The certificate is valid and trusted."* → Chain validity is not identity; confirm the endpoint checks which
  service the certificate names, or any issued certificate is any service.
- *"They are all issued by our CA."* → A broad shared authority means every peer it issued to can authenticate;
  scope the anchor and check the specific identity.
- *"We read the service name from a header."* → A header is caller-set and spoofable; derive identity from the
  verified certificate, not from what the caller claims.
- *"The connection is encrypted."* → Encryption is not authentication; a TLS session can be established without
  the peer's identity ever being verified.

## Executing this in practice

You need, per endpoint, whether a client certificate is required, how the chain is verified and to what anchor,
whether the named identity is checked against an allowlist, and whether identity comes from the certificate or a
spoofable attribute. For each, connect with no certificate, an untrusted one, and a valid-but-wrong-identity
one. Reading the mutual-TLS configuration shows the intended checks; a wrong or absent certificate that
authenticates shows whether they hold.

## Related

- `auditing-service-mesh-mtls-and-authz-trust` - meshes automate mutual TLS; this skill is the underlying
  identity-verification question their authorization decisions depend on.
- `auditing-tls-and-certificate-validation` - the certificate-validation mechanics shared with server-side TLS;
  chain and trust-anchor checks apply in both directions.
- `mapping-service-account-impersonation-chains` - once a peer identity is established, what it can reach and
  impersonate is the paired reachability question.
- `auditing-grpc-service-authorization` - gRPC calls often carry mutual-TLS identity; verifying the peer and
  authorizing the method are the two halves.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the unverified or misbound peer, sink = the called
  service it authenticates to, evidence = the missing certificate requirement, identity check, or trust-anchor
  scoping.
