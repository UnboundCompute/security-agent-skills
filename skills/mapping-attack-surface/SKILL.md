---
name: mapping-attack-surface
description: >-
  Map and prioritize the attack surface of an authorized black-box web target
  before testing it - enumerate hosts, endpoints, parameters, auth flows, and
  technologies, then order them by where bugs actually live. Use at the start of
  an in-scope engagement or bug-bounty target when you have a URL/app but no
  source, and need a systematic surface inventory instead of poking random
  endpoints; when you need to know what to test first. Enforces a scope gate and
  produces a prioritized surface inventory that feeds the vuln-class skills.
license: MIT
---

# Mapping attack surface (black-box)

You can't test what you haven't found, and you'll waste the engagement testing
low-value surface first. Recon is the discipline of turning "here's a URL" into a
prioritized inventory of everything that takes input, ordered by where bugs live.
This skill is the front of the black-box workflow; per-class hunting skills act on
its output.

## Scope gate - before anything else

Establish and write down scope *first*, and check every action against it:

- **Record the authorization**: which hosts/domains/apps are in scope, which are
  explicitly out, the rules (rate limits, no-DoS, no social engineering, test-
  account only), and the reporting channel. Keep it where you'll re-read it.
- **Check every request against scope before sending it.** A wildcard in a
  program's scope is not permission to hit a third party's system that happens to
  be reachable.
- **Passive before active; low-impact before high.** Prefer observation over
  probing until you've confirmed a target is in scope and the action is allowed.
- **Never run destructive or state-changing actions** (delete, mass-write,
  account takeover attempts) without explicit authorization for them.

If you can't point to the authorization for a host or an action, it's out of
scope. Stop and confirm.

## The recon loop

1. **Enumerate hosts.** From the in-scope roots: subdomains, related domains,
   and the apps behind them. Distinguish the origin from CDN/WAF front - testing
   a CDN edge as if it were the origin yields noise.

2. **Fingerprint the stack.** Server, framework, language, CMS, reverse proxy,
   auth provider, cloud. The stack tells you which bug classes are plausible and
   which default misconfigurations to check.

3. **Enumerate endpoints and parameters.** Walk the app as an authenticated user
   (with an in-scope test account), capture the real traffic, and pull the API
   surface from it. Add documented surface (OpenAPI/GraphQL introspection, JS
   bundles that reveal routes and params). Most surface is *not* what you clicked
   - it's referenced in client code and specs.

4. **Map the auth and session model.** How you log in, what a token/cookie
   represents, what scopes/roles exist, where the boundary between users sits.
   This is where the highest-severity black-box bugs (IDOR/BOLA, BFLA, auth
   bypass) live - map it deliberately, not incidentally.

5. **Catalog state-changing and input-taking operations.** Every endpoint that
   writes, uploads, redirects, fetches a URL, renders a template, or takes an id.
   These are your candidate anchors for the vuln-class skills.

6. **Prioritize.** Order the surface by expected yield: authenticated
   state-changers and object references (IDOR/BFLA) and anything that reflects,
   fetches, or parses input first; static/low-privilege surface last. Rank is
   triage - the whole inventory stays on the list; the order just says what to
   test first.

## What "good recon output" looks like

A written inventory, not a memory: hosts (origin vs edge), stack, an endpoint
table (method, path, params, auth required, what it does), the auth/role map, and
a prioritized test order. Everything downstream reads from this.

## Worked example

> **Target: a SaaS app, one in-scope root.**
> 1. Subdomain enum → `app.`, `api.`, `admin.` (confirm each in scope).
> 2. Fingerprint → framework X behind a CDN; note the CDN vs origin.
> 3. Authenticated crawl + captured traffic → 140 endpoints; JS bundle reveals 20
>    more never navigated to, including `/api/v1/org/{id}/members`.
> 4. Auth map → two roles (member, admin); token is a JWT with an `org` claim.
> 5. State-changers → invite, role-change, export, avatar-upload, redirect-after-
>    login.
> 6. Prioritize → `{id}`/`{org}` object refs and the role-change endpoint first
>    (IDOR/BFLA), then the URL-fetch and upload endpoints, then reflected params.
> Output: a surface table handed to the IDOR, auth, SSRF, and upload skills.

## Rationalizations to reject

- *"It's reachable, so it's in scope."* → Reachability is not authorization. Check
  the written scope for every host and action.
- *"I'll just start testing the first endpoint I see."* → Untriaged testing burns
  the engagement on low-value surface. Inventory and prioritize first.
- *"What I clicked is the surface."* → Most surface hides in client code, specs,
  and un-linked routes. Pull it from captured traffic and bundles.
- *"I'll remember the endpoints."* → Write the inventory. Recon you didn't record
  is recon you'll redo.

## Executing this in practice

Use whatever capture and enumeration tooling you have: an intercepting proxy or
browser capture for authenticated traffic, subdomain/asset enumeration for hosts,
and spec/bundle extraction for hidden routes. The output is tool-agnostic - a
written, prioritized surface inventory. Confirmed issues found while testing it
are written up with `writing-vuln-reports`.

## Related

- `writing-vuln-reports` - for anything confirmed during testing.
- (roadmap) per-class black-box skills - IDOR/BOLA, auth-bypass/BFLA, SSRF, open
  redirect, upload, injection - each consuming this inventory.
