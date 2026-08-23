---
name: hunting-subdomain-takeover-and-dangling-dns
description: >-
  Hunt DNS records that point at infrastructure the organization no longer controls, so an attacker
  can claim the target and serve content under a trusted name. Covers a CNAME or ALIAS to a
  decommissioned platform host that the provider lets anyone re-register, a dangling NS delegation
  whose nameserver or zone no longer exists in the account, a record that outlives its resource
  because infrastructure-as-code or a teardown pipeline deletes the resource but not the record, and a
  wildcard or unclaimed virtual host routed to a default backend. Use when reviewing DNS zone files,
  the DNS records declared in infrastructure-as-code, and the pipelines that provision and tear down
  them. The resolvable record whose target is unclaimed is the source, an attacker serving content on
  the trusted origin is the sink, and a claimable dangling target is the bug.
license: MIT
---

# Hunting subdomain takeover and dangling DNS: who can claim the name you still point at

A subdomain takeover is not a source-code bug, which is exactly why source-focused reviews miss it.
It is a bug in DNS state: a record still resolves, but the thing it points at is gone, and the shared
provider will hand that target to whoever asks next. When an attacker claims it, they serve content on
your trusted origin, with a real certificate, inheriting parent-domain cookies, allowlist and
content-policy trust, and OAuth redirect standing. You hunt it by inventorying the records, resolving
each target, and asking one question per record: is the target still owned, and if not, can an
attacker claim it. The discipline is separating a record that merely returns a 404 from one whose
target is genuinely re-registrable, because only the second is a takeover.

## When to use

- You have DNS zone files, or the DNS records declared in infrastructure-as-code, for a domain in scope.
- A subdomain points at a platform host, storage bucket, content-delivery endpoint, or third-party service.
- You want to know which records are dangling and, of those, which an attacker can actually claim.

## Scope check

Enumerate and fingerprint only domains you own or are authorized to assess, and never register or
claim a target you do not own outside an authorized engagement, claiming it is the point of no return
and can hijack live traffic. Demonstrate claimability by fingerprint, not by seizing the name. If you
can't name the authorization, stop.

## The loop

1. **Inventory the records and resolve every target.** Collect the CNAME, ALIAS, NS, and wildcard
   records from the zone and from any infrastructure-as-code that declares them, and resolve each to the
   host or service it points at. The record set is the surface; a record whose target no longer answers,
   or answers as unrecognized, is the lead.

2. **Fingerprint the target's claim state.** For each dangling-looking target, read the response the
   provider returns. A takeover needs two things together: a provider fingerprint that says the host is
   unclaimed (an unrecognized-domain page, a no-such-bucket error, a default landing that invites
   registration) and a provider that permits open re-registration of that name. One without the other is
   not a takeover.

3. **Check for dangling delegation.** An NS record that delegates a subdomain to a nameserver or hosted
   zone that no longer exists in the account hands the entire subtree to whoever can create that zone.
   Delegation takeovers are more severe than a single CNAME because they capture every name beneath the
   delegated label.

4. **Trace the provisioning and teardown pipelines.** Look for the record that outlives its resource: a
   name declared in one infrastructure-as-code state while the resource it targets was removed from
   another, and a teardown pipeline that deletes the cloud resource but leaves the DNS record to a
   separate, manual, or later step. Ordering that frees the resource before the record creates a dangling
   name on every teardown, by design.

5. **Check wildcards and unclaimed virtual hosts.** A wildcard record that resolves every label to a
   shared ingress routing by host header lets an attacker present an unclaimed host that a default or
   attacker-reachable backend serves. Enumerate which labels are actually bound and which fall through.

6. **Confirm and record.** Confirm by matching the provider's unclaimed fingerprint and verifying the
   provider allows re-registration of that exact name, without seizing it. Kill the lead if the target is
   still owned by a live resource in another account or region despite the 404, if the provider verifies
   ownership or scopes the name to the original account so it cannot be re-registered, if the record is
   NXDOMAIN with no delegation and no wildcard behind it (a dead record, not a dangling one), if the name
   resolves only inside a private or split-horizon zone unreachable from the attacker's position, or if
   the resource is intentionally managed out of band and still owned. Record the record, the target, the
   provider fingerprint, and the claim path.

## Where dangling DNS leaks

- **A dangling record is an infrastructure-state bug, not a code bug.** Source review never sees it; you
  have to read the zone and resolve the targets to find it.
- **A 404 is not a takeover.** The record has to point at a target the provider will hand to someone
  else; a target still owned by the organization returns 404 and is inert.
- **Delegation takeovers own the whole subtree.** A dangling NS record is worse than a dangling CNAME,
  because claiming the delegated zone captures every name under it.
- **Teardown ordering manufactures dangling names.** When the pipeline deletes the resource before the
  record, or a different owner holds the record, every teardown leaves an orphan; the fix is ordering,
  not a one-time cleanup.
- **The claimed origin inherits the parent's trust.** A taken-over subdomain gets a valid certificate,
  parent-domain cookies, allowlist and content-policy standing, and redirect trust; that inheritance is
  what makes it critical.

## Worked example (a confirm and a kill)

> **Confirm.** A subdomain CNAMEs to a platform-as-a-service host whose application slug was
> decommissioned; the host now returns the provider's unrecognized-application page, and the provider
> lets any account register that slug. Registering it (in an authorized test) would serve attacker
> content on the trusted subdomain with a valid certificate. **Confirmed** subdomain takeover via a
> dangling CNAME to a re-registrable platform host, `high` rising to `critical` where the parent domain
> shares cookies or an allowlist, remediation = remove the dangling record, and reorder teardown so the
> record is deleted before or with the resource.
>
> **Kill.** A subdomain CNAMEs to a storage host that returns a no-such-bucket 404, but the bucket name
> is still reserved by the organization in another region and the provider scopes bucket names globally
> so no other account can register it. The record resolves, the target 404s, but the name cannot be
> claimed. **Killed**, `kill_reason` = "target returns 404 but the name is account-reserved and the
> provider forbids re-registration by any other account, so there is no claim path."

## Rationalizations to reject

- *"The subdomain returns a 404, so it is dead and harmless."* -> A 404 from an unclaimed, re-registrable
  target is the takeover. Fingerprint the provider and check whether the name can be claimed.
- *"We deleted the resource, so the record is fine."* -> The record still resolves to the freed target.
  Deleting the resource without the record is what creates the danger.
- *"It is only a staging subdomain."* -> Staging usually shares the parent domain's cookies and trust; a
  takeover there reaches production users through that shared origin.
- *"The wildcard covers it."* -> A wildcard to a host-routing ingress lets an attacker present an
  unclaimed label the default backend serves; the wildcard is the exposure, not the cover.
- *"Nobody knows that record exists."* -> Passive DNS and certificate logs publish it. Obscurity is not
  ownership; claimability is.

## Executing this in practice

You need the zone records and the DNS declared in infrastructure-as-code, the resolved target of each
record, the provider fingerprint for any target that no longer answers as owned, the provider's rules
on re-registering that name, and the provisioning and teardown pipelines that create and remove the
records. For each dangling target, decide whether an attacker can claim it. Resolving the record tells
you it is dangling; the provider fingerprint and its registration rules tell you whether it is
takeoverable.

## Related

- `mapping-attack-surface` - the recon that inventories the DNS surface this hunt adjudicates; surface
  discovery finds the records, this skill decides which are claimable.
- `auditing-session-lifecycle-and-fixation` - its broad-cookie-domain shape hands off here: a session
  cookie scoped to a parent domain is capturable once a sibling subdomain is taken over.
- `auditing-cors-and-cross-origin-trust` - a taken-over subdomain becomes a trusted origin, so the two
  weaknesses compound where an allowlist trusts sibling subdomains.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the resolvable record with an unclaimed target,
  sink = an attacker serving content on the trusted origin, evidence = the provider fingerprint and the
  claim path, without seizing the name.
