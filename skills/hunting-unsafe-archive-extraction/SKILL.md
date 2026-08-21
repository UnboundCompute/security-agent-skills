---
name: hunting-unsafe-archive-extraction
description: >-
  Hunt unsafe extraction of untrusted compressed archives: an entry's declared path escaping the
  destination directory (traversal or an absolute path), a symlink or hardlink entry that a later
  entry writes through to reach outside, decompression amplification where a small archive expands
  to exhaust disk or memory, and content or name confusion where an extracted file is later
  executed, served, or loaded. Covers import, restore, plugin-install, and upload features that
  unpack archives from users or remote sources, and the difference between checking a path and
  checking the path a symlink resolves to. Use when reviewing code that extracts an archive whose
  contents come from an untrusted source. The archive entry's path, link target, or declared size
  is the source, the filesystem write or allocation during extraction is the sink, and the missing
  containment check is the bug.
license: MIT
---

# Hunting unsafe archive extraction: when unpacking a file writes outside the box

Extracting an archive feels like a read, but it is a series of attacker-directed writes. Each entry
carries a name, sometimes a link target, and a declared size, and a naive extractor trusts all three:
it joins the name to the destination and writes, follows a link, and allocates whatever the entry
claims. A crafted archive uses that trust to write outside the destination with a traversal path or a
symlink, to overwrite a file that matters, or to expand a few kilobytes into enough data to exhaust
disk or memory. You find these by tracing each entry's path, link, and size to the write it drives and
asking what confirms the write stays inside and stays bounded.

## When to use

- Code extracts a user-supplied or externally fetched archive to disk (import, restore, install, upload).
- Entry paths, symlinks, or declared sizes from the archive are trusted while writing.
- A written file is later executed, served, loaded, or read back as configuration.

## Scope check

Unpack crafted archives only into systems and paths you own or are authorized to test, and in a
sandbox where a write outside the destination cannot damage anything real. A path-escape proof can
overwrite live files. If you can't name the authorization, stop.

## The loop

1. **Map where untrusted archives are extracted.** Inventory the code paths that unpack a compressed
   archive whose contents come from a user or a remote source. Note the intended destination directory
   for each and whether the same routine handles nested archives. These extraction points are where an
   entry's metadata becomes a filesystem operation.

2. **Trace each entry path to its write destination.** For every entry, follow how its declared name is
   turned into a path and written. If the name is joined to the destination without normalizing it and
   confirming the result is still inside, a `../` sequence or an absolute path writes outside the
   destination. The check has to be on the final resolved path, not on the raw name.

3. **Check for symlink and hardlink escape.** An archive can carry a symlink entry that points outside
   the destination, followed by a later entry whose path resolves through that link. Even when regular
   files are path-checked, a write that follows a link the archive created lands wherever the link
   points. Determine whether links are extracted at all and whether writes follow them.

4. **Check decompression amplification.** Determine whether total extracted size, per-entry size, entry
   count, and nesting depth are bounded during extraction. A small archive can declare or expand to an
   enormous size, and an archive of archives multiplies it. Without a running cap that aborts, unpacking
   exhausts disk or memory from a tiny upload.

5. **Check what happens to the extracted files.** An entry named to land on an existing file overwrites
   it; an entry with an attacker-chosen name or extension that is later executed, served as content,
   loaded as a library, or read as configuration turns a write into code or trust. Trace where extracted
   files go after the unpack, not only where they land.

6. **Confirm and record.** Confirm by extracting a crafted archive in a sandbox and showing a file
   written outside the destination, a link followed out of it, a bounded input expanding past a safe
   limit, or an extracted file reaching an execution or load path. Kill the lead if every entry path is
   resolved and confirmed inside the destination, links are not extracted or not followed, total and
   per-entry size and depth are capped with an abort, and extracted files never reach an execution,
   serve, or load sink. Record with the entry, the resolved path, and where the write landed.

## Where archive extraction leaks

- **The name is attacker data, not a filename.** An entry path is chosen by whoever built the archive.
  Joining it to a destination without resolving and containing it is a write-anywhere primitive.
- **A path check that ignores links is only half a check.** Validating regular-file paths while
  following symlinks the archive itself supplied lets a write escape through the link.
- **Declared size is a promise the attacker makes.** Trusting per-entry or total size lets a small
  archive expand without bound; the cap has to be enforced as data is written, not read from a header.
- **Overwrite is as damaging as escape.** An entry aimed at an existing configuration, credential, or
  startup file changes behavior without ever leaving the destination directory.
- **The danger is often after extraction.** A safe write of a file that is later executed, served, or
  loaded is still a compromise; follow the file past the unpack.

## Worked example (a confirm and a kill)

> **Confirm.** An import feature unpacks a user-uploaded archive by joining each entry name to the
> import directory and writing, with no path resolution. A crafted archive whose entry name is a
> traversal sequence writes a file into a directory outside the import root that the service loads at
> startup, turning an upload into persistent code execution. **Confirmed** archive path traversal to
> code execution, `critical`, remediation = resolve every entry to an absolute path and reject any that
> is not strictly inside the destination, refuse link entries, and cap total and per-entry size and
> depth.
>
> **Kill.** The extractor resolves each entry to its final path and skips any that escapes the
> destination, does not extract symlinks or hardlinks, enforces a running total-size, per-entry, count,
> and depth limit that aborts the unpack, and writes into a quarantine directory nothing executes or
> loads from. Crafted traversal, link, and bomb archives are all rejected or contained. **Killed**,
> `kill_reason` = "entry paths resolved and contained, links not extracted, size and depth capped with
> abort, output quarantined away from any execution or load path."

## Rationalizations to reject

- *"We only accept archives from logged-in users."* → An authenticated user is still an untrusted
  archive author. The entry names are attacker data regardless of who uploaded them.
- *"We check the filename for `..`."* → Substring checks miss absolute paths, encoded traversal, and
  symlink escape. Resolve the final path and confirm containment instead.
- *"The archive is only a few kilobytes."* → Compression ratio, not archive size, decides the damage;
  a tiny archive can expand to fill a disk, and nested archives multiply it.
- *"Extraction just writes files, it does not run anything."* → Until something later executes, serves,
  or loads one of those files. A write into the wrong directory is an execution primitive.
- *"The library handles this safely."* → Only if it resolves paths, refuses links, and bounds size by
  default, and only if you called it that way. Confirm the behavior; do not assume it.

## Executing this in practice

You need the extraction points for untrusted archives, the intended destination for each, how entry
names and link targets become paths, what bounds size and depth during the unpack, and where extracted
files go afterward. For each, ask whether a crafted entry can escape the destination, follow a link out,
expand without limit, or reach an execution or load sink. Reading the extractor shows the intent;
unpacking a crafted archive in a sandbox shows where the writes actually land.

## Related

- `hunting-redos-and-complexity-dos` - a decompression bomb is the resource-exhaustion sibling: small
  input, super-linear cost, missing bound.
- `hunting-dynamic-linker-hijacks` - an extracted file that later gets loaded as a library is the
  write-then-load chain this skill hands off to.
- `hunting-business-logic-flaws` - trusting attacker-declared metadata (a path, a size) is the same
  missing-check pattern the logic hunt looks for.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the entry path, link target, or declared size,
  sink = the filesystem write or allocation during extraction, evidence = where the write landed.
