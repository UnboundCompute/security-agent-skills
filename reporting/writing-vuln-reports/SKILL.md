---
name: writing-vuln-reports
description: >-
  Turn a confirmed finding into a clear, reproducible vulnerability report a
  maintainer or triager can act on without a back-and-forth. Use after a finding
  is confirmed (via the finding schema) and you need a writeup — a bug-bounty
  submission, a security advisory, an internal ticket, or a disclosure email.
  Covers the report structure that gets findings fixed, writing a reproduction
  that actually reproduces, justifying severity honestly, and the disclosure
  etiquette that keeps you in bounds.
license: MIT
---

# Writing vulnerability reports

A finding that isn't clearly reported doesn't get fixed. The job of a report is
to move a specific defect from your head into the maintainer's, with enough proof
that they can reproduce it in minutes and enough impact that they prioritize it —
and nothing else. This skill turns a schema-shaped finding into that report.

## When to use

- A finding is `confirmed` (see [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md)) and
  you're writing it up for a human: bounty submission, advisory, ticket, email.
- You have several findings and need consistent, triager-friendly writeups.

## Scope check

Only report findings from authorized testing, to the party entitled to receive
them (the program's channel, the maintainer's security contact, your client).
Don't disclose someone else's data or a third party's system you weren't scoped
to touch. When in doubt about the channel, ask before sending — a report is
outward-facing and hard to unsend.

## The structure that gets findings fixed

Lead with impact, prove it fast, make the fix obvious. Sections:

1. **Title** — the specific defect and where. "IDOR in `admin_bulk_delete` lets
   any user delete any post," not "Access control issue."
2. **Summary** (2–3 sentences) — what the bug is, who can trigger it, what they
   get. A busy triager should grasp severity from this alone.
3. **Impact** — the concrete consequence, tied to the target's threat model. Who
   is harmed, what they lose, what precondition is needed (auth level, timing,
   config). Don't inflate; don't undersell.
4. **Steps to reproduce** — numbered, exact, copy-pasteable. Real request(s) with
   method, path, headers that matter, and body. State the starting state (logged
   in as user A, two accounts, a seeded record). A stranger must reproduce it
   from these steps alone.
5. **Proof** — the request/response pair (or crash/log) that shows it firing.
   Redact secrets and unrelated PII. For whitebox findings, include the
   source→sink path and the decisive code excerpt from the schema's `evidence`.
6. **Root cause** — the actual defect, at the decisive hop (the missing check,
   the unbounded copy, the non-atomic act). One or two sentences.
7. **Remediation** — the fix at that hop, plus the general pattern
   ("parameterize the query," "resolve against base and reject escapes," "atomic
   conditional update"). Offer the fix; don't demand a specific patch.
8. **Severity** — a rating with the reasoning. If the program uses CVSS, give the
   vector and score; otherwise a justified low/med/high/critical. The reasoning
   matters more than the number.
9. **References** (optional) — CWE id, relevant advisory, similar prior fixes.

## From schema to report

The finding schema already holds most of this. Map it: `title`→title;
`source`+`sink`+`impact`→summary/impact; `path`+`evidence`→reproduction/proof;
`kill_reason` never ships (killed findings aren't reported); `remediation`→
remediation; `severity`→severity; `commit`→pin the "affected version." If a field
is thin, that's a signal the finding isn't report-ready — go back and fill it,
don't paper over it.

## Reproduction is the report

Most rejected or slow-triaged reports fail here. Rules:

- **Reproduce it yourself from your own written steps**, in a clean state, before
  sending. If your steps don't reproduce for you, they won't for them.
- **Minimize.** Strip every step and header that isn't load-bearing. The shortest
  path that still fires is the best path.
- **Make preconditions explicit.** "Two accounts," "a coupon that hasn't been
  redeemed," "run the two requests concurrently." Hidden state is the #1 reason a
  triager can't reproduce.
- **Show, don't assert.** "Returns 200 with user B's record" beats "it's
  vulnerable."

## Worked example (outline)

> **Title:** IDOR in `POST /api/reports/export` — `filename` allows arbitrary
> file write
> **Summary:** An authenticated user can set `export.filename` to an absolute or
> traversal path; the server writes the export there, overwriting arbitrary files
> the service can write.
> **Impact:** Arbitrary file write as the service account → RCE by dropping a
> cron/config file. Precondition: any authenticated account.
> **Steps:** 1. Log in as any user. 2. `POST /api/reports/export` with body
> `{"filename":"../../../../etc/cron.d/x", ...}`. 3. Observe the file written
> outside the export dir.
> **Proof:** [request/response], plus source path `handler.export_report →
> ReportJob → export_writer.write → open()` with the unchecked `filename`.
> **Root cause:** `filename` is used to build the output path with no
> normalization or base-containment check.
> **Remediation:** Resolve the path against the export base and reject if it
> escapes; or ignore client filenames and generate one.
> **Severity:** High (CVSS vector + reasoning).

## Rationalizations to reject

- *"They'll figure out the repro."* → They won't; they'll close it. You reproduce
  it from your own steps first.
- *"Bigger impact wording = higher payout."* → Inflation gets you dinged and
  distrusted. Rate it honestly; let the facts carry it.
- *"I'll include the whole capture/log."* → Minimize and redact. Noise buries the
  bug and can leak PII.
- *"One report, many bugs."* → One defect per report unless the program says
  otherwise; bundled reports get partially triaged and partially lost.

## Related

- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) — the source record every report is
  built from.
- `adjudicating-taint-paths`, `auditing-guard-gaps`, `detecting-memory-safety-bugs`,
  `detecting-race-conditions` — where confirmed findings come from.
