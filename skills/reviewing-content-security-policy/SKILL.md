---
name: reviewing-content-security-policy
description: >-
  Review a content security policy as a script-injection defense and judge whether it would actually
  stop injected script, with the discipline that a weak policy is a real finding mainly where an
  injection sink it would otherwise block exists. Covers a script source that allows inline script with
  no neutralizing nonce or hash, that allows arbitrary hosts or data URLs, or that trusts a host serving
  attacker-usable script; a nonce that is static, reused, low-entropy, or reflected from input; a
  missing base-uri or object-src that defeats an otherwise strong nonce policy; and a report-only header
  shipped as the only policy. Use when reviewing a policy in a response header, a meta tag, or config,
  alongside the pages it protects. The policy is the control under test, injected script is the sink it
  must block, and a gap the injection reaches is the bug.
license: MIT
---

# Reviewing a content security policy: whether it stops the injection or only looks strict

A content security policy is a defense-in-depth control against injected script, and reviewing one is
less about listing weak directives than about a single judgment: if an attacker gets markup onto this
page, does the policy stop the script from running. That reframing matters both ways. A permissive
policy on a page with a real injection vector turns a mitigated cross-site-scripting bug into an
exploitable one, and is a genuine finding. The same permissive policy on a page with no injection sink
is defense-in-depth worth hardening but not an exploit. And a policy that looks weak, an inline
allowance sitting next to a nonce, is often not weak at all, because the browser ignores the legacy
allowance when the nonce is present. You review it by reading the policy against the pages it protects
and asking what an injected script could still do.

## When to use

- A response sets a content security policy, in a header, a meta tag, or configuration.
- You are already looking at a page that reflects or stores user input, or want to know whether one is protected.
- You need to tell a policy that blocks injected script from one that only appears to.

## Scope check

Review policies on applications you own or are authorized to assess, and demonstrate a bypass only
against pages in scope. A confirmed policy bypass paired with an injection is a working cross-site
script, so treat it as such and coordinate. If you can't name the authorization, stop.

## The loop

1. **Map the policy, its delivery, and the sinks it must protect.** Read the policy and how it is
   delivered, and find whether the pages it covers have an injection sink: a place user input reaches
   markup or script. The presence or absence of that sink decides whether a weakness is exploitable or
   only defense-in-depth, so establish it first.

2. **Check inline script.** A script source that allows inline script with no nonce or hash lets an
   injected inline script or event-handler attribute run. This is the single most common real weakness,
   but read it carefully: a nonce or hash present alongside the inline allowance makes a modern browser
   ignore the allowance, so that combination is mitigated, not broken.

3. **Check for over-broad and bypassable sources.** A script source of a wildcard, any-host scheme, or a
   data URL lets an attacker load or inline script freely. A specific allowlist is only as strong as its
   weakest host: one that serves attacker-shaped script through a callback endpoint, hosts arbitrary user
   content, or is a known script gadget is an allowlist bypass even though it looks constrained.

4. **Check nonce quality and the directives that guard a nonce policy.** A nonce is only protection if it
   is unpredictable and single-use: a static, reused, short, or input-reflected nonce is bypassable. And a
   nonce policy is undone by omission elsewhere, a missing base-uri lets an injected base element re-point
   relative script URLs around the nonce, and a missing object-src leaves a plugin-based script vector
   open. A strong nonce policy makes the host allowlist largely irrelevant, which is the intended shape.

5. **Check enforcement.** A policy delivered only in report-only form blocks nothing; it reports. Shipped
   as the sole policy it is decorative. Report-only alongside a genuinely enforced policy is a normal way
   to stage a change and is not the bug.

6. **Confirm and record.** Confirm by pairing an injection sink on a covered page with the policy gap: show
   that an injected inline script runs under an unneutralized inline allowance, that a permitted host
   serves usable script, or that an injected base element defeats the nonce. Kill the lead if the inline
   allowance is neutralized by a co-present nonce, the evaluation allowance is unused by the app, the
   permissive policy covers a page with no injection sink, the nonce policy carries base-uri and object-src,
   or a report-only header accompanies an enforced one. Record the directive, the sink, and the bypass.

## Where a policy leaks

- **A policy is only a finding where an injection can reach the gap.** Judge the directive and the page
  together; a weak policy over a page with no sink is hardening, not an exploit.
- **An inline allowance next to a nonce is usually mitigated.** The browser ignores the legacy allowance
  when a nonce is present, so the obvious-looking weakness is often not one.
- **The allowlist is only as strong as its weakest host.** A callback endpoint, a user-content host, or a
  known gadget on the list is a bypass regardless of how tight the rest looks.
- **A nonce policy is defeated by what it forgot.** A missing base-uri re-points scripts around the nonce;
  a missing object-src leaves a plugin vector; the nonce alone is not the whole policy.
- **Report-only is not enforcement.** As the only policy it blocks nothing; it is a staging mode mistaken
  for a control.

## Worked example (a confirm and a kill)

> **Confirm.** A page reflects a query parameter into the document without escaping, and the response
> policy allows inline script with no nonce or hash present. The injected inline script runs because the
> policy does not neutralize it, turning a reflected injection into a working cross-site script. **Confirmed**
> policy fails to block an injection it covers, `high`, remediation = remove the inline allowance, adopt a
> per-response random nonce with the strict-dynamic behavior, and add base-uri and object-src restrictions;
> fix the underlying injection as well.
>
> **Kill.** The same page's policy allows inline script but also sets a fresh high-entropy per-response
> nonce, so the browser ignores the inline allowance; it sets base-uri and object-src to none, its host
> allowlist carries no callback or user-content origin, and it is enforced rather than report-only. An
> injected inline script without the nonce does not run. **Killed**, `kill_reason` = "inline allowance
> neutralized by a co-present per-response nonce, base-uri and object-src locked down, allowlist free of
> usable-script hosts, and the policy enforced; injected script is blocked."

## Rationalizations to reject

- *"The policy allows inline script, so it is broken."* -> Not if a nonce or hash is present; the browser
  then ignores the inline allowance. Check for the nonce before flagging.
- *"There is no injection here, but the policy is weak."* -> Then it is hardening, not an exploit. Record it
  as defense-in-depth and rate it accordingly, not as a live cross-site script.
- *"The allowlist is specific, so it is safe."* -> One host on it that serves attacker-usable script is a
  bypass. Specific is not the same as safe.
- *"We use a nonce, so we are covered."* -> A nonce policy without base-uri and object-src is bypassable
  around the nonce. The nonce is necessary, not sufficient.
- *"The policy is deployed."* -> In report-only mode it blocks nothing. Confirm it is enforced, not just
  present.

## Executing this in practice

You need the policy text and how it is delivered, whether a nonce or hash is present and how it is
generated, the host allowlist and what each host can serve, the base-uri and object-src directives, and
whether the covered pages have an injection sink. For each weakness, decide whether an injection could
reach it and whether another part of the policy neutralizes it. Reading the policy against the page tells
you the gap; pairing it with a real injection sink tells you whether the gap is reachable.

## Related

- `testing-client-side-dom-vulnerabilities` - finds the injection sink this review needs; a policy gap
  matters exactly where such a sink exists.
- `auditing-cors-and-cross-origin-trust` - the other browser-enforced control on the same pages; the two
  weaknesses often compound on one response.
- `adjudicating-taint-paths` - the same discipline of confirming a sink is reachable before calling a
  weakness a bug, applied to the injection this policy is meant to stop.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the injected markup, sink = script execution the
  policy must block, evidence = the injected script that runs despite the policy.
