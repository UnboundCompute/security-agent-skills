---
name: hunting-formula-injection-in-exports
description: >-
  Hunt formula injection, also called CSV injection, where an untrusted field stored by the application
  is later written into an exported CSV, TSV, or spreadsheet and a spreadsheet program evaluates it as a
  formula when a victim opens the file. A cell whose first character is an equals, plus, minus, or at
  sign, or a tab or carriage-return prefix, is treated as a formula, so a stored value can call a data
  connection or hyperlink to exfiltrate other cells, trigger a legacy dynamic-data command that runs a
  program, or spoof content the recipient trusts. The vulnerable step is the export, not the page that
  stored the value, and the victim is whoever opens the download. Use when the app exports user-controlled
  data to a spreadsheet format. The stored untrusted field is the source, the exported cell a spreadsheet
  evaluates is the sink, and formula execution in the recipient's client is the bug.
license: MIT
---

# Hunting formula injection in exports: when a cell runs code on open

An application that lets users store text and later offers an export to a spreadsheet format has quietly
built a delivery channel. Spreadsheet programs decide whether a cell is data or a formula by its first
character: a leading equals, plus, minus, or at sign marks a formula, and so does a leading tab or
carriage return that the program strips before parsing. So a value a user typed into a profile field, a
comment, or a device name, stored verbatim and written into a downloaded report, becomes live code when
an administrator opens that report. The formula can pull other cells out to an attacker URL through a web
or hyperlink function, invoke a legacy dynamic-data exchange command that launches a program, or display
misleading content the recipient trusts. The bug lives in the export writer, not the input page, and the
victim is the person who opens the file. You find it by tracing stored untrusted fields into every export
and checking whether dangerous leading characters are neutralized.

## When to use

- The application exports user-controlled data to CSV, TSV, or a spreadsheet workbook format.
- A stored field (name, comment, description, tag) is written into a downloaded report opened by staff.
- An admin, finance, or analytics workflow opens exports of user-submitted content in a spreadsheet.

## Scope check

Test formula injection only against applications you own or are authorized to assess, and open confirming
exports only in an isolated spreadsheet environment with no network egress and no sensitive cells, because
a live formula can call out or launch a program on the machine that opens it. Never plant a payload where a
real recipient would open it. If you can't name the authorization, stop.

## The loop

1. **Establish that the export writes cells without neutralization first.** Locate every code path that
   produces a CSV, TSV, or workbook export and read how it writes each cell. This is the false-positive
   killer: if the writer prefixes at-risk cells with a safe character, wraps values so the leading token is
   inert, forces the cell type to text, or strips leading formula characters, a stored value cannot become
   a formula and there is no bug. Name the exports that write stored values verbatim.

2. **Inventory the stored fields that reach the export.** Trace which user-controlled fields flow into
   exported cells: profile and account fields, free-text comments, names of user-created objects, imported
   values. The source is stored, so a value planted through one feature surfaces in an unrelated report.

3. **Check the leading-character trigger.** Determine whether a stored value can begin with equals, plus,
   minus, or at sign, or with a tab or carriage return that the spreadsheet strips before parsing. These
   are the characters that flip a cell from data to formula; if input validation forbids all of them at
   storage time across every entry path, the trigger is closed.

4. **Map the reachable formula impact.** Identify what a formula can do in the recipient's environment: a
   web or hyperlink function that fetches an attacker URL with other cell contents appended, exfiltrating
   data; a dynamic-data exchange command that asks to run a local program; a formula that renders spoofed
   or misleading content. The severity depends on which functions and prompts the recipient's client
   allows, so state the assumed client behavior.

5. **Check the neutralization that actually holds.** The reliable fix is at export time: prefix any cell
   beginning with a formula character with a leading single quote or a space, or force a text cell type in
   a workbook format, applied to every export path. Input-time filtering helps but is fragile because
   second-order and imported data can bypass it. Determine which defense stands and whether it covers all
   exports.

6. **Confirm and record.** Confirm by storing a benign observable formula (for example, one that
   concatenates a constant or requests a URL you control) and opening the resulting export in an isolated,
   egress-controlled spreadsheet, showing the cell evaluates. Kill the lead if every export neutralizes
   leading formula characters or forces text cells, if storage strictly rejects those leading characters on
   all paths, or if no export of the field exists. Record the stored field, the export path, the trigger
   character, and the formula behavior observed. Set `kill_reason` when killing.

## Where formula injection leaks

- **The vulnerable code is the export, not the form.** Reviewing the input page shows nothing; the writer
  that emits the cell is where the value becomes a formula.
- **The victim is a third party.** Whoever opens the download runs the formula, often an administrator with
  more access than the attacker, which is why exfiltration through a cell reference matters.
- **Stripped whitespace re-arms the payload.** A leading tab or carriage return looks harmless in the file
  but is removed by the spreadsheet before parsing, exposing the formula character behind it.
- **Second-order storage evades input filters.** Values imported, synced, or set through an API can skip the
  form validation, so only export-time neutralization is dependable.
- **Legacy dynamic-data commands escalate to code.** Where the client still honors a dynamic-data exchange
  formula, an open-and-confirm prompt can launch a program, turning a stored string into execution.

## Worked example (a confirm and a kill)

> **Confirm.** A support console exports a ticket report as CSV, writing the reporter's display name into a
> cell verbatim. A user sets their display name to a formula that uses a web function to request an
> attacker URL with a neighboring cell appended. When an agent opens the report in a spreadsheet, the cell
> evaluates and the request carries another ticket's contents off the machine. **Confirmed** formula
> injection to data exfiltration, `medium`, remediation = at export prefix any cell starting with a formula
> character with a single quote or force a text cell type, on every export path.
>
> **Kill.** A billing export writes every cell through a helper that prepends a single quote to any value
> beginning with equals, plus, minus, at, tab, or carriage return, applied uniformly across all export
> endpoints. A stored value beginning with a formula character opens as literal text and never evaluates.
> **Killed**, `kill_reason` = "export writer neutralizes all formula-triggering leading characters for every
> export path, so no stored value is parsed as a formula."

## Rationalizations to reject

- *"We sanitize input on the form."* -> Values arrive by import, sync, and API too; only export-time
  neutralization covers second-order data, so the form filter is not enough.
- *"It is only a CSV, not code."* -> The spreadsheet that opens it evaluates formulas, and web, hyperlink, and
  dynamic-data functions reach the network and the shell from a cell.
- *"The attacker only harms their own row."* -> A formula reads other cells and sends them out; the payload
  exfiltrates the report's contents, not the attacker's own field.
- *"Modern spreadsheets warn the user."* -> Warnings are dismissed and vary by client and function; the
  robust control is on your side at export, not the recipient's prompt.
- *"No one opens these files."* -> Exports exist to be opened, usually by staff with elevated access; that is
  precisely the victim you are protecting.

## Executing this in practice

You need every export path and the exact cell-writing code, the stored fields that flow into cells, and
whether any neutralization is applied at write time. For each export, decide whether a stored value can
begin with a formula-triggering character, whether the writer neutralizes or forces text, and what a
formula could do in the recipient's client. Reading the writer shows the neutralization; opening a benign
stored formula in an isolated, egress-controlled spreadsheet shows whether the cell evaluates.

## Related

- `auditing-file-upload-and-content-handling` - the sibling for content that is dangerous to the consumer;
  here the export, not an upload, is the artifact whose interpretation causes harm.
- `hunting-reflected-and-stored-xss` - another stored-then-rendered class where the sink is a downstream
  interpreter; the spreadsheet, rather than a browser, evaluates the payload here.
- `reviewing-rate-limiting-and-abuse-controls` - stored payloads planted in bulk across records are an abuse
  pattern that skill helps bound.
- `adjudicating-taint-paths` - use it to connect a stored field to the export cell across the storage and
  reporting layers.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the stored untrusted field, sink = the exported
  cell a spreadsheet evaluates, evidence = a benign stored formula evaluating on open in an isolated,
  egress-controlled spreadsheet.
