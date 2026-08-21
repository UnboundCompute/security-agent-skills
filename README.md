# security-agent-skills

![license](https://img.shields.io/badge/license-MIT-blue?style=flat-square)
![skills](https://img.shields.io/badge/skills-61-2ea44f?style=flat-square)
![Claude Code plugin](https://img.shields.io/badge/Claude%20Code-plugin-8A63D2?style=flat-square)
![method](https://img.shields.io/badge/method-tool--agnostic-orange?style=flat-square)

**61 tool-agnostic security-testing skills for AI coding agents: white-box bug
hunting, AI-agent and LLM red-teaming, cloud identity and CI/CD trust, skill and
supply-chain trust, API and object-authorization depth, host and network-service
trust, firmware and embedded-software trust, web trust boundaries, authentication
and identity depth, rate-limiting and multi-tenant isolation, injection and
resource-exhaustion depth, defensive detection and logging, and appsec depth.**

Agent skills that encode security-testing *methodology*: the reasoning, ordering,
and adjudication discipline behind real black-box and white-box testing, written
as portable [Claude Code skills](https://docs.claude.com/en/docs/claude-code/skills).

The bet is that a skill's value is judgment, not a script. Enumerate the whole
taxonomy before you look at one family. Rank is triage, not a filter. A lead is a
fact, never a verdict. That reasoning transfers to anyone, whatever tools they
run, so every skill here is tool-agnostic method. Each one names the *capability*
a step needs (for example, "something that answers who calls this from a real
parse") without prescribing a product. Bring your own.

## Scope and ethics

These skills are for **authorized** security work only: your own code, OSS you
contribute to, CTFs, and engagements where testing is in scope. Every skill opens
with a scope check or gate. Nothing here targets systems you do not have
permission to test.

## Install

### As a plugin (recommended)

Installs all skills at once and keeps them updatable. In Claude Code:

```
/plugin marketplace add UnboundCompute/security-agent-skills
/plugin install security-agent-skills@unboundcompute
```

The skills are then available to Claude automatically, matched by their name and
description. Update later with `/plugin marketplace update unboundcompute`.

### One skill by hand

Copy any single skill directory into your skills path:

```
# project-local (checked in with your repo)
cp -r skills/detecting-race-conditions .claude/skills/

# or personal (available in every project)
cp -r skills/detecting-race-conditions ~/.claude/skills/
```

### Any other agent

The `SKILL.md` bodies are standalone playbooks. Point any agent runtime that
follows the [Agent Skills](https://agentskills.io) format at the `skills/`
directory, or read a `SKILL.md` and follow the loop by hand, running your own
tools at each step and emitting findings in the shared schema.

## The skills

| Skill | Lane | What it does |
|-------|------|--------------|
| [hunting-bugs-with-a-code-graph](skills/hunting-bugs-with-a-code-graph) | white-box | Master loop: orient, enumerate the whole taxonomy, adjudicate |
| [adjudicating-taint-paths](skills/adjudicating-taint-paths) | white-box | Turn a source-to-sink lead into a confirmed finding or a documented kill |
| [auditing-guard-gaps](skills/auditing-guard-gaps) | white-box | Find the unguarded peer of a guarded function |
| [detecting-memory-safety-bugs](skills/detecting-memory-safety-bugs) | white-box | UAF, double-free, OOB, uninitialized, NULL deref |
| [detecting-race-conditions](skills/detecting-race-conditions) | white-box | TOCTOU, check-then-act, atomicity, lock misuse |
| [hunting-business-logic-flaws](skills/hunting-business-logic-flaws) | white-box | Step-skip, limit-overrun, replay, and paywall bypass the scanners miss |
| [reviewing-ai-generated-code](skills/reviewing-ai-generated-code) | white-box | Hallucinated imports, insecure defaults, and claim-versus-behavior gaps in model-written code |
| [finding-crypto-misuse](skills/finding-crypto-misuse) | white-box | Nonce reuse, key recovery, padding oracles, length extension proven to a recovery |
| [finding-fail-open-flaws](skills/finding-fail-open-flaws) | white-box | Controls that allow on error, empty allowlist, missing input, or a caught denial |
| [auditing-declarative-authorization](skills/auditing-declarative-authorization) | white-box | Routes that skip the filter, permissive defaults, ownership not checked |
| [hunting-bug-variants](skills/hunting-bug-variants) | research | One confirmed bug to its siblings and the paths a fix missed |
| [extracting-nday-from-a-patch](skills/extracting-nday-from-a-patch) | research | A fix or version diff to the bug it fixed and the variants it left |
| [adjudicating-dependency-cve-reachability](skills/adjudicating-dependency-cve-reachability) | research | Is a dependency CVE actually reachable from your code, or noise |
| [hunting-supply-chain-risks](skills/hunting-supply-chain-risks) | research | Dependency confusion, slopsquatting, poisoned pipeline execution, CI privilege |
| [auditing-the-lethal-trifecta](skills/auditing-the-lethal-trifecta) | agent | Where private data, untrusted content, and exfiltration meet in one agent context |
| [testing-agents-for-indirect-prompt-injection](skills/testing-agents-for-indirect-prompt-injection) | agent | Does the agent obey instructions hidden in content it ingests |
| [auditing-mcp-tool-integrations](skills/auditing-mcp-tool-integrations) | agent | Tool poisoning, shadowing, rug-pulls, token passthrough, output injection |
| [red-teaming-multi-agent-systems](skills/red-teaming-multi-agent-systems) | agent | Agent-to-agent injection, delegation loops, confused deputy, distributed trifecta |
| [auditing-ai-agent-permissions](skills/auditing-ai-agent-permissions) | agent | Excessive agency, missing approval gates, egress, denial-of-wallet |
| [testing-rag-and-memory-poisoning](skills/testing-rag-and-memory-poisoning) | agent | Poisoned index, memory, and search results that fire on an innocent query |
| [testing-llm-insecure-output-handling](skills/testing-llm-insecure-output-handling) | agent | Model output to XSS, markdown-image exfil, terminal escapes, smuggled unicode |
| [auditing-ml-model-supply-chain](skills/auditing-ml-model-supply-chain) | agent | Deserialization RCE on model load, poisoned weights, hub name confusion |
| [evaluating-model-guardrails](skills/evaluating-model-guardrails) | agent | Policy-grounded, reproducible bypass-rate testing of safety guardrails |
| [auditing-skill-and-mcp-instructions](skills/auditing-skill-and-mcp-instructions) | supply-chain | Lint a skill or MCP server's instruction text for hidden, override, and exfil instructions |
| [auditing-declared-vs-used-permissions](skills/auditing-declared-vs-used-permissions) | supply-chain | The consent gap: permissions a skill declares versus what its code actually uses |
| [vetting-skills-before-install](skills/vetting-skills-before-install) | supply-chain | Vet a third-party skill or MCP server to an install, constrain, or deny verdict |
| [mapping-attack-surface](skills/mapping-attack-surface) | black-box | Authorized recon to a prioritized surface inventory |
| [testing-web-cache-attacks](skills/testing-web-cache-attacks) | black-box | Cache poisoning and cache deception through the key-versus-response gap |
| [auditing-saml-and-oidc-flows](skills/auditing-saml-and-oidc-flows) | black-box | Signature wrapping, audience and redirect_uri abuse, replay, identity confusion |
| [testing-request-smuggling](skills/testing-request-smuggling) | black-box | Front-end and back-end desync from conflicting length signals |
| [testing-client-side-dom-vulnerabilities](skills/testing-client-side-dom-vulnerabilities) | black-box | DOM XSS, clobbering, prototype pollution, unsafe messaging, cross-origin leaks |
| [exploiting-ssrf-to-cloud-metadata](skills/exploiting-ssrf-to-cloud-metadata) | black-box | SSRF proven to internal reach and instance-credential theft |
| [hunting-iam-privilege-escalation-paths](skills/hunting-iam-privilege-escalation-paths) | cloud | Chain role assumptions, policy rewrites, and loose trust from low-priv to admin |
| [auditing-cicd-oidc-trust](skills/auditing-cicd-oidc-trust) | cloud | Fork-run secret exposure, poisoned pipeline execution, over-broad token trust |
| [hunting-non-human-identity-and-secret-reachability](skills/hunting-non-human-identity-and-secret-reachability) | cloud | Machine credentials that are live, over-privileged, and actually reachable |
| [hunting-broken-object-level-authorization](skills/hunting-broken-object-level-authorization) | api | BOLA/IDOR: a client-supplied object reference reaching data with no owner binding |
| [hunting-mass-assignment-and-property-authz](skills/hunting-mass-assignment-and-property-authz) | api | Payload fields the client should never write: role, owner, price, verified |
| [auditing-graphql-attack-surface](skills/auditing-graphql-attack-surface) | api | Introspection, depth and cost, batching limits, per-field and per-mutation authz |
| [hunting-redos-and-complexity-dos](skills/hunting-redos-and-complexity-dos) | appsec | Single-request DoS: backtracking regex, quadratic loops, and hash flooding with no bound |
| [hunting-unsafe-archive-extraction](skills/hunting-unsafe-archive-extraction) | appsec | Archive path escape, symlink escape, decompression bombs, and post-extract execution |
| [hunting-orm-and-query-builder-injection](skills/hunting-orm-and-query-builder-injection) | appsec | Raw-query escape hatches, identifier and sort injection, filter-object operator injection |
| [hunting-setuid-and-capability-escalation](skills/hunting-setuid-and-capability-escalation) | host | Setuid/setgid and capability carriers paired with a reachable exec, read, write, or load primitive |
| [hunting-scheduled-job-and-search-path-hijacks](skills/hunting-scheduled-job-and-search-path-hijacks) | host | Writable job scripts, PATH hijack, and wildcard/argument injection (filenames as flags) |
| [hunting-dynamic-linker-hijacks](skills/hunting-dynamic-linker-hijacks) | host | Preload across a boundary, writable library paths, rpath and $ORIGIN into a writable dir |
| [testing-smtp-smuggling-and-email-spoofing](skills/testing-smtp-smuggling-and-email-spoofing) | network | SMTP smuggling boundary desync, SPF/DKIM/DMARC alignment and enforcement gaps, open relay |
| [enumerating-snmp-exposure](skills/enumerating-snmp-exposure) | network | Default community strings, downgrade, sensitive read views, and writable device state |
| [auditing-ssh-trust-and-agent-forwarding](skills/auditing-ssh-trust-and-agent-forwarding) | network | Forwarded-agent abuse, proxy-command injection, host-key trust gaps, loose key options |
| [auditing-security-logging-completeness](skills/auditing-security-logging-completeness) | defense | Whether security decisions are recorded, plus secrets-in-logs and log injection |
| [reviewing-detection-rules-for-evasion](skills/reviewing-detection-rules-for-evasion) | defense | Variant, anchor, and self-defeating-exclusion bypasses of detection-as-code rules |
| [auditing-serverless-event-source-trust](skills/auditing-serverless-event-source-trust) | defense | Event handlers that trust the payload or its source; event injection and blast radius |
| [auditing-secure-boot-and-firmware-signing](skills/auditing-secure-boot-and-firmware-signing) | firmware | Does a verified signature gate every path to flash and boot |
| [hunting-firmware-secrets-and-debug-interfaces](skills/hunting-firmware-secrets-and-debug-interfaces) | firmware | Embedded secrets, backdoor credentials, debug consoles, and unauthenticated command paths |
| [auditing-webhook-authenticity-and-callback-trust](skills/auditing-webhook-authenticity-and-callback-trust) | web-trust | Proving the webhook caller and pinning the callback target against SSRF |
| [auditing-cors-and-cross-origin-trust](skills/auditing-cors-and-cross-origin-trust) | web-trust | Reflected origins, null origin, substring allowlists, and postMessage origin checks |
| [reviewing-content-security-policy](skills/reviewing-content-security-policy) | web-trust | Whether a policy actually stops injected script or only looks strict |
| [auditing-webauthn-and-passkey-flows](skills/auditing-webauthn-and-passkey-flows) | auth | Passkey and WebAuthn ceremonies: challenge binding, origin, user-verification and counter checks |
| [auditing-device-code-and-pkce-flows](skills/auditing-device-code-and-pkce-flows) | auth | Proof-key and device-code grants: a code becoming a token without proof |
| [auditing-randomness-and-nonce-quality](skills/auditing-randomness-and-nonce-quality) | auth | Guessable-by-construction secrets: weak generators, short or reused nonces, low entropy |
| [reviewing-rate-limiting-and-abuse-controls](skills/reviewing-rate-limiting-and-abuse-controls) | abuse | Missing, spoofable-keyed, or per-instance limits on sensitive and expensive endpoints |
| [auditing-multi-tenant-isolation](skills/auditing-multi-tenant-isolation) | abuse | Whether every data operation is scoped to the caller's tenant |
| [writing-vuln-reports](skills/writing-vuln-reports) | reporting | Confirmed finding to a reproducible writeup |

Every finding, from any skill, is emitted in the shared
[finding schema](FINDING-SCHEMA.md), so results are consistent, deduplicable, and
ready for `writing-vuln-reports` without reformatting.

Each skill is a directory with a `SKILL.md` (YAML frontmatter plus body). The
white-box skills assume you have *some* way to answer structural questions from a
real parse: a code property graph, a static analyzer, or careful manual tracing
on a small target. The method does not depend on which.

## Roadmap

- [x] White-box: code-graph bug hunting, taint adjudication, guard-gap audit,
      memory safety, race conditions
- [x] Reporting: finding to reproducible writeup
- [x] Black-box: attack-surface recon and triage
- [x] Research: bug-variant hunting, n-day from a patch, dependency-CVE
      reachability
- [x] AI-agent red-teaming: the lethal trifecta, indirect prompt injection,
      MCP tool-integration abuse, multi-agent and delegation abuse, agent
      permissions and least-privilege, RAG and memory poisoning, insecure
      output handling
- [x] AI/ML security depth: untrusted model supply chain, guardrail evaluation,
      AI-generated-code review
- [x] Supply chain: dependency confusion, slopsquatting, poisoned pipeline
      execution, CI privilege abuse
- [x] Skill and MCP supply-chain trust: instruction-text linting, the
      permission consent-gap, and vetting a third-party skill or server to an
      install-or-deny verdict before you trust it
- [x] Business-logic depth: step-skip, limit-overrun, replay, and paywall bypass
- [x] Web depth: cache poisoning and cache deception
- [x] Access-control depth: fail-open controls, declarative authorization gaps
- [x] Appsec depth: crypto misuse, federation (SAML and OIDC) flows, request
      smuggling, client-side DOM attacks, SSRF to cloud metadata
- [x] Cloud identity and CI/CD trust: IAM privilege-escalation paths, pipeline
      OIDC-trust abuse, non-human identity and secret reachability
- [x] API and object-authorization depth: broken object-level authorization
      (BOLA/IDOR), mass assignment and object-property authorization, GraphQL
      attack surface
- [x] Host privilege escalation: setuid and capability carriers, scheduled-job
      and search-path hijacks (including wildcard/argument injection),
      dynamic-linker hijacks
- [x] Network-service trust: SMTP smuggling and email spoofing, SNMP exposure,
      SSH trust and agent forwarding
- [x] Injection and resource-exhaustion depth: ReDoS and algorithmic-complexity
      denial of service, unsafe archive extraction, ORM and query-builder
      injection
- [x] Defensive detection and logging: security-logging completeness,
      detection-rule evasion review, serverless event-source trust
- [x] Firmware and embedded-software trust: secure-boot and firmware-signing
      audit, embedded-secret and debug-interface hunting
- [x] Web trust boundaries: webhook authenticity and callback-SSRF, CORS and
      cross-origin trust, content-security-policy review
- [x] Authentication and identity depth: passkey and WebAuthn ceremonies,
      proof-key and device-code grants, randomness and nonce quality
- [x] Abuse and isolation: rate-limiting and abuse-control coverage,
      multi-tenant isolation

## Design

These are built on the patterns that make skills reliably trigger and get used:
gerund names, a `description` written as a routing rule (what plus when plus
trigger terms, third person), a lean body with progressive disclosure, concrete
worked examples, and a "rationalizations to reject" section per skill. See
[CONTRIBUTING.md](CONTRIBUTING.md) for the authoring checklist. The structure is
informed by Anthropic's Agent Skills guidance and the conventions of existing
open-source security skill libraries.

## Contributing

Contributions are welcome. New skills lead with tool-agnostic method, keep the
scope check, name capabilities rather than products, and emit the shared finding
schema. Read [CONTRIBUTING.md](CONTRIBUTING.md) before opening a PR, and file an
issue first if you want to discuss a new skill or a larger change.

## License

MIT. See [LICENSE](LICENSE).
