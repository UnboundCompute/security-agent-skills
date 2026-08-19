---
name: testing-smtp-smuggling-and-email-spoofing
description: >-
  Test a mail setup for sender spoofing that survives authentication: SPF, DKIM, and DMARC
  records that exist but do not align or enforce, subdomains left unprotected, and the
  end-of-data desync known as SMTP smuggling, where an inbound and an outbound mail server
  disagree on where one message ends so a second message with a forged, auth-passing sender
  is smuggled in. Covers policy present but not enforced, alignment gaps between the
  envelope and header sender, missing subdomain policy, open relay, and inconsistent
  message-boundary parsing between hops. Use when auditing a domain's mail authentication or
  a mail server's boundary handling. The crafted or smuggled message is the source, an
  accepted spoofed delivery is the sink.
license: MIT
---

# Testing SMTP smuggling and email spoofing: authenticated, and still forged

Email authentication is a stack of three records that only stops spoofing if all three are
present, aligned, and enforced - and even then only if every server on the delivery path
agrees where one message ends and the next begins. The failures are quiet: a policy published
but set to take no action, a signature that validates a domain that is not the visible sender,
a subdomain with no policy, and the boundary desync where two servers parse the end-of-data
sequence differently, letting a forged message ride inside a legitimate one and pass every
check. You find them by testing what actually gets delivered and trusted, not by reading the
records and assuming they hold.

## When to use

- You are auditing a domain's email authentication or a mail server's message handling.
- The domain publishes sender-authentication records, or receives mail through more than one hop.
- You can send test mail into the path and observe what is accepted and how it is judged.

## Scope check

Test mail authentication and delivery only for domains and servers you own or are authorized to
assess, sending only to mailboxes you control. Spoofed mail to third parties is out of scope and
often unlawful. If you can't name the authorization, stop.

## The loop

1. **Map the mail path and the records.** Identify the servers a message crosses inbound and
   outbound, and retrieve the domain's sender-authentication records: the authorized-senders
   policy, the signing setup, and the alignment-and-reporting policy. Note the published
   enforcement action and whether it is set to reject, quarantine, or merely observe. The path
   and the records together define what should stop a forgery.

2. **Check enforcement, not just presence.** A record that exists but sets its action to none, or
   a reporting policy in monitor mode, authenticates nothing in practice. Confirm the policy
   instructs receivers to actually reject or quarantine failures, and that receivers on the path
   honor it. Present-but-permissive is the most common gap.

3. **Check alignment between the envelope and the visible sender.** Authentication that passes for
   a domain the recipient never sees does not stop a spoof of the domain they do. Verify that the
   authorized-senders check and the signature align with the header sender the mailbox displays,
   not only with the envelope or a signing domain the user never reads. A pass without alignment is
   a bypass.

4. **Check subdomains and absent policy.** A strict policy on the root domain does not cover a
   subdomain that publishes none, and many receivers treat an absent policy as no enforcement. Test
   spoofing from a subdomain and from cousin names to find where the policy stops covering.

5. **Test the end-of-data boundary desync.** Craft a message whose end-of-data sequence is
   interpreted as the message end by one hop but not the other, so content after it is parsed by the
   next hop as a new, separate message. If the smuggled message asserts a sender from a domain the
   receiving hop trusts and it passes authentication, the boundary desync defeats the whole stack.
   Test the accepted variations of the terminating sequence between the inbound and outbound servers.

6. **Confirm and record.** Confirm by delivering, to a mailbox you control, a message that displays a
   forged sender and passes the domain's authentication - through a non-enforced policy, an alignment
   gap, an unprotected subdomain, an open relay, or a boundary desync. Kill the lead if policy is
   enforced and aligned across the root and subdomains, relay is closed, and both hops parse the
   message boundary identically. Record with the exact message, the path, and the auth verdict the
   recipient saw.

## Where email authentication leaks

- **Present is not enforced.** A policy set to take no action, or a receiver that ignores it,
  authenticates nothing. The enforcement action is the control.
- **A pass on the wrong domain is a bypass.** Authentication has to align with the sender the mailbox
  displays; a valid result for an envelope or signing domain the user never sees does not stop the
  spoof they do see.
- **Policy stops at the names it covers.** A subdomain or cousin name with no policy is an unauthenticated
  send from your organization's address space.
- **Two servers, one boundary, is an assumption.** If the inbound and outbound hops disagree on where a
  message ends, a forged message rides inside a real one and passes every check.
- **An open relay makes all of this moot.** A server that forwards mail for anyone is a spoofing engine
  regardless of the records.

## Worked example (a confirm and a kill)

> **Confirm.** The domain publishes all three records, but the alignment-and-reporting policy is set to
> take no action on failure. A message with a forged header sender from the domain, failing alignment,
> is delivered to a controlled mailbox and shown as authentic because no receiver enforces the failing
> policy. **Confirmed** spoofing through a non-enforced policy, `high`, remediation = move the policy to
> reject after confirming legitimate senders are aligned, and cover subdomains with an explicit policy.
>
> **Kill.** All three records are present, the policy enforces reject and is honored on the path, signature
> and authorized-senders both align with the visible header sender, root and subdomains carry enforcing
> policy, relay is closed to unauthenticated senders, and the inbound and outbound hops parse the
> end-of-data sequence identically so no boundary desync is possible. Every spoof and smuggle attempt is
> rejected or shown as failing. **Killed**, `kill_reason` = "enforced, aligned policy across root and
> subdomains, relay closed, consistent message-boundary parsing between hops."

## Rationalizations to reject

- *"We publish all three records."* → Published with what enforcement action, and honored by whom? A
  monitor-only or no-action policy stops nothing.
- *"Authentication passes."* → For which domain? A pass that does not align with the sender the mailbox
  shows is a bypass, not a defense.
- *"The main domain is locked down."* → And its subdomains and look-alike names? Policy only covers the
  names it is published for.
- *"Our mail server is standard."* → Standard servers have disagreed on the end-of-data sequence; that is
  exactly the smuggling class. Test the boundary between your hops.
- *"It is internal mail only."* → An open relay or a trusted-hop desync spoofs internal senders too, which
  is often the higher-value target.

## Executing this in practice

You need the inbound and outbound mail path, the domain's authentication records and their enforcement
actions, the ability to send crafted test messages into the path, and controlled mailboxes to observe the
verdict the recipient sees. The test is behavioral: send the aligned and misaligned, root and subdomain,
relayed, and boundary-desync variants and read what is accepted and how it is authenticated. Reading the
records tells you the intent; delivering the messages tells you the truth.

## Related

- `auditing-saml-and-oidc-flows` - the other federation-and-identity surface, where signatures and audiences
  are confused much as sender alignment is here.
- `enumerating-snmp-exposure` - a sibling network-service audit in the same lane, for management exposure.
- `testing-request-smuggling` - the same desync idea at the web layer: two parsers disagreeing on where one
  message ends.
- `mapping-attack-surface` - inventory the mail servers and records before testing them.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the crafted or smuggled message, sink = the accepted
  spoofed delivery, evidence = the message and the authentication verdict the recipient saw.
