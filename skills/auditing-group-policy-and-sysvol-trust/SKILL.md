---
name: auditing-group-policy-and-sysvol-trust
description: >-
  Audit trust placed in group policy content and the domain policy share, where a low-privileged principal
  can write a policy object, its files on the domain share, or its link, causing that policy to run scripts,
  set scheduled tasks, install software, or change security settings on every machine the policy applies to,
  or where a stored credential or an unsigned policy file lets any domain user act on the policy. Use when
  policy objects, their share files, or their links may be writable by non-administrators or carry stored
  secrets. Covers writable policy content, writable links and scope, embedded credentials, and unsigned or
  world-readable policy files. The low-privileged write to policy content or link is the source, the machines
  applying the policy are the sink, and running attacker-chosen policy on those machines is the bug.
license: MIT
---

# Auditing group policy and SYSVOL trust: when a writable policy runs on every machine

Group policy is a domain-wide instruction stream: a policy object names settings, scripts, scheduled tasks,
and software, its files live on a share every domain machine reads, and its links decide which machines apply
it. Machines apply that policy with high privilege and largely without questioning it. That is safe only when
nobody below a domain administrator can change what a policy says, where its files live, or which machines it
targets. When a low-privileged principal can write a policy object, edit its files on the share, or alter its
link, they run their own scripts and tasks on every machine in scope. A second failure hides in the content
itself: a credential stored in a policy file that every domain user can read, or an unsigned policy file a
machine applies without verifying its origin. The bug is trust in policy content or scope that a
non-administrator can influence. You audit these by reading who can write each policy, its share files, and
its links, and what the content discloses.

## When to use

- Policy objects, their files on the domain share, or their links may be writable by non-administrators.
- A policy file on the share may carry a stored credential readable by any domain user.
- Policy files may be applied without verifying their origin or may be world-readable on the share.

## Scope check

Audit policy and share trust only in directories you own or are authorized to assess, on non-production or a
lab domain, making only reversible changes to test policy objects you are permitted to modify and never
altering a production policy. A confirmed write runs code on every targeted machine, so keep every probe
within scope. If you can't name the authorization, stop.

## The loop

1. **Establish who can actually write the policy, its files, and its link first.** For each policy object,
   read the permissions on the object, on its files on the domain share, and on its links, and determine
   whether a low-privileged principal holds a write to any of them. This is the false-positive killer: a
   policy whose object, share files, and links are writable only by administrators cannot be turned against
   the machines it applies to, however powerful its content. Name the specific write before claiming a path.

2. **Enumerate policies, their scope, and their content.** List the policy objects, the machines and users
   each applies to through its links, and the actions each carries: scripts, scheduled tasks, software
   installation, and security settings. A writable policy with a broad scope and executable content is the
   highest-value candidate.

3. **Check the write to content.** Determine whether a low-privileged principal can edit the policy object or
   its files on the share to add or change a script, a scheduled task, or an installed package. A write here
   runs attacker-chosen actions on every machine in scope at the next application.

4. **Check the write to scope.** Determine whether a low-privileged principal can create or alter a link, so a
   policy they control, or an existing policy, applies to machines it should not, extending the reach of a
   writable policy or pulling a benign machine under a malicious one.

5. **Check the content for disclosed secrets and unverified origin.** Determine whether any policy file on the
   share carries a stored credential readable by domain users, and whether machines apply policy files
   without verifying their origin, so a tampered or planted file is honored. Both let a low-privileged user
   act even without a direct write to the object.

6. **Confirm and record.** Confirm by making a reversible change to a test policy you are permitted to modify
   and observing a benign marker action run on a targeted test machine on an isolated lab domain, or by
   reading a disclosed credential from the share. Kill the lead if the policy object, its share files, and its
   links are all admin-only writable, if no policy file discloses a usable secret, and if applied files are
   origin-verified. Record the policy, the write or disclosure, the scope, and the benign proof, or set a
   `kill_reason`.

## Where policy and share trust leaks

- **The write is the finding, not the policy's power.** Policy running with high privilege is by design; a
  non-administrator write to its content, files, or link is the escalation. Name the write.
- **Writable content runs everywhere in scope.** A script or scheduled task added to a writable policy runs on
  every machine the policy applies to, turning one write into fleet-wide execution.
- **Writable links redraw the target.** Control over a link lets an attacker apply a policy they control to
  new machines or pull machines under a malicious policy, extending reach without touching content.
- **Stored credentials on the share are readable by all.** A credential embedded in a policy file on a share
  every domain user can read is disclosed to every domain user, independent of any write.
- **Unverified policy files are trusted blindly.** A machine that applies policy files without checking their
  origin honors a tampered or planted file, so integrity of the share content is part of the trust.

## Worked example (a confirm and a kill)

> **Confirm.** A policy object applied to a broad set of machines has files on the domain share writable by a
> low-privileged principal. Adding a benign marker scheduled task to the policy files causes that task to run
> on a targeted test machine at the next policy application on an isolated lab domain, reversed afterward.
> **Confirmed** fleet-wide code execution through writable policy content, `high`, remediation = restrict the
> policy object, its share files, and its links to administrators, verify the origin of applied policy files,
> and remove any stored credentials from policy content.
>
> **Kill.** The same policy object, its files on the share, and its links are writable only by administrators,
> no policy file on the share carries a readable credential, and machines verify the origin of policy files
> before applying them. A low-privileged principal can change neither the content, the scope, nor a disclosed
> secret. **Killed**, `kill_reason` = "policy object, share files, and links are all admin-only writable, no
> credential is disclosed, and applied files are origin-verified; no low-privileged influence reaches the
> machines."

## Rationalizations to reject

- *"Only admins edit group policy."* -> Read the actual permissions on the object, its share files, and its
  links; delegated edit rights and loosened share permissions are exactly what this audit finds.
- *"The share is read-only to users."* -> Read-only still discloses any credential stored in a policy file to
  every domain user; and confirm write permissions per file rather than assuming a blanket read-only.
- *"That policy only sets a wallpaper."* -> Check every action the policy carries and whether the content is
  writable; a writable benign policy can be given a script or task.
- *"Links cannot be changed by users."* -> Confirm the link and container permissions; a delegated right over
  an organizational unit can let a non-administrator alter which policies apply.
- *"Policy files are trusted because they are on the domain share."* -> Trust must come from verified origin,
  not location; a writable or planted file on the share is honored unless its origin is checked.

## Executing this in practice

You need every policy object with its permissions, the permissions on its files on the domain share and on its
links, the machines each policy applies to, the actions it carries, and any credential stored in its content.
For each, decide whether a low-privileged principal can change the content, the scope, or read a disclosed
secret, and whether applied files are origin-verified. Reading the permissions and content settles most leads;
a reversible change to a permitted test policy that runs a benign marker on a targeted machine on an isolated
lab domain settles the rest.

## Related

- `hunting-active-directory-acl-abuse-paths` - a write over a policy object or its link is one of the object-
  permission abuses that skill traces, feeding the policy write this one exploits.
- `hunting-windows-credential-material-exposure` - a credential stored in a policy file on the share is the
  kind of exposed material that skill pursues from the access side.
- `hunting-windows-service-privilege-escalation` - a policy-installed scheduled task or service is another
  route to code execution as a privileged account, sharing the planted-execution shape.
- `auditing-workload-secret-exposure-surface` - secrets embedded in distributed policy files are the exposure
  that skill audits across a workload, here in the domain-share form.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the low-privileged write to policy content or link,
  sink = the machines applying the policy, evidence = a benign marker action running on a targeted machine, or
  a disclosed credential read from the share, on an isolated lab domain.
