---
name: auditing-clickjacking-and-ui-redressing
description: >-
  Audit a web application for UI-redressing attacks where an attacker frames the real site and tricks a user
  into acting on it unknowingly: a sensitive page that can be embedded in an attacker's iframe because it lacks
  frame-ancestors or X-Frame-Options, a state-changing action reachable by a single click that an overlay or
  transparent frame steers, a drag-and-drop or keystroke redressing that captures input meant for the attacker
  page, and a confirmation step that a framed overlay hides. Covers web pages with authenticated, state-changing
  actions (settings changes, purchases, approvals, connect flows) that could be loaded inside a frame the user
  cannot see. Use when a sensitive action can be triggered by a click and the page can be framed by another
  origin, making framing the boundary. The attacker page that frames or overlays the real site is the source,
  the unknowing state-changing click is the sink, and the missing framing protection or unguarded one-click
  action is the bug.
license: MIT
---

# Auditing clickjacking and UI redressing: the user clicks your button on the attacker's page

Clickjacking works because the browser will happily render your real, authenticated page inside a frame on
someone else's site, and the user cannot tell. The attacker loads your page in a transparent or hidden iframe,
positions it under decoy content, and lures the user into clicking what looks like the attacker's button, when
the click actually lands on your page, taken with the user's own session. So any sensitive, state-changing
action that a single click can trigger, changing a setting, confirming a purchase, granting an approval,
connecting an account, is exposed if your page can be framed by another origin. The defense is to refuse
framing by untrusted origins, with a Content-Security-Policy `frame-ancestors` directive (and the older
X-Frame-Options as a fallback), and, for the highest-value actions, to not make them one-click blind: a
confirmation the overlay cannot fake. Related redressing tricks, drag-and-drop and keystroke capture, exploit
the same framing to steal input. The audit checks, for every sensitive page, whether it can be framed and
whether its state-changing actions are reachable by a single hidden click. You audit this by trying to frame
the real page from another origin and drive its actions.

## When to use

- A web page performs authenticated, state-changing actions triggerable by a click (settings, purchase,
  approval, connect, delete).
- The page may lack a `frame-ancestors` CSP directive or X-Frame-Options, so another origin can frame it.
- A sensitive action may be one-click with no confirmation an overlay cannot hide.

## Scope check

Audit clickjacking only on applications you own or are authorized to assess. Framing the real site to
demonstrate the attack uses a real authenticated session, so use test accounts and your own attacker-page test
harness and never trick a real user or act on an account that is not yours. If you can't name the
authorization, stop.

## The loop

1. **Establish which actions are sensitive and whether the page can be framed first.** Inventory the
   authenticated, state-changing actions on each page and name which are high-value (money, permissions, data
   deletion, account connection). Then determine the intended framing policy: which origins, if any, may embed
   the page. This is the false-positive killer: a page that refuses framing by untrusted origins via
   `frame-ancestors` and whose highest-value actions require a confirmation an overlay cannot fake is not
   clickjackable. Name the sensitive actions and the framing policy, then test framing.

2. **Test whether the page can be framed.** From an attacker-origin test page, load the target in an iframe and
   confirm whether the browser renders it. Check for a `frame-ancestors` CSP directive and, as a fallback,
   X-Frame-Options; confirm they name only trusted origins. A page that renders in an arbitrary origin's frame
   is embeddable for a redress.

3. **Map one-click state-changing actions.** For each sensitive action, determine whether a single click
   triggers it with no further step. A framed one-click action, positioned under a decoy, is taken the moment
   the user clicks the decoy. Actions that change money, permissions, or data with one click are the priority
   targets.

4. **Test overlay and confirmation hiding.** Simulate the attack: overlay the framed page with decoy content
   and confirm whether the sensitive control can be positioned under an innocuous-looking click. Check whether
   any confirmation step is one the overlay can hide or auto-satisfy, or a genuine barrier the attacker cannot
   fake. A confirmation the frame can cover is not protection.

5. **Check drag-and-drop and keystroke redressing.** Beyond clicks, test whether framing enables dragging a
   value into a field, or capturing keystrokes the user believes go to the attacker page, to fill a sensitive
   form on the framed site. These redressing variants exploit the same missing framing protection.

6. **Confirm and record.** Confirm from an attacker-origin test page by framing the real page and driving a
   sensitive action with a click aimed at a decoy, using a test account and your own harness, without tricking a
   real user. Kill the lead if the page refuses framing by untrusted origins via `frame-ancestors` (with
   X-Frame-Options fallback) and its high-value actions require a confirmation the overlay cannot fake. Record
   the attacker page that frames the site, the unknowing state-changing click, and the missing framing
   protection or unguarded one-click action.

## Where UI-redressing trust leaks

- **No frame-ancestors or X-Frame-Options.** A page with no framing restriction renders inside any origin's
  iframe, ready to be overlaid and redressed.
- **Permissive framing policy.** A `frame-ancestors` or X-Frame-Options that allows broad or wrong origins lets
  an attacker-controlled or compromised origin frame the page.
- **One-click sensitive actions.** A state-changing action a single click triggers is taken the instant the
  user clicks a decoy positioned over it.
- **Fakeable confirmations.** A confirmation step the framed overlay can hide, cover, or auto-satisfy is not a
  barrier to a redress.
- **Input redressing.** Framing that allows drag-and-drop or keystroke capture fills sensitive fields on the
  real page with input the user meant for the decoy.

## Worked example (a confirm and a kill)

> **Confirm.** An account settings page lets a user connect an external account or change a security setting
> with a single button click, and the page sends no `frame-ancestors` CSP directive and no X-Frame-Options. An
> attacker-origin test page loads it in a transparent iframe under a decoy game, positions the connect button
> under the decoy's "play" button, and a test user's click connects the account without their awareness.
> **Confirmed** clickjacking of a sensitive one-click action via missing framing protection, `high`,
> remediation = set `frame-ancestors 'self'` (and X-Frame-Options DENY or SAMEORIGIN as fallback) on sensitive
> pages, and require a confirmation an overlay cannot fake for high-value actions.
>
> **Kill.** Sensitive pages send `frame-ancestors 'self'` (with X-Frame-Options as a fallback) so no untrusted
> origin can frame them, and the highest-value actions require a confirmation the overlay cannot hide or
> auto-satisfy. An attacker-origin frame is refused by the browser, so no decoy overlay can steer a click.
> **Killed**, `kill_reason` = "page refuses framing by untrusted origins via frame-ancestors and high-value
> actions require a non-fakeable confirmation; no framed click reaches a sensitive action."

## Rationalizations to reject

- *"Who would frame our site."* → Framing is a single iframe tag on any attacker page; the barrier is
  `frame-ancestors`, not obscurity, and without it the page is embeddable by anyone.
- *"We set X-Frame-Options."* → Confirm it is actually sent on the sensitive pages and not overridden, and add
  `frame-ancestors` since X-Frame-Options is the legacy fallback; a header set on some pages but not the
  sensitive one still exposes it.
- *"The action needs a confirmation."* → Confirm the overlay cannot hide or auto-satisfy that confirmation; a
  dialog the frame can cover or a checkbox the attacker pre-clicks is not a barrier.
- *"It is behind login."* → Clickjacking uses the victim's own logged-in session; being authenticated is the
  precondition for the attack, not a defense.
- *"CSP is for script."* → CSP also governs framing via `frame-ancestors`; the same header that constrains
  script is where you refuse untrusted framers.

## Executing this in practice

You need the sensitive, state-changing actions on each page, the framing headers each page sends
(`frame-ancestors` and X-Frame-Options), whether each sensitive action is one-click, and whether its
confirmation is fakeable under an overlay. From an attacker-origin test harness, try to frame each sensitive
page and drive its actions under a decoy. Reading the response headers shows the intended framing policy; a
sensitive action driven from a framed decoy on your harness shows whether it holds.

## Related

- `auditing-third-party-script-and-sri-trust` - the other front-end trust boundary set by response headers; CSP
  carries both `frame-ancestors` here and the script allowlist there.
- `auditing-cors-and-cross-origin-trust` - the cross-origin trust boundary framing sits within; both decide
  what another origin may do with your authenticated context, one via requests, the other via a frame.
- `hunting-wallet-drainer-and-dapp-approval-abuse` - a redressed dApp UI hides a signing prompt; UI redressing
  and approval deception combine to get a user to authorize what they cannot see.
- `mapping-attack-surface` - use it to enumerate the authenticated, state-changing pages that need framing
  protection before testing each.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the attacker page that frames or overlays the real
  site, sink = the unknowing state-changing click, evidence = the missing framing protection or unguarded
  one-click action.
