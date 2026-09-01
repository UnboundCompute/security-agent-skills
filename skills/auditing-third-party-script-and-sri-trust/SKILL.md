---
name: auditing-third-party-script-and-sri-trust
description: >-
  Audit a web application for trust placed in third-party scripts it loads into its own page: an external
  script tag with no Subresource Integrity hash so a compromised CDN or vendor serves altered code that runs
  with full page privileges, a tag manager or analytics loader that injects further scripts the site never
  reviews, a script served over a mutable URL or wildcard source that can be swapped, and a Content-Security-
  Policy that is missing or permissive enough to allow arbitrary external script. Covers browser front ends,
  marketing and analytics tags, payment and widget embeds, and any page that loads JavaScript it did not author
  from another origin. Use when a page runs third-party script in its own security context and the integrity of
  that code is the boundary. The compromised or swapped third-party script is the source, the full-privilege
  execution in the page (data theft, skimming, defacement) is the sink, and the missing integrity pin or
  permissive script policy is the bug.
license: MIT
---

# Auditing third-party script and SRI trust: a script tag is code you did not write running as you

Every external script a page loads runs in that page's security context: it can read the DOM, cookies
accessible to script, form fields, and tokens, and it can send them anywhere. So a third-party script is not a
dependency the way a backend library is; it is code you did not write, delivered by a party you do not control,
executing with your page's full privileges on your users. The risk is supply-chain: the vendor's CDN is
compromised, the vendor ships a malicious update, or the script URL is swapped, and altered code runs on every
visitor. A form-skimming attack that steals payment details is exactly this, a legitimate analytics or widget
script replaced with one that also reads the checkout form. The defenses are integrity pinning (a Subresource
Integrity hash so the browser refuses altered code) and a Content-Security-Policy that constrains which
origins may serve script at all, and a tag manager that injects further scripts undoes both by loading code
nobody reviewed. The audit inventories every third-party script, checks whether its integrity is pinned and its
origin constrained, and treats any unpinned externally served script as attacker-controllable. You audit this
by listing what the page loads and asking, for each, what happens if that code is swapped.

## When to use

- A web page loads JavaScript from another origin: a CDN, an analytics or marketing tag, a payment or chat
  widget, a tag manager.
- External script tags may lack Subresource Integrity hashes, or scripts may be served over mutable or wildcard
  URLs.
- A tag manager or loader may inject further scripts, and the Content-Security-Policy may be missing or
  permissive.

## Scope check

Audit third-party script trust only on sites and properties you own or are authorized to assess. Reviewing what
a page loads is passive, but testing swap or injection scenarios touches real third-party relationships, so
stay within your own properties and test deployments and never tamper with a vendor's delivery or another
site's scripts. If you can't name the authorization, stop.

## The loop

1. **Establish the intended set of third-party code first.** Inventory every external script the page loads,
   who serves it, and why, including scripts injected by tag managers and loaders. Name what each is allowed to
   do and what would happen if it were swapped. This is the false-positive killer: a page that loads a known,
   minimal set of scripts, each pinned with Subresource Integrity, from origins a strict CSP allows, with no
   uncontrolled tag-manager injection, has bounded its third-party trust. Name the intended set, then check the
   controls.

2. **Check integrity pinning (SRI).** For each externally served script, confirm it carries a Subresource
   Integrity hash and `crossorigin` so the browser refuses to run code that does not match. An external script
   with no SRI hash runs whatever the CDN or vendor serves, so a compromised or updated source executes altered
   code with no barrier.

3. **Assess the Content-Security-Policy for script.** Determine which origins the CSP allows to serve script and
   whether it permits inline script, `unsafe-eval`, or wildcard sources. A missing CSP, or one permissive enough
   to allow arbitrary external script, means nothing constrains what code the page will run. Confirm the policy
   is a real allowlist, not a formality.

4. **Trace tag managers and script injectors.** Follow any tag manager, consent manager, or loader that injects
   further scripts at runtime. These load code the site never directly reviewed and often cannot pin, so they
   are a hole straight through SRI and CSP. Confirm injected scripts are governed (reviewed, pinned where
   possible, and CSP-constrained) rather than trusted wholesale.

5. **Check URL mutability and origin control.** Determine whether each script URL is versioned and immutable or
   points at a mutable "latest" path, and whether the serving origin is one the vendor could repoint. A script
   at a mutable URL or a wildcard origin can be swapped without the page changing at all, defeating a review
   done once.

6. **Confirm and record.** Confirm by demonstrating (on your own test deployment) that an unpinned or
   permissively allowed script would execute swapped code with full page privileges, reading form fields or
   tokens and exfiltrating them, without tampering with a real vendor. Kill the lead if every third-party
   script is integrity-pinned, served from a strict-CSP-allowed immutable origin, and no tag manager injects
   ungoverned code. Record the compromised or swapped script, the full-privilege execution in the page, and the
   missing integrity pin or permissive script policy.

## Where third-party script trust leaks

- **No Subresource Integrity.** An external script with no SRI hash runs whatever its CDN or vendor serves, so a
  compromise or update executes altered code unchallenged.
- **Missing or permissive CSP.** No script allowlist, or one that permits wildcard origins, inline script, or
  `unsafe-eval`, leaves nothing constraining what code the page runs.
- **Tag-manager injection.** A tag or consent manager that injects further scripts at runtime loads unreviewed,
  often unpinnable code straight past SRI and CSP.
- **Mutable script URLs.** A script at a "latest" or unversioned URL can be swapped without any change to the
  page, defeating a one-time review.
- **Wildcard or repointable origins.** A CSP or tag that allows a broad origin lets a vendor (or an attacker who
  compromises it) serve different code from an allowed source.

## Worked example (a confirm and a kill)

> **Confirm.** A checkout page loads an analytics script from a vendor CDN with a plain `<script src>` tag, no
> SRI hash, and a CSP that allows the vendor's wildcard origin. On a test deployment, replacing the served
> script with one that also reads the card-number field and posts it to an external URL runs unchallenged,
> because nothing pins the code and the origin is broadly allowed, the shape of a form-skimming compromise.
> **Confirmed** third-party script skimming exposure via missing SRI and permissive CSP, `high`, remediation =
> add a Subresource Integrity hash and `crossorigin` to the script, pin it to an immutable versioned URL, and
> tighten the CSP to the exact origins and paths needed with no wildcard or inline script.
>
> **Kill.** Every third-party script carries an SRI hash and `crossorigin`, is loaded from an immutable
> versioned URL on an origin a strict CSP allows by exact host, the CSP forbids inline script and `unsafe-eval`,
> and no tag manager injects ungoverned code. A swapped script fails the integrity check and never runs; an
> unlisted origin is blocked by the CSP. **Killed**, `kill_reason` = "all third-party script is integrity-pinned
> from strict-CSP-allowed immutable origins with no ungoverned injection; swapped or altered code does not
> execute."

## Rationalizations to reject

- *"It is a reputable vendor."* → Reputable CDNs and vendors are exactly the supply-chain targets; pin the code
  with SRI so a compromise of the reputable party still cannot run altered script.
- *"We reviewed the script."* → A review is a point in time; an unpinned or mutable-URL script can be swapped
  after review with no change you would notice, so pin and version it.
- *"We have a CSP."* → Confirm it is a real allowlist without wildcards, inline script, or `unsafe-eval`; a
  permissive CSP allows the attacker's script as readily as the vendor's.
- *"The tag manager handles that."* → A tag manager injects code you did not review and often cannot pin; it is
  a hole through your controls, not a control.
- *"It is just analytics."* → Analytics runs with full page privileges and can read the same forms and tokens as
  any script; there is no low-privilege third-party script.

## Executing this in practice

You need the full inventory of external scripts (including tag-manager-injected ones), each script's SRI hash
and `crossorigin` state, the CSP's script-source directives, and whether each URL is immutable and versioned.
List what the page loads, check each for an integrity pin and a strict allowed origin, and follow every
injector. Reading the page source and CSP shows the intended trust; a swapped script that runs on a test
deployment shows what the controls fail to stop.

## Related

- `auditing-clickjacking-and-ui-redressing` - the other front-end trust boundary enforced by response headers;
  CSP and framing controls are audited from the same page configuration.
- `hunting-supply-chain-risks` - the build-time supply chain; third-party script is the runtime, in-browser
  supply chain of the same kind of trust.
- `mapping-attack-surface` - use it to enumerate every origin and script a page loads before checking each
  one's integrity and policy.
- `hunting-wallet-drainer-and-dapp-approval-abuse` - a compromised dApp script is how a drainer swaps a spender
  or prompt; front-end integrity and approval abuse meet in the page.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the compromised or swapped third-party script, sink =
  the full-privilege execution in the page, evidence = the missing integrity pin or permissive script policy.
