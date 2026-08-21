---
name: auditing-security-logging-completeness
description: >-
  Audit whether an application actually records the security events an investigation would need, and
  whether the logs themselves leak or lie: a security decision (authentication, an authorization
  denial or sensitive grant, a credential or privilege change, access to sensitive data) that fires
  with no durable record, an audit entry missing the actor, target, or outcome, secrets or personal
  data flowing into a widely-readable log, and untrusted input written to a log without neutralizing
  line breaks so entries can be forged. Covers coverage gaps at the decision points, record
  sufficiency and tamper-resistance, log-as-disclosure, and log injection. Use when assessing whether
  the code emits the security signal downstream detection and forensics depend on. The security
  action or the secret is the source, the audit or log sink is the sink, and the missing or unsafe
  record is the finding.
license: MIT
---

# Auditing security-logging completeness: whether the record exists when you need it

Detection and forensics can only work with signals the application actually emits. A gate that denies
access but writes nothing is invisible after the fact; a privilege change with no audit entry cannot be
reconstructed; a log that quietly captures a token or an email address becomes a disclosure store; a log
that accepts raw newlines can be forged to hide a trail. This audit is the mirror of the usual hunt: the
source is a security-relevant action that must reach an audit sink, or a secret that must never reach a
log sink, and the finding is a record that is missing, insufficient, leaking, or forgeable. You do it by
enumerating the decisions that matter and following each to whether, and how, it is recorded.

## When to use

- You are assessing whether an application produces the security events an investigation would need.
- Authorization denials, authentication, privilege changes, or sensitive reads may go unrecorded.
- Logs may capture secrets or personal data, or accept unneutralized untrusted input.

## Scope check

Audit logging and log contents only in systems you own or are authorized to assess, and treat any
secrets or personal data you find in logs as sensitive: report their presence, do not copy them out. If
you can't name the authorization, stop.

## The loop

1. **Enumerate the security decisions that should be recorded.** Inventory where a security-relevant
   event happens: authentication attempts and their outcome, authorization denials and sensitive grants,
   credential and privilege changes, account lifecycle events, and access to sensitive data. This is the
   set of sources that each need to reach a durable audit sink.

2. **Check each decision actually reaches an audit sink.** For every one, trace whether it emits a
   durable, security-relevant record, not just a response to the user. A denial that returns an error but
   logs nothing, or logs only at a debug level that is off in production, is invisible to an
   investigation. A decision that fires with no record is the core gap.

3. **Check the record is sufficient and tamper-evident.** A log line that omits the actor, the target, the
   source, or the outcome cannot answer the question it exists for. Determine whether audit events are
   distinguishable from operational noise, retained long enough, and protected from being altered or
   deleted by the same identities whose actions they record.

4. **Check for secrets and personal data flowing into logs.** Trace whether credentials, tokens, keys,
   session identifiers, or personal data reach a log sink, directly or by logging a whole request, header
   set, or object. A log is a lower-trust, widely-readable, long-retained store; a secret written there is
   a disclosure and often outlives the secret's rotation.

5. **Check for log injection.** Trace whether untrusted input is written into a log without neutralizing
   line breaks and control characters. An attacker who can inject a newline can forge additional entries,
   corrupt log parsing, or bury their own activity under noise, undermining every downstream detection and
   investigation that reads the log.

6. **Confirm and record.** Confirm a completeness gap by exercising a security decision in scope and
   showing no adequate record results; confirm a leak or injection by showing a secret reaching a log or
   an injected newline forging an entry. Kill the lead if every enumerated decision emits a durable,
   sufficient, tamper-resistant audit record, secrets and personal data are kept out of logs, and
   untrusted input is neutralized before it is written. Record with the decision or value, the sink, and
   what was missing or exposed.

## Where security logging leaks

- **A silent denial is a blind spot.** If the gate rejects but writes nothing, no detection rule and no
  investigation can ever see it. Coverage at the decision point is the whole point of the audit.
- **A record without the actor or outcome cannot be used.** Retention is wasted on entries that do not say
  who did what to whom with what result; sufficiency matters as much as existence.
- **Audit logs editable by the recorded identity are not evidence.** If the actor can alter or delete the
  record of their action, the log protects no one.
- **A log is a disclosure store.** Secrets and personal data written to logs spread widely, persist past
  rotation, and are read by many; the safe amount in a log is none.
- **Unneutralized input lets the attacker write the log.** Injected line breaks forge entries and blind
  the reader; neutralize control characters before writing untrusted data.

## Worked example (a confirm and a kill)

> **Confirm.** Authorization denials on a sensitive administrative action return an error to the caller
> but emit no audit event, and successful privilege grants are logged only at a debug level disabled in
> production. An attacker probing the action leaves no trace, and a granted escalation cannot be
> reconstructed afterward. **Confirmed** missing security-audit coverage on privileged actions, `high`,
> remediation = emit a durable audit event with actor, target, source, and outcome for every denial and
> grant on sensitive actions, at a level retained in production and protected from the acting identity.
>
> **Kill.** Every authentication, authorization denial and sensitive grant, credential and privilege
> change, and sensitive read emits a durable audit record carrying actor, target, source, time, and
> outcome, written to a store the recorded identities cannot alter; secrets and personal data are
> redacted before logging; and untrusted fields are neutralized against line-break injection. Exercised
> decisions all produce sufficient records and an injected newline is escaped. **Killed**, `kill_reason`
> = "all enumerated decisions emit sufficient, tamper-resistant audit records; no secrets or personal
> data in logs; untrusted input neutralized before writing."

## Rationalizations to reject

- *"We log plenty already."* → Volume is not coverage. The question is whether each specific security
  decision produces a sufficient record, not how many lines the app writes.
- *"The error response is the record."* → A response to the user is not a durable audit event and is not
  retained or protected. The record has to survive on your side.
- *"Debug logging covers it."* → Debug levels are off in production exactly when it matters. If it is
  security-relevant, it belongs at a retained level.
- *"It is just a token in the log, internal only."* → Logs are widely read and long retained; an internal
  secret in a log is a disclosure with a long tail. Redact it.
- *"Who would inject a log line?"* → Anyone who can put a newline in a field you log. Unneutralized input
  lets the attacker forge the very record you would investigate with.

## Executing this in practice

You need the list of security-decision points, the trace from each to whether and how it is recorded, the
sufficiency and protection of the audit store, the set of secret and personal-data values and whether any
reach a log sink, and whether untrusted input is neutralized before it is written. For each, ask whether
the record exists, whether it is enough, and whether the log can be leaked or forged. Reading the code
shows the intended logging; exercising the decisions and following sensitive values shows what is really
recorded.

## Related

- `finding-fail-open-flaws` - that skill hunts the gate that fails to deny; this one hunts the missing
  record of the gate firing at all.
- `reviewing-detection-rules-for-evasion` - detection can only match events the app emits; a coverage gap
  here is an upstream blind spot no rule can close.
- `writing-vuln-reports` - a completeness or leak finding needs the decision, the sink, and what was
  missing stated precisely; use the shared schema.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the security action or the secret, sink = the
  audit or log sink, evidence = the missing, insufficient, leaking, or forgeable record.
