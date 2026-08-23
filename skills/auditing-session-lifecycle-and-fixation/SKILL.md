---
name: auditing-session-lifecycle-and-fixation
description: >-
  Audit how an application issues, rotates, and destroys session identifiers, so an attacker cannot
  fixate or outlive a session. Covers a session identifier not regenerated at login or privilege change,
  a logout that clears the client cookie but leaves the server session valid, a session that never
  expires or has no idle or absolute timeout, an identifier accepted from a URL or a header an attacker
  can seed, a session cookie missing the secure, http-only, or same-site attributes, and a cookie scoped
  to a parent domain shared with untrusted subdomains. Use when reviewing authentication, logout, and
  session-management code and the cookie attributes it sets; it assumes the identifier is unguessable and
  scopes to lifecycle, not entropy. An attacker who can set or keep a session identifier is the source,
  the victim authenticating into it is the sink, and a session that is not rotated or invalidated is the
  bug.
license: MIT
---

# Auditing session lifecycle and fixation: whether a session is rotated, scoped, and destroyed

A session identifier can be perfectly random and still be a bug, because the attack is not guessing it
but controlling its lifecycle. If the identifier issued to an anonymous visitor survives unchanged into
their authenticated session, an attacker who planted it now holds that session; if logout clears only
the client cookie, a captured identifier stays valid; if nothing expires, a stolen identifier lives
forever. This audit assumes the identifier is unguessable, that is a separate concern, and asks whether
it is rotated at the right moments, invalidated when it should be, timed out, and scoped so only the
right party can hold it. You audit it by reading the login, privilege-change, and logout paths and the
cookie attributes, against the framework's session defaults. The discipline is checking those defaults
first, because the rotation or the attribute is often set centrally, not at the call site.

## When to use

- You are reviewing authentication, logout, or session-management code, or the cookie attributes it sets.
- You want to know whether a session can be fixated, replayed after logout, or held indefinitely.
- The identifier is already assumed unguessable and you are auditing its lifecycle, not its entropy.

## Scope check

Exercise session handling only against applications and accounts you own or are authorized to assess; a
fixation or replay demonstration rides a real user's authenticated session. Use test accounts and
coordinate. If you can't name the authorization, stop.

## The loop

1. **Establish the framework's session defaults first.** Determine what the framework does on its own:
   whether session-fixation protection regenerates the identifier at login by default, whether the
   session cookie's secure, http-only, and same-site attributes come from a global cookie policy or a
   gateway, and whether the server session store enforces its own timeout. These defaults decide what a
   missing call at the handler means, so settle them before flagging anything.

2. **Check rotation at login and privilege change.** The identifier issued before authentication must be
   replaced with a fresh one at login, and again at any privilege elevation (a step-up, a role grant,
   assuming another identity). Reusing the pre-authentication identifier is classic fixation: an attacker
   who seeds it in the victim's client is logged in as the victim once they authenticate.

3. **Check invalidation and expiry.** Logout must invalidate the session on the server, not only clear
   the client cookie, or the identifier stays replayable. And the session must have an idle and an
   absolute timeout enforced server-side, so a captured identifier eventually dies; a session that never
   expires or refreshes without an absolute cap never does.

4. **Check where the identifier is accepted from.** The identifier should come only from a server-set
   cookie, never read from a URL, query parameter, or a header an attacker can seed, since any
   attacker-settable channel is a fixation vector. Confirm the accepted sources.

5. **Check the cookie attributes and domain scope.** The session cookie needs secure (so it is not sent
   over cleartext), http-only (so script cannot read it), and a same-site setting justified by its
   cross-site needs. And a cookie scoped to a parent domain shared with untrusted or claimable subdomains
   can be read or set by a sibling host; this is the seam where a subdomain takeover captures the session.

6. **Confirm and record.** Confirm by planting an identifier in a victim client, authenticating, and
   showing the same identifier is now authenticated (fixation), or by replaying an identifier after logout
   or past its intended lifetime and showing it still works. Kill the lead if the framework regenerates
   the identifier at login by default and it is not overridden, if the missing cookie attributes are
   supplied by a global policy or a gateway rather than the call site, if a never-expiring client cookie
   only bears a short-lived server session record, if logout invalidates through a revocation store out of
   view of the handler, if the broad cookie domain covers only first-party subdomains none of which is
   claimable, or if a cross-site same-site setting is deliberate and paired with other defenses. Record
   the path, the missing lifecycle control, and the demonstration.

## Where session lifecycle leaks

- **A random identifier with a bad lifecycle is still a bug.** The attack is fixation, replay, or an
  endless lifetime, none of which entropy prevents; this audit is orthogonal to how the identifier is
  generated.
- **The control is often a framework default, not a call at the handler.** Rotation on login and the
  cookie attributes are frequently set centrally; read the configuration before calling either missing.
- **Logout that clears the cookie is not invalidation.** If the server session survives, the identifier
  is replayable; the destroy has to happen on the server side.
- **A never-expiring session never dies.** Without an idle and an absolute server-side timeout, a captured
  identifier is valid indefinitely; a client-cookie lifetime is not the session lifetime.
- **A broad cookie domain shares the session with the neighbors.** A parent-domain cookie is readable by
  sibling subdomains; if one is untrusted or claimable, the session leaks through it.

## Worked example (a confirm and a kill)

> **Confirm.** The login handler authenticates the user and issues no new session identifier, and the
> framework's fixation protection is turned off in configuration; the identifier accepted before login is
> the same one carried afterward. An attacker who sets that identifier in a victim's browser, then waits
> for the victim to log in, holds an authenticated session. **Confirmed** session fixation through a
> missing rotation on login, `high`, remediation = regenerate the session identifier at login and at every
> privilege change, and invalidate the prior one server-side.
>
> **Kill.** The handler shows no explicit regeneration call, but the framework's session-fixation
> protection is enabled by default and the authentication filter regenerates the identifier before the
> handler returns; the session cookie's secure, http-only, and same-site attributes come from a global
> cookie policy, and logout destroys the server session through a central store. Planting an identifier
> and authenticating yields a fresh one. **Killed**, `kill_reason` = "the framework regenerates the
> identifier at login by default and it is not overridden, attributes are set by the global policy, and
> logout invalidates server-side."

## Rationalizations to reject

- *"The session identifier is long and random."* -> This audit assumes that. Fixation, post-logout replay,
  and an endless lifetime all work on a perfectly random identifier; entropy is not the control here.
- *"The handler has no regenerate call, so it is fixation."* -> Not if the framework rotates by default.
  Read the session configuration before flagging; the rotation may be central.
- *"Logout deletes the cookie."* -> On the client only? Then the server session lives and the identifier
  replays. Invalidate it on the server.
- *"The cookie never expires for convenience."* -> Then a stolen identifier never expires either. Enforce
  an idle and an absolute timeout on the server side.
- *"The cookie is scoped to the whole domain so all our apps share login."* -> Every subdomain, trusted or
  not, can then hold the session. If any sibling is untrusted or claimable, that scope leaks it.

## Executing this in practice

You need the framework's session defaults (rotation, cookie policy, store timeout), the login and
privilege-change paths and whether they regenerate the identifier, the logout path and whether it
invalidates server-side, the idle and absolute timeouts, the channels the identifier is accepted from,
and the session cookie's attributes and domain scope. For each control, decide whether the lifecycle is
handled or the default covers it. Reading the paths and the configuration tells you what is enforced;
planting or replaying an identifier tells you whether the session holds when it should not.

## Related

- `auditing-randomness-and-nonce-quality` - the adjacent audit this one assumes is sound: it covers
  whether the identifier is guessable, while this covers whether its lifecycle is mishandled.
- `hunting-subdomain-takeover-and-dangling-dns` - the destination of the broad-cookie-domain seam: a
  parent-domain session cookie becomes capturable once a sibling subdomain is taken over.
- `auditing-tls-and-certificate-validation` - its downgrade-to-cleartext shape is what makes a session
  cookie missing the secure attribute capturable on the wire.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = an attacker who can set or keep an identifier,
  sink = the victim authenticating into it, evidence = the fixated or replayed session still authenticated.
