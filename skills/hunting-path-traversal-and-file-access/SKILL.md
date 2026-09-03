---
name: hunting-path-traversal-and-file-access
description: >-
  Hunt path traversal and unsafe file access where untrusted input builds a filesystem path and the
  resolved path escapes the intended base directory, so the process reads or writes files it should not.
  Covers download and upload names, template and include paths, static-file routes, and archive members
  driven to dot-dot or absolute or drive-letter targets, encoded and double-encoded separators, null-byte
  truncation on legacy stacks, symlink following, and the difference between checking the raw input and
  checking the canonicalized absolute path. Use when input becomes a path passed to an open, read, write,
  include, or send-file call. The untrusted path input is the source, the filesystem open or write is the
  sink, and reaching a file outside the intended directory is the bug.
license: MIT
---

# Hunting path traversal and file access: when a name escapes its directory

Applications constantly turn a name into a file: a download parameter into a file to send, an upload name
into a destination to write, a template name into an include, a route into a static asset. The danger is
that a filesystem path is hierarchical, and a name carrying dot-dot segments, an absolute prefix, or a
drive letter can climb out of the directory the code meant to stay in. When that happens a read reaches
secrets, configuration, or source outside the intended base, and a write lands a file, sometimes an
executable one, where it should never go. The reliable defense is to resolve the path to a canonical
absolute form and then confirm it is still under the intended base, but many applications check the raw
input before decoding or normalizing, which an encoded or layered payload walks straight past. You find
these by locating every path built from input and asking whether confinement is verified after
canonicalization.

## When to use

- Untrusted input names a file to read, write, include, or send: downloads, uploads, templates, static routes.
- Archive extraction or a bulk import writes members to paths derived from the archive.
- A path is validated against a base directory and you need to confirm the check survives normalization.

## Scope check

Test path traversal only against applications you own or are authorized to assess, on non-production data,
because a confirming read exposes real files and a confirming write can corrupt or plant content. Read a
benign marker file and write to a scratch location for any proof, and prefer an isolated instance. If you
can't name the authorization, stop.

## The loop

1. **Establish whether confinement is checked after canonicalization first.** For each path built from
   input, find where and how the code confirms the target stays under the intended base. This is the
   false-positive killer: resolving to a canonical absolute path and verifying it is a descendant of the
   base directory holds, while a check on the raw input before decoding or normalization, or a blocklist of
   dot-dot, is bypassable. Name the check and when it runs relative to normalization.

2. **Trace input into the path.** Follow parameters, headers, upload names, and archive members into the
   string passed to the file call. Confirm the attacker controls path-significant bytes, not just a
   trailing name the code pins under a fixed directory.

3. **Test the traversal shapes.** Try dot-dot segments, encoded and double-encoded separators, mixed
   separators on the target platform, an absolute path or a drive letter that replaces the base entirely,
   and, on legacy stacks, a null byte that truncates an appended extension. Identify which shape reaches
   outside the base.

4. **Check symlink and resolution behavior.** Determine whether the resolved path follows a symlink out of
   the base, whether the canonicalization the check uses matches the one the file call uses, and whether a
   time-of-check to time-of-use gap lets the target change between the check and the open.

5. **Separate read from write impact.** For a read, determine what sensitive files the escape reaches. For a
   write, determine whether the attacker chooses both the path and the content, and whether the destination
   is executed or served, which turns traversal into code execution or a served payload.

6. **Confirm and record.** Confirm a read by retrieving a benign marker file outside the base, a write by
   landing a scratch file outside the base, on an isolated instance. Kill the lead if the resolved canonical
   path is verified to stay under the base, if input is confined to a name with no path separators under a
   fixed directory, or if the file call cannot reach a sensitive location. Record the sink, the input, the
   traversal shape, and what it reached. Set `kill_reason` when killing.

## Where path traversal leaks

- **The check often runs before decoding.** A raw-input check for dot-dot misses an encoded or
  double-encoded separator that a later decode restores; only a post-canonicalization check holds.
- **Absolute and drive-letter inputs replace the base.** Joining a base with an absolute path or a drive
  letter can discard the base entirely on common platforms, so the file call starts from the attacker's root.
- **Writes are worse than reads.** Choosing the destination and the content of a write can plant an
  executable or a served file, escalating disclosure into execution.
- **Symlinks and canonicalization mismatches slip through.** If the check and the open resolve the path
  differently, or a symlink points out of the base, confinement verified on one form fails at the other.
- **Archive members are attacker-named paths.** Extracting a member to a path from the archive is the same
  escape without a request parameter; a member named with dot-dot writes outside the extraction root.

## Worked example (a confirm and a kill)

> **Confirm.** A download endpoint joins a base directory with a request-supplied filename and opens the
> result, checking the raw parameter for dot-dot but decoding percent-encoding afterward. A double-encoded
> separator passes the check, decodes to dot-dot segments, and the open reads a configuration file outside
> the base containing credentials. **Confirmed** path traversal to sensitive file disclosure, `high`,
> remediation = resolve the joined path to a canonical absolute form and reject it unless it is a descendant
> of the intended base, decode before checking, and confine input to a bare filename with no separators.
>
> **Kill.** A different endpoint resolves the candidate path to its canonical absolute form, verifies it is
> under the fixed base after all decoding, rejects any input containing a separator, and opens files with
> symlink resolution that stays within the base. A dot-dot or encoded payload resolves outside the base and
> is rejected before any open. **Killed**, `kill_reason` = "path is canonicalized and verified to stay under
> the base after decoding, input is confined to a separator-free name, so no traversal shape reaches outside
> the directory."

## Rationalizations to reject

- *"We block dot-dot."* -> A blocklist misses encoded, double-encoded, and mixed-separator forms and does
  nothing against an absolute or drive-letter input; confinement must be verified on the canonical path.
- *"We check the filename before using it."* -> If the check runs before decoding or normalization, a layered
  payload is restored afterward; the check must run on the final resolved path.
- *"It only reads, it cannot write."* -> A read of configuration, keys, or source is already high impact;
  and confirm the same input does not reach a write or include sink elsewhere.
- *"The base directory is fixed."* -> Joining a fixed base with an absolute path or drive letter can discard
  the base entirely on common platforms; a fixed base is not confinement.
- *"Extraction is from our own archives."* -> If any archive is attacker-influenced, a member named with
  dot-dot writes outside the root; treat archive members as untrusted paths.

## Executing this in practice

You need every path built from input, the file call it feeds, and the confinement check with its position
relative to decoding and canonicalization. For each, decide whether the resolved canonical path is verified
to stay under the intended base and whether input is confined to a separator-free name. Reading the code
tells you when the check runs and how the path resolves; retrieving a marker file or landing a scratch file
outside the base on an isolated instance tells you whether the escape holds.

## Related

- `hunting-unsafe-archive-extraction` - the archive-member variant of the same escape, where a member named
  with dot-dot writes outside the extraction root; the confinement question is identical.
- `auditing-file-upload-and-content-handling` - upload destinations and served files are a common path sink;
  that skill covers the content and storage side.
- `hunting-server-side-and-edge-side-includes` - an include path built from input is a traversal sink that
  can also trigger directive processing; adjacent concern.
- `adjudicating-taint-paths` - use it to connect an input to the exact file call through any join, decode, or
  normalization on the way.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the untrusted path input, sink = the filesystem
  open, write, or include, evidence = a benign marker file read or a scratch file written outside the intended
  base on an isolated instance.
