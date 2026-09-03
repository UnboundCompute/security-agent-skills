---
name: auditing-open-redirect-and-forced-navigation
description: >-
  Audit redirect and navigation flows where an untrusted return, next, or callback URL, a stored value, or
  a referer drives a server redirect, a client-side location assignment, or a meta refresh to an
  attacker-chosen destination, enabling phishing, credential capture on a lookalike page, or theft of an
  OAuth code or token carried on the redirect. Use when a parameter or stored value controls where a user
  is sent after login, logout, an action, or an authorization step. Covers protocol-relative and backslash
  hosts, userinfo and whitespace tricks, and dangerous schemes that a naive allowlist misses. The untrusted
  target URL is the source, the redirect or navigation is the sink, and sending the user to an
  attacker-controlled destination is the bug.
license: MIT
---

# Auditing open redirect and forced navigation: when the app sends the user wherever the URL says

An application constantly sends users onward: back to where they were after login, to a success page after
an action, to an identity provider and back during authorization. When the destination comes from an
untrusted parameter, a stored value, or a header, and the app redirects there without confirming it is one
of its own pages, the app becomes a trusted springboard to an attacker's site. The link starts on the real
domain, so it survives a glance and a mail filter, and lands on a lookalike login that harvests
credentials. Worse, when the redirect carries an authorization code or a token in the URL, forcing the
destination steals that secret outright. The bug is not the redirect feature; it is a destination taken
from the request and trusted without an allowlist. You find it by locating every redirect and navigation
and asking whether an attacker chooses where it goes.

## When to use

- A return, next, redirect, callback, or continue parameter controls where a user is sent after an action.
- Client code assigns a location or writes a meta refresh from a value influenced by the request or storage.
- An authorization or single-sign-on flow returns to a URL supplied in the request and carries a code.

## Scope check

Test redirect behavior only against applications you own or are authorized to assess, sending the redirect
to a benign destination you control rather than a live phishing page, and never capturing a real user's
token. A confirmed forced redirect that carries a token is credential theft, so keep every probe within the
authorized scope. If you can't name the authorization, stop.

## The loop

1. **Establish whether the target is checked against an allowlist first.** For each redirect, determine
   whether the destination is confirmed to be one of the app's own pages, by an allowlist of paths or hosts
   matched exactly, or whether it is used as given. This is the false-positive killer: a redirect that only
   ever sends the user to a fixed internal path, or validates the target against an exact allowlist, is not
   open, while one that reflects the parameter is. Name the check before crafting a target.

2. **Map every redirect and navigation fed untrusted input.** Trace return and next parameters, stored
   redirect values, and referer or other headers into server redirects, client-side location assignments,
   and meta refreshes, including the return URL of any authorization flow. Each is a candidate sink.

3. **Test the bypasses a naive check misses.** A check that requires the target to start with a slash is
   beaten by a protocol-relative slash-slash-host and by a backslash the browser treats as a slash; a host
   allowlist is beaten by a userinfo at-sign, an embedded credential, a lookalike or encoded host, and
   trailing whitespace or control characters. Decide which of these the check as written lets through.

4. **Judge the destination scheme.** A redirect that permits a script or data scheme in a client-side
   navigation is not only off-site but executes in the current origin; confirm whether the sink restricts to
   HTTP and HTTPS or passes any scheme, since the impact differs from a plain off-site send.

5. **Follow what the redirect carries.** A plain open redirect is a phishing primitive; a redirect in an
   authorization flow carries a code or token in the URL or fragment, so forcing its destination steals that
   secret. Determine whether the sink is a bare navigation or one that transports a credential, because that
   sets the severity.

6. **Confirm and record.** Confirm by supplying a target that resolves to a benign host you control and
   observing the app send the user there, and for authorization flows, the code or token arriving at your
   host, on an isolated instance. Kill the lead if the target is matched against an exact allowlist of the
   app's own pages, if only a fixed internal path is used, or if no untrusted value reaches the redirect.
   Record the parameter, the sink, the bypass used, and whether a credential was carried, or set a
   `kill_reason`.

## Where forced navigation leaks

- **The allowlist is the whole defense.** A destination confirmed to be one of the app's own pages cannot be
  redirected off-site; a reflected parameter can. The presence and exactness of the allowlist is the finding.
- **Starts-with-slash is not enough.** A protocol-relative slash-slash-host and a backslash-host both pass a
  naive leading-slash check and send the user to another origin.
- **Host checks fall to userinfo and lookalikes.** An at-sign puts the real host in the userinfo and the
  attacker host after it, and an encoded or homoglyph host defeats a substring match; only an exact host
  comparison holds.
- **Schemes matter on the client.** A client-side navigation that accepts a script or data scheme executes in
  the origin, turning an open redirect into script execution rather than a mere off-site send.
- **Authorization redirects carry secrets.** When the redirect transports a code or token, controlling the
  destination hands that credential to the attacker, which is far worse than phishing.

## Worked example (a confirm and a kill)

> **Confirm.** A login flow reflects a return parameter into the post-login redirect after only checking that
> it begins with a slash. A value of backslash-backslash-host slips past the check, and the app sends the
> authenticated user, carrying the session-establishing fragment, to a host the tester controls on an
> isolated instance. **Confirmed** open redirect carrying a credential to an attacker host, `high`,
> remediation = validate the return target against an exact allowlist of the app's own paths, reject
> protocol-relative, backslash, userinfo, and non-HTTP targets, and never place a code or token on a redirect
> whose destination is client-influenced.
>
> **Kill.** The same flow resolves the return target against an allowlist of the app's own relative paths,
> rejects any absolute URL, protocol-relative or backslash form, userinfo, and non-HTTP scheme, and falls
> back to a fixed internal path when the target is not on the list. A crafted off-site target is discarded and
> the user lands on the default page. **Killed**, `kill_reason` = "redirect target is matched against an exact
> allowlist of the app's own paths and every off-site or alternate-scheme form is rejected; the destination is
> never attacker-chosen and carries no credential off-site."

## Rationalizations to reject

- *"It only redirects within our site."* -> Confirm that with an exact allowlist; a leading-slash or substring
  check is bypassed by protocol-relative, backslash, and userinfo forms that leave the site.
- *"An open redirect is low severity."* -> On an authorization flow it steals the code or token, and anywhere
  it is a credible phishing springboard from your trusted domain; rate it by what it carries.
- *"We block http and https other hosts."* -> Also block protocol-relative and backslash hosts and non-HTTP
  schemes; the bypasses live in the forms a naive host check does not parse.
- *"The parameter is validated."* -> Validated how; a check that the value is a URL or starts with a slash is
  not a check that it is one of your pages. Only an exact allowlist answers that.
- *"Users can see the address bar."* -> The link begins on your domain and the redirect is instant; the user
  and the mail filter both trust the starting host, which is the whole point of the abuse.

## Executing this in practice

You need every redirect and client-side navigation fed an untrusted target, the check applied to each, and
whether the sink carries a code or token. For each, decide whether an allowlist confines the destination to
the app's own pages or a bypass reaches another origin, and whether the scheme is restricted. Reading the
validation settles most leads; supplying a target that resolves to a benign controlled host, and watching
for a carried credential, on an isolated instance settles the rest.

## Related

- `hunting-crlf-and-response-splitting` - the Location header is the shared sink; a redirect target can also
  carry a CRLF, so the two analyses meet on the same header.
- `hunting-host-header-and-url-parsing-trust` - the parser differentials and host-confusion tricks that beat a
  redirect allowlist are the same ones that skill treats for URL parsing trust.
- `hunting-unicode-normalization-and-canonicalization-bypass` - an encoded or homoglyph host that defeats the
  redirect allowlist is a canonicalization failure that skill analyzes directly.
- `hunting-reflected-and-stored-xss` - a client-side navigation that accepts a script or data scheme turns an
  open redirect into the script execution that skill hunts.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the untrusted target URL, sink = the redirect or
  navigation, evidence = the user sent to a benign controlled host, or a carried code or token arriving there,
  on an isolated instance.
