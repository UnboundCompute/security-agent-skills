---
name: hunting-active-directory-acl-abuse-paths
description: >-
  Hunt privilege escalation through Active Directory object permission abuse, where a low-privileged
  principal holds a right over a more privileged object, letting it reset a password, add itself to a group,
  set a delegation or logon script, write a service principal name, or take ownership and rewrite the object's
  permissions, chaining one granted right into control of a higher-privileged account. Use when directory
  object permissions may grant a standard user a write over a privileged user, group, computer, or
  organizational unit. Covers force-password-reset, group membership writes, generic-write and write-owner
  abuse, and delegated control chains. The low-privileged right over a privileged object is the source, the
  directory operation that right authorizes is the sink, and gaining control of the higher-privileged account
  is the bug.
license: MIT
---

# Hunting Active Directory ACL abuse paths: when one granted right owns a privileged account

Every object in an Active Directory carries an access control list, and the rights on it decide who may change
it. Most of those rights are mundane, but a handful hand control of the object to whoever holds them: the
right to reset a password logs you in as that account, the right to write group membership adds you to a
privileged group, a generic write lets you set a logon script or a delegation the account will honor, and
taking ownership lets you rewrite the permissions to grant yourself anything. When one of these rights sits on
a privileged object and is held by a low-privileged principal, that principal owns the object; when the object
it then controls holds a right over a further object, the control chains. The bug is a right over a more
privileged object held by a principal who should not control it. You hunt these by reading the permissions on
privileged objects and tracing each dangerous right to the account it seizes.

## When to use

- Directory object permissions may grant a standard user a write over a privileged user, group, or computer.
- A delegated-control grant on an organizational unit may reach objects more privileged than the grantee.
- A chain of object rights may lead from a low-privileged principal to a privileged account.

## Scope check

Test object-permission abuse only in directories you own or are authorized to assess, on non-production or a
lab domain, making only reversible changes to test objects you are permitted to modify and never seizing a
real privileged account outside the authorization. A confirmed chain yields control of a privileged account,
so keep every probe within scope. If you can't name the authorization, stop.

## The loop

1. **Establish which dangerous right the low-privileged principal actually holds over a privileged object
   first.** For each privileged object, read its permissions and determine whether a low-privileged principal
   holds a right that seizes control: reset-password, write-membership, a generic write, write-owner, or
   take-ownership. This is the false-positive killer: a right to read the object, or a write over an object no
   more privileged than the principal, is not an escalation. Name the specific right and the object it
   controls before claiming a path.

2. **Enumerate privileged objects and their permissions.** List the privileged users, groups, computers, and
   organizational units, and read the principals and rights on each object's access control list. Note every
   right held by a principal less privileged than the object itself.

3. **Classify each dangerous right by what it seizes.** Reset-password takes the account directly;
   write-membership adds the principal to the group; a generic write sets a logon script, a delegation, or a
   service principal name the account will act on; write-owner and take-ownership let the principal become the
   owner and then rewrite the whole access control list. Decide which control each right yields.

4. **Trace the chain.** A right that seizes one object matters more when that object holds a right over a
   further object; follow the edges from the low-privileged principal through each seized object to the
   highest-privileged account reachable. The finding is the whole path, not only the first hop.

5. **Check the constraints that break the chain.** Protected accounts whose permissions are reset by policy,
   inheritance blocked at a boundary, or a right that looks dangerous but is scoped to a property that grants
   no control can stop a link. Determine whether each hop actually authorizes the operation it appears to,
   against the object's live permissions.

6. **Confirm and record.** Confirm one link with a reversible change to a test object you are permitted to
   modify on an isolated lab domain, observing the control the right grants, then undo it. Kill the lead if the
   principal holds no controlling right over a more privileged object, if a protected-account policy resets
   the permission, or if the apparent right is scoped to a property that grants no control. Record the
   principal, the right, the object, the chain to the privileged account, and the reversible proof, or set a
   `kill_reason`.

## Where ACL abuse leaks

- **The dangerous right is the finding.** Reset-password, write-membership, generic write, and write-owner
  each seize the object; a benign read or a scoped property write does not. Name which right and which object.
- **Delegated control on an organizational unit reaches its contents.** A control grant meant for a helpdesk
  role can reach privileged objects that happen to sit under the same container.
- **Generic write is indirect but total.** A generic write sets a logon script, a delegation, or a service
  principal name the account will honor, turning a write into execution or authentication as that account.
- **Write-owner rewrites the rest.** Becoming the owner lets the principal grant itself every other right, so
  a single write-owner is control of the whole access control list.
- **The chain is the point.** One low-privileged right over a mid-tier object that itself controls a
  privileged account is a full escalation; trace the path end to end, not just the first edge.

## Worked example (a confirm and a kill)

> **Confirm.** A low-privileged principal holds the reset-password right over an account that is a member of a
> group with delegated control over a privileged organizational unit. Resetting the intermediate account's
> password on an isolated lab domain yields its context, from which the delegated control reaches a privileged
> object, all with reversible changes to test objects. **Confirmed** privilege escalation through an object-
> permission chain, `high`, remediation = remove the reset-password and control rights from principals that
> do not require them, enforce protected-account policy on privileged objects, and audit delegated control on
> organizational units that contain privileged objects.
>
> **Kill.** The same privileged objects grant the low-privileged principal read rights only, protected-account
> policy resets their permissions on a schedule, and the delegated control on the organizational unit is
> scoped to non-privileged property sets that grant no account control. No held right seizes a more privileged
> object. **Killed**, `kill_reason` = "the principal holds only read and scoped-property rights over more
> privileged objects, and protected-account policy resets any drift; no right seizes control of a privileged
> account."

## Rationalizations to reject

- *"They can only reset a password, not log in."* -> Resetting the password sets a credential the principal
  then uses to authenticate as that account; the reset is the takeover.
- *"It is just a generic write."* -> A generic write sets a logon script, a delegation, or a service principal
  name the account honors, so it is control of the account, not a harmless attribute edit.
- *"That right is on a low-tier account."* -> Trace what the low-tier account controls; a chain from it can
  reach a privileged object even when the first hop looks unimportant.
- *"Delegated control is for the helpdesk."* -> If the delegation's scope includes a container that holds
  privileged objects, the helpdesk grant reaches them; confirm the scope, not the intent.
- *"Inheritance protects those objects."* -> Confirm inheritance is actually blocked and protected-account
  policy is enforced; drifted permissions and disabled protection are exactly what this hunt finds.

## Executing this in practice

You need the access control list of every privileged object, the rights each low-privileged principal holds,
and the delegated control on organizational units that contain privileged objects. For each dangerous right,
name what it seizes and trace the chain to the highest-privileged account reachable, checking each hop against
the live permissions. Reading the permissions and following the edges settles most leads; a reversible change
to a permitted test object that demonstrates one link on an isolated lab domain settles the rest.

## Related

- `hunting-adcs-certificate-template-abuse` - a write over a certificate template or its enrollment rights is
  one of the object-permission abuses this skill traces, leading into that certificate escalation.
- `hunting-kerberos-and-ad-delegation-abuse` - a generic write that sets a service principal name or a
  delegation feeds the ticket-based escalation that skill pursues.
- `auditing-group-policy-and-sysvol-trust` - a write over a group policy object or its link is an object-
  permission abuse whose payload that skill covers from the policy-content side.
- `auditing-windows-token-and-privilege-abuse` - once a chain yields a privileged account, the privileges that
  account holds are the next lever, which that skill audits.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the low-privileged right over a privileged object,
  sink = the directory operation the right authorizes, evidence = control of the higher-privileged account
  demonstrated by a reversible change on an isolated lab domain.
