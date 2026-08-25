---
name: auditing-websocket-connection-trust
description: >-
  Audit a WebSocket endpoint for trust established once at the handshake and never re-checked, so a
  cross-site page or a post-handshake message drives a privileged action, after the origin check and the
  credential source are resolved. Covers a missing or always-true origin check on the upgrade with
  ambient-cookie authentication, authentication at the handshake with no per-message authorization, a
  message treated as transport into a downstream injection sink, an unbounded frame or connection
  allowance inviting denial of service, tunneling past HTTP-layer controls that inspect only the request,
  and sensitive data broadcast to under-scoped subscribers. Use when reviewing the upgrade handler, the
  per-message dispatch, and the origin and credential checks, not the wss certificate validation the
  transport skill owns. A cross-origin or unauthenticated handshake is the source, a privileged action
  reached over the socket is the sink, and trust checked only at the upgrade is the bug.
license: MIT
---

# Auditing WebSocket connection trust: trust checked once at the upgrade, never per message

A WebSocket endpoint fails where it establishes trust once, at the upgrade, and never re-checks it. If the
handshake authenticates purely from an ambient cookie and does not validate the origin, any web page can
open an authenticated socket on the user's behalf (cross-site hijacking); if it authenticates at the
handshake but never authorizes individual messages, a connection can escalate mid-stream. You audit it by
resolving how the handshake authenticates (an ambient cookie, or an unpredictable token a cross-site page
cannot read) and whether the origin is enforced, then following each privileged message to see whether it
is re-authorized. A known false-positive trap lives here: an origin check delegated to a proxy is only a
defense if the proxy actually rejects, so confirm the backstop. The wss certificate check belongs to the
transport skill.

## When to use

- You are reviewing a WebSocket upgrade handler, its origin check, and its per-message dispatch.
- The endpoint authenticates at the handshake, from a cookie or a token, and then handles messages.
- You want to know whether a cross-site page or a later message can drive a privileged action.

## Scope check

Audit only endpoints you own or are authorized to assess, and open a socket or send a message only against
a service in scope, a privileged message drives real state. Adjudicate on the handshake and the dispatch.
If you can't name the authorization, stop.

## The loop

1. **Resolve the origin check and the credential source first.** Determine how the handshake
   authenticates: an ambient session cookie sent automatically by the browser, or an unpredictable
   per-connection token (a ticket, a bearer in a subprotocol or query) that a cross-site page cannot read.
   Then determine whether the origin is validated on the upgrade, and where. Ambient credentials plus a
   missing origin check is the cross-site-hijacking precondition, so settle both before flagging.

2. **Check the origin gate on the upgrade.** Look for a missing origin check, or an origin callback that
   returns true unconditionally, on a handshake that authenticates from an ambient cookie, letting an
   attacker's page open an authenticated socket and act as the victim.

3. **Check per-message authorization.** Look for authentication established at the handshake with no
   authorization on individual messages, so a connection authenticated as a low-privilege user can send a
   privileged message the endpoint acts on without re-checking.

4. **Check the message-as-transport sink.** Look for a message field flowing into a downstream injection
   sink (a query, a command, markup) where the socket is merely the transport; the second layer, not the
   socket, decides whether it is exploitable.

5. **Check denial of service and control tunneling.** Look for an unbounded frame or message size or an
   unlimited connection allowance inviting resource exhaustion, for messages that tunnel past HTTP-layer
   controls that inspect only the upgrade request and not the frames, and for sensitive data broadcast to
   subscribers scoped more broadly than the data warrants.

6. **Confirm and record.** Confirm by opening a socket from an attacker origin (for hijacking) or sending
   a privileged message on a low-privilege connection (for per-message escalation) and showing the action
   succeeds. Kill the lead if the origin is validated at the reverse proxy or ingress before the upgrade
   and the proxy actually rejects a non-allowlisted origin (verify the rejection, do not assume it), if
   the handshake requires a non-cookie unpredictable token a cross-site page cannot obtain, if a
   strict-or-lax same-site session cookie already blocks the cross-site handshake, if per-message
   authorization is enforced in a dispatcher rather than per handler, if the downstream injection sink is
   parameterized or escaped so the socket is only transport, or if the endpoint is unauthenticated public
   broadcast with nothing to steal. Record the endpoint, the trust checked only at the upgrade, and the
   action reached.

## Where connection trust leaks

- **Ambient auth plus no origin check is cross-site hijacking.** A cookie-authenticated handshake with no
  origin gate lets any page open the victim's socket; the origin, or a non-ambient token, is the defense.
- **A handshake is not a per-message gate.** Authenticating once at the upgrade does not authorize each
  message; a privileged message on a low-privilege connection needs its own check.
- **A proxy origin check is a defense only if it rejects.** Delegating the origin check to an ingress is
  fine only when the ingress actually denies a bad origin; confirm the backstop rather than assuming it.
- **The socket is often just transport.** A message reaching a query or command sink is exploitable or not
  based on that downstream layer; parameterization there kills the finding.
- **Frames slip past request-only controls.** A control that inspects the HTTP upgrade but not the frames is
  bypassed by the messages; and unbounded frames or connections are a denial-of-service allowance.

## Worked example (a confirm and a kill)

> **Confirm.** An upgrade handler sets its origin callback to always return true and authenticates the
> connection purely from the session cookie, with no per-connection token. An attacker's page opens a
> socket to the endpoint; the browser attaches the victim's cookie, the socket authenticates, and the page
> drives a privileged message. **Confirmed** cross-site WebSocket hijacking through a missing origin check
> on ambient-cookie authentication, `high`, remediation = validate the origin on the upgrade, require a
> non-ambient per-connection token, and authorize each privileged message.
>
> **Kill.** The same endpoint requires a per-session ticket minted for a same-origin request and passed in
> the handshake, which a cross-site page cannot read, and the ingress rejects any non-allowlisted origin
> before the upgrade reaches the app. The cross-site page can neither carry the token nor pass the origin
> gate. **Killed**, `kill_reason` = "the handshake credential is not ambient and the origin is enforced and
> rejected upstream; the cross-site page cannot open an authenticated socket."

## Rationalizations to reject

- *"We authenticate the connection."* -> With an ambient cookie and no origin check? Then any page opens the
  victim's socket; the credential must be non-ambient or the origin must be enforced.
- *"The origin is checked at the proxy."* -> Does the proxy actually reject a bad origin? Confirm the
  rejection; a delegated check that never denies is not a check.
- *"They authenticated at connect."* -> Authenticated as whom, to do what? A privileged message needs its
  own authorization; the handshake does not cover it.
- *"The message goes to the database."* -> Through a parameterized query? Then the socket is transport and
  the sink is safe; through string-built SQL it is injection at that layer.
- *"It is a public feed."* -> Then hijacking steals nothing; but confirm no privileged message and no
  per-subscriber data crosses the same socket.

## Executing this in practice

You need the upgrade handler and its origin check (and where it runs), the credential the handshake trusts
and whether a cross-site page can obtain it, the per-message dispatch and any per-message authorization,
the downstream sinks messages reach, the frame and connection limits, and the ingress origin policy and
whether it rejects. For each privileged action, decide whether trust is re-checked past the upgrade.
Reading the handshake tells you how trust is established; sending a message tells you whether it is
re-checked.

## Related

- `auditing-grpc-service-authorization` - the sibling wire-protocol audit; there trust is assumed in an
  interceptor that misses a method, here trust is established once at the handshake and not re-checked.
- `auditing-cors-and-cross-origin-trust` - owns the HTTP preflight and cross-origin-header semantics; a
  WebSocket deliberately sits outside that model (the server validates the origin itself), a genuinely
  different trust decision kept separate here.
- `auditing-session-lifecycle-and-fixation` - owns cookie issuance and fixation; this skill owns whether an
  established connection re-checks trust per message, and the wss certificate check defers to the transport
  skill.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = a cross-origin or unauthenticated handshake, sink
  = a privileged action reached over the socket, evidence = the missing origin or per-message check and the
  action driven.
