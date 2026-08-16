# Finding schema

Every skill in this library emits findings in one shape, so results are
machine-readable, deduplicable, and ready to hand to the report skill without
reformatting. A finding is a decided fact - either `confirmed` or `killed`, never
"maybe." Leads that haven't been adjudicated are not findings.

## Fields

| Field | Required | Meaning |
|-------|----------|---------|
| `id` | yes | Stable slug, e.g. `wb-2026-idor-bulk-delete`. |
| `title` | yes | One line, the specific defect - not the class. |
| `class` | yes | Vuln class, e.g. `path-traversal`, `use-after-free`, `race-condition`, `missing-authz`. |
| `status` | yes | `confirmed` or `killed`. Killed findings are kept - they record what was ruled out and why. |
| `severity` | confirmed only | `info` / `low` / `medium` / `high` / `critical`. Justify in `impact`. |
| `source` | yes | The untrusted entry point (request param, header, filename, env var, deserialized field…). Be exact: which input. |
| `sink` | yes | The dangerous operation and the *exact argument* that is dangerous. |
| `path` | yes | The witness: `source → … → sink`, hop by hop, as function/decl references. |
| `evidence` | yes | Source excerpts at each decisive hop, at the commit you adjudicated against. Quote the code. |
| `reachable` | yes | `true` / `false` / `conditional`. If conditional, state the condition. |
| `guard_status` | guard/authz findings | `absent` / `bypassable` / `weaker-than-peer` / `after-sink`, vs. the anchor that gets it right. |
| `confidence` | yes | `confirmed-by-source` / `needs-runtime-poc`. Static confirmation vs. a claim awaiting a live PoC. |
| `impact` | confirmed only | What an attacker gains when the value lands. Justifies severity. |
| `kill_reason` | killed only | The exact hop where taint breaks / the guard that covers it / why unreachable. |
| `remediation` | confirmed only | The fix, at the decisive hop. |
| `commit` | yes | The revision adjudicated against. A finding is only valid against a stated commit. |

## YAML example (confirmed)

```yaml
id: wb-2026-pathtrav-export
title: "Report export writes to attacker-controlled absolute path"
class: path-traversal
status: confirmed
severity: high
source: "HTTP body field `export.filename` on POST /api/reports/export"
sink: "open(path, 'w') - the `path` argument, export_writer.py"
path:
  - "handler.export_report  (reads body.filename)"
  - "ReportJob.__init__     (stores as self.name, no normalization)"
  - "export_writer.write    (path = base / self.name)"
  - "open(path, 'w')        (sink)"
evidence:
  - "handler.export_report: name = body['filename']  # unchecked"
  - "export_writer.write: path = self.base / self.name  # `/etc/x` escapes base"
reachable: true
confidence: confirmed-by-source
impact: "Authenticated user overwrites arbitrary files the service can write; RCE via cron/config drop."
remediation: "Resolve against base and reject if the resolved path is not under base; or use a generated name."
commit: 1f45798
```

## YAML example (killed)

```yaml
id: wb-2026-sqli-search
title: "Search endpoint SQL built from user query"
class: sql-injection
status: killed
source: "GET /api/search?q="
sink: "cursor.execute(sql) - search_repo.py"
path: ["handler.search → SearchRepo.run → cursor.execute"]
reachable: true
kill_reason: "SearchRepo.run passes q as a bound parameter (execute(sql, (q,))); q never concatenated into sql. Confirmed at every hop."
confidence: confirmed-by-source
commit: 1f45798
```

Keep killed findings. Re-opening a ruled-out lead next pass, because nobody wrote
down why it died, is pure wasted work.
