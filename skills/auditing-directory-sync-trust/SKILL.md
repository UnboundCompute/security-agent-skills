---
name: auditing-directory-sync-trust
description: >-
  Audit bulk directory synchronization (LDAP, HR-system, IdP, or cross-directory feeds) between an external
  identity source and an application for trust misplaced in the sync feed: a sync that trusts a source attribute
  (group, department, an admin-like flag) to set local privilege or tenancy without validating it, a connector
  authenticated by a broad credential that can read and reshape the whole directory, a mapping that lets an
  external group name land on a privileged internal group, a sync that matches accounts by a spoofable
  key (email, external id) so an attacker record merges into an existing identity, and a source deletion that
  does not propagate so departed users linger. Use when an external directory feed drives account and privilege
  state and the application's trust in that feed is the boundary. The attacker-influenced source record
  is the source, the over-privileged, merged, or lingering internal account is the sink, and the unvalidated
  attribute mapping or spoofable match key is the bug.
license: MIT
---

# Auditing directory sync trust: a bulk feed reshapes identity, so validate what it says before you apply it

Directory synchronization keeps an application's users, groups, and privileges in step with an external source,
an LDAP directory, an HR system, an identity provider, another directory, by importing that source's records in
bulk on a schedule or on events. That makes the sync feed a firehose of identity changes the application applies
largely without a human in the loop, so whatever the source says about who is an admin, who belongs to which
group, and which account is which becomes internal truth. When the application trusts source attributes it never
validates, a group membership, a department, an is-admin-like flag, an email, then whoever can influence those
attributes at the source sets internal privilege. When the sync connector authenticates with a broad or
long-lived credential, that credential can read and reshape the entire directory. When an external group name is
allowed to map onto a privileged internal group, an attacker who controls a source group name escalates. When
accounts are created or matched by a spoofable key like email or external id, an attacker record merges into an
existing identity. And when a deletion or suspension in the source does not propagate, departed users linger
with access. The audit treats the feed as untrusted input, checks that every attribute that sets privilege or
identity is validated, and confirms the connector is scoped and deletions propagate. You audit this by feeding
shaped source records and watching what internal state they create.

## When to use

- An external directory feed (LDAP, HR system, identity provider, or cross-directory sync) drives internal
  account, group, and privilege state in bulk.
- The sync may trust source attributes (group, department, admin flag, email) to set privilege or tenancy
  without validation, or map an external group onto a privileged internal one.
- Accounts may be matched by a spoofable key, the connector credential may be broad, or source deletions may not
  propagate.

## Scope check

Test directory sync trust only against directories and applications you own or are authorized to assess, on
non-production tenants. Feeding records and triggering syncs changes real identity state in bulk, so use test
directories and tenants and never create, merge, elevate, or delete identities that are not yours. If you can't
name the authorization, stop.

## The loop

1. **Establish which source attributes set internal state and what the connector may do first.** Name every
   source attribute the sync trusts to set privilege, group, or tenancy, how accounts are matched between source
   and application, what the connector credential can read and write, and how source deletions are meant to
   propagate. This is the false-positive killer: a sync that validates every privilege-setting attribute, matches
   accounts on a non-spoofable key, runs under a scoped connector credential, and propagates deletions is
   behaving correctly. Name the intended trust, then feed shaped records.

2. **Test attribute-to-privilege mapping.** Feed source records carrying various group, department, and
   admin-like attributes and confirm each maps to exactly the intended internal privilege. Attempt a source
   attribute that should map to an ordinary role and check it cannot set an admin or elevated internal group. A
   sync that trusts an unvalidated source attribute to set privilege lets whoever controls that attribute
   escalate.

3. **Test group name mapping and collision.** Examine how external group names map to internal groups. Try a
   source group whose name collides with or maps onto a privileged internal group, and confirm the mapping is an
   explicit allowlist, not a name match. An external group allowed to land on a privileged internal group is an
   escalation for anyone who can name a source group.

4. **Test account matching and merging.** Determine the key the sync uses to match a source record to an existing
   internal account (email, external id, username). Feed a record with a spoofable key that collides with an
   existing identity and confirm it does not merge an attacker record into that account. Matching on a spoofable
   key lets a crafted source record take over an existing internal identity.

5. **Test connector scope and deletion propagation.** Examine the connector credential: whether it is scoped to
   the objects the sync needs or can read and reshape the whole directory, and whether it rotates. Then delete
   and suspend a user in the source and confirm the change propagates so the internal account is disabled and its
   access revoked. A broad connector credential is a directory-wide compromise, and a deletion that does not
   propagate leaves a departed user active.

6. **Confirm and record.** Confirm on a test tenant by elevating an internal account through an unvalidated
   source attribute, mapping an external group onto a privileged internal one, merging an attacker record into an
   existing identity, or keeping a user active after a source deletion, without touching real identities. Kill the
   lead if the sync validates every privilege-setting attribute, matches on a non-spoofable key, maps groups by
   explicit allowlist, runs under a scoped credential, and propagates deletions. Record the attacker-influenced
   source record, the over-privileged, merged, or lingering internal account, and the unvalidated attribute
   mapping or spoofable match key.

## Where directory sync trust leaks

- **Unvalidated privilege-setting attributes.** A source group, department, or admin-like flag trusted to set
  internal privilege lets whoever influences that attribute escalate in bulk.
- **External-to-privileged group mapping.** An external group name allowed to map onto a privileged internal
  group escalates for anyone who can name a source group.
- **Spoofable match keys.** Matching source records to internal accounts by email or external id lets a crafted
  record merge into and take over an existing identity.
- **Broad connector credential.** A sync connector that can read and reshape the whole directory is a
  directory-wide credential whose compromise reshapes all identity.
- **Deletions that do not propagate.** A source deletion or suspension the sync does not apply leaves a departed
  user active internally with standing access.

## Worked example (a confirm and a kill)

> **Confirm.** An application syncs from an external directory and creates or updates internal group membership
> directly from the source group names, matching accounts by email, under a connector credential with full
> directory read-write. On a test tenant, a source record whose group name matches the internal administrators
> group grants admin, and a record with an existing user's email merges into that user's account, because the
> sync trusts the group name and matches on email. **Confirmed** privilege escalation and account merge via
> unvalidated group mapping and spoofable match key, `high`, remediation = map external groups to internal ones
> by an explicit allowlist that never reaches privileged groups, match accounts on a non-spoofable key, validate
> privilege-setting attributes, and scope the connector credential.
>
> **Kill.** The sync validates every attribute that sets privilege, maps external groups to internal ones only
> through an explicit allowlist with privileged groups unreachable, matches accounts on a non-spoofable stable
> key so no email collision merges identities, runs under a connector credential scoped to the synced objects and
> rotated, and propagates source deletions so a removed user is disabled. A crafted group name does not escalate,
> a spoofed email does not merge, and a deleted source user goes inactive. **Killed**, `kill_reason` = "the sync
> validates privilege-setting attributes, maps groups by allowlist, matches on a non-spoofable key, runs under a
> scoped credential, and propagates deletions; no shaped source record over-privileges, merges, or lingers."

## Rationalizations to reject

- *"The directory is authoritative."* → Authoritative is not the same as validated; confirm the application
  checks each privilege-setting attribute rather than applying whatever the feed asserts.
- *"Group names come from the source."* → A source group name an attacker can set must not map onto a privileged
  internal group; use an explicit allowlist, not a name match.
- *"We match users by email."* → Email is spoofable in a source record; match on a stable non-spoofable key or an
  attacker record merges into an existing account.
- *"The connector needs access to sync."* → It needs access to the synced objects, not the whole directory; a
  broad connector credential is a directory-wide compromise waiting to happen.
- *"Deletions sync eventually."* → Confirm a source deletion actually disables the internal account and revokes
  access; a deletion that does not propagate leaves a departed user active.

## Executing this in practice

You need which source attributes set internal privilege and tenancy, how external groups map to internal ones,
the key used to match accounts, what the connector credential can do, and how deletions propagate. On a test
tenant, feed shaped source records with crafted groups, spoofed match keys, and source deletions, and watch the
internal state each produces. Reading the sync mapping and connector configuration shows the intended trust; an
over-privileged, merged, or lingering internal account from a shaped record shows whether it holds.

## Related

- `auditing-scim-provisioning-trust` - the push-provisioning counterpart; SCIM and directory sync both trust an
  external source to shape internal identity, and both hinge on attribute-to-privilege mapping.
- `auditing-jit-provisioning-and-role-mapping` - the at-login counterpart; the same attribute-to-role validation
  applies whether identity is synced in bulk or provisioned at login.
- `auditing-multi-tenant-isolation` - the tenant scoping a sync must respect; a feed that crosses tenants is that
  skill's boundary applied to synchronization.
- `hunting-iam-privilege-escalation-paths` - a group over-granted by sync is an escalation edge that skill traces
  onward from the internal account it creates.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the attacker-influenced source record, sink = the
  over-privileged, merged, or lingering internal account, evidence = the unvalidated attribute mapping or
  spoofable match key.
