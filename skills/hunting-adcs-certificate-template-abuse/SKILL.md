---
name: hunting-adcs-certificate-template-abuse
description: >-
  Hunt privilege escalation through Active Directory Certificate Services template and enrollment
  misconfiguration, where a low-privileged principal can enroll for a certificate that authenticates as a
  more privileged account, because a template lets the enrollee supply the subject, allows a client
  authentication use, has weak enrollment permissions, or the certificate authority accepts a requester-set
  identity. Use when a certificate authority issues templates for domain authentication and a standard user
  can enroll. Covers enrollee-supplied subject alternative names, overly broad enrollment rights, dangerous
  template flags, and requester-controlled identity. The low-privileged enrollment for an identity-bearing
  certificate is the source, the certificate authority issuing it is the sink, and authenticating as a more
  privileged account with that certificate is the bug.
license: MIT
---

# Hunting ADCS certificate template abuse: when a standard user enrolls as a domain admin

A certificate authority in a Windows domain issues certificates that clients then use to authenticate, and
the identity a certificate carries is whatever the authority put in it. That is safe only when the authority
decides the identity from the authenticated enrollee, never from what the enrollee asked for. A template that
lets the requester supply the subject alternative name, combined with a use that permits client
authentication and enrollment rights granted to ordinary users, lets a standard user request a certificate
naming a domain administrator and then authenticate as that administrator. Other template settings weaken the
same boundary: a flag that carries the requester's identity into the issued certificate, or enrollment
permissions loose enough that any user obtains a template meant for privileged roles. The bug is a certificate
whose identity the low-privileged enrollee, not the authority, decided. You hunt these by reading each
template's enrollment rights, subject source, and permitted uses.

## When to use

- A certificate authority issues templates usable for domain authentication and standard users may enroll.
- A template lets the enrollee supply the subject or a subject alternative name rather than deriving it.
- Enrollment permissions or template flags may let an ordinary user obtain a privileged-identity certificate.

## Scope check

Test certificate template abuse only in directories you own or are authorized to assess, on non-production or
a lab domain, enrolling only for an identity you are permitted to test with and never authenticating as a real
privileged principal outside the authorization. A confirmed abuse yields authentication as a privileged
account, so keep every probe within scope. If you can't name the authorization, stop.

## The loop

1. **Establish where the certificate's identity actually comes from first.** For each template a standard user
   can enroll in, determine whether the authority derives the subject and any alternative name from the
   authenticated enrollee, or whether the enrollee supplies them, and whether the resulting certificate
   permits client authentication. This is the false-positive killer: a template that fixes the identity from
   the enrollee and forbids requester-supplied names cannot mint a privileged identity, however broad its
   enrollment rights. Name the identity source before crafting a request.

2. **Enumerate enrollable templates and their permissions.** List the templates the authority publishes, the
   principals granted enroll and autoenroll rights on each, and whether the granted set includes ordinary
   users or broad groups. Templates enrollable by low-privileged principals are the candidates.

3. **Check the subject and alternative-name source.** Determine which templates let the enrollee specify the
   subject or a subject alternative name in the request rather than the authority setting it from the
   directory. A template that honors a requester-supplied alternative name lets the requester name any
   principal, including a privileged one.

4. **Check the permitted uses and dangerous flags.** Determine whether the certificate permits client
   authentication or a use that authenticates to the domain, and whether any template flag carries the
   requester's chosen identity into the issued certificate or removes an approval that would otherwise gate
   issuance. A privileged identity is only usable if the certificate can authenticate.

5. **Check the issuance gate.** Determine whether issuance requires manager approval, authorized signatures,
   or additional constraints that would stop a standard user from obtaining the certificate unattended. A
   template that issues immediately on request removes the human check that would otherwise catch the abuse.

6. **Confirm and record.** Confirm by enrolling as a low-privileged principal for a certificate naming a test
   identity you are permitted to assume and authenticating with it on an isolated lab domain, observing the
   privileged context. Kill the lead if the authority sets the identity from the enrollee, if requester-
   supplied names are rejected, if the certificate cannot authenticate to the domain, or if issuance requires
   an approval the standard user cannot satisfy. Record the template, the enrollment right, the identity
   source, and the authentication observed, or set a `kill_reason`.

## Where certificate template abuse leaks

- **The identity source is the finding.** The whole bug is a certificate whose identity the enrollee decided;
  a template that derives identity from the authenticated requester cannot be abused this way.
- **Requester-supplied alternative names name anyone.** A template that honors a subject alternative name from
  the request lets a standard user put a privileged principal's name in their own certificate.
- **Enrollment rights that are too broad open the door.** A privileged-purpose template enrollable by ordinary
  users hands them a certificate they were never meant to obtain.
- **Client authentication makes the identity usable.** A minted privileged identity matters only when the
  certificate can authenticate to the domain; the permitted use decides whether the identity is live.
- **Missing approval removes the human check.** A template that issues immediately, without manager approval or
  authorized signatures, lets the abuse complete unattended.

## Worked example (a confirm and a kill)

> **Confirm.** A template enrollable by all authenticated users lets the enrollee supply a subject alternative
> name and issues certificates valid for client authentication, with no manager approval. A low-privileged
> principal enrolls for a certificate naming a test privileged identity and authenticates to the lab domain as
> that identity on an isolated instance. **Confirmed** privilege escalation through certificate template
> abuse, `high`, remediation = derive the subject and alternative name from the authenticated enrollee rather
> than the request, restrict enrollment on authentication-capable templates to the roles that need them, and
> require manager approval for sensitive templates.
>
> **Kill.** The same authority publishes only templates that set the subject from the enrolling account,
> reject requester-supplied alternative names, and gate any authentication-capable template behind manager
> approval and enrollment rights limited to a privileged group. A standard user cannot obtain a certificate
> naming another principal. **Killed**, `kill_reason` = "the authority sets identity from the authenticated
> enrollee and rejects requester-supplied names; no low-privileged enrollment yields a certificate that
> authenticates as a more privileged account."

## Rationalizations to reject

- *"Only administrators can enroll."* -> Read the actual enrollment and autoenroll rights on each template; a
  privileged-purpose template is frequently left enrollable by all authenticated users.
- *"The subject is set by the CA."* -> Confirm it for this template; a template that honors a requester-supplied
  alternative name lets the enrollee override the identity regardless of the primary subject.
- *"It is only a certificate, not a login."* -> A certificate permitting client authentication is a domain
  credential; the abuse authenticates as the named principal, which is a login.
- *"Manager approval will catch it."* -> Only if the template requires it; many authentication-capable
  templates issue immediately, so confirm the approval gate exists rather than assuming it.
- *"We would notice a rogue admin certificate."* -> The certificate names a legitimate-looking principal and
  is issued by the trusted authority; nothing looks rogue without inspecting the enrollment.

## Executing this in practice

You need every template a low-privileged principal can enroll in, the enrollment and autoenroll rights on
each, whether the subject and alternative name come from the enrollee or the directory, whether the
certificate permits client authentication, and whether issuance requires approval. For each, decide whether a
standard user can obtain a certificate naming a more privileged principal that authenticates to the domain.
Reading the template settings and permissions settles most leads; enrolling for a permitted test identity and
authenticating with it on an isolated lab domain settles the rest.

## Related

- `hunting-active-directory-acl-abuse-paths` - a permission to modify a template or its enrollment rights is
  an object-permission abuse that skill treats, feeding the misconfiguration this one exploits.
- `hunting-ntlm-coercion-and-relay` - a coerced authentication relayed to a certificate authority's
  enrollment interface obtains a certificate for the coerced account, joining relay to template abuse.
- `auditing-tls-and-certificate-validation` - a certificate authenticating as the wrong principal is a
  validation-trust question that skill audits from the verifier's side.
- `hunting-kerberos-and-ad-delegation-abuse` - a certificate that authenticates to the domain feeds the
  ticket-based escalation that skill pursues once a credential is in hand.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the low-privileged enrollment for an identity-bearing
  certificate, sink = the certificate authority issuing it, evidence = authentication as a more privileged
  account with the issued certificate on an isolated lab domain.
