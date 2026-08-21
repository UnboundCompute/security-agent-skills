# Changelog

Notable changes to this skill library. Versions follow the plugin version in
`.claude-plugin/plugin.json`.

## 0.8.0

A breadth wave: one skill per underserved security topic, growing the library
from 51 to 61 skills across four new lanes. Firmware and embedded-software trust
brings the source-to-sink discipline to shipped device code, where the reader is
firmware source, init scripts, and default configuration rather than a web app.
Web trust boundaries packages the cross-origin controls that are widely
documented but rarely packaged as an adjudicating skill. Authentication and
identity depth niches out three failure classes that general federation reviews
skip. Abuse and isolation covers two systemic-property audits, one on limiting
and one on the tenant boundary, where the strongest signal is a peer path that
enforces where this one does not.

Firmware is reframed as embedded-software security, executable by a coding agent
over firmware source and configuration, not a physical-bench exercise. Each new
skill keeps the coverage-and-adjudication stance: leads are facts, the false-
positive killers ("check the other layer, the other build, the central policy")
are first-class, and every finding lands in the shared schema.

Added, firmware and embedded-software trust:

- `auditing-secure-boot-and-firmware-signing`
- `hunting-firmware-secrets-and-debug-interfaces`

Added, web trust boundaries:

- `auditing-webhook-authenticity-and-callback-trust`
- `auditing-cors-and-cross-origin-trust`
- `reviewing-content-security-policy`

Added, authentication and identity depth:

- `auditing-webauthn-and-passkey-flows`
- `auditing-device-code-and-pkce-flows`
- `auditing-randomness-and-nonce-quality`

Added, abuse and isolation:

- `reviewing-rate-limiting-and-abuse-controls`
- `auditing-multi-tenant-isolation`

Changed:

- README badge, hero, skills table, and roadmap updated for all four lanes.
- Plugin and marketplace manifests updated to version 0.8.0, with `firmware`,
  `embedded-security`, `secure-boot`, `webauthn`, `passkeys`, `oauth`, `cors`,
  `csp`, `webhooks`, `rate-limiting`, and `multi-tenancy` keywords.
- Social preview regenerated to 61 skills.

## 0.7.0

Added two lanes, growing the library from 45 to 51 skills: injection and
resource-exhaustion depth, and defensive detection and logging. The first fills
appsec ground that sits between the injection catalogs and nobody packaged
(single-request denial of service, unsafe archive extraction, and injection that
survives an object-relational mapper). The second extends the library to
blue-team and platform work without leaving its source-to-sink discipline:
whether an application records the security events an investigation needs,
whether detection rules survive an attacker who has read them, and whether an
event-driven function trusts its trigger.

Added, injection and resource-exhaustion depth:

- `hunting-redos-and-complexity-dos`
- `hunting-unsafe-archive-extraction`
- `hunting-orm-and-query-builder-injection`

Added, defensive detection and logging:

- `auditing-security-logging-completeness`
- `reviewing-detection-rules-for-evasion`
- `auditing-serverless-event-source-trust`

Changed:

- README badge, hero, skills table, and roadmap updated for both lanes.
- Plugin and marketplace manifests updated to version 0.7.0, with
  `denial-of-service`, `injection`, `detection-engineering`, `blue-team`, and
  `serverless` keywords.
- Social preview regenerated to 51 skills.

## 0.6.0

Added three lanes, growing the library from 36 to 45 skills: API and
object-authorization depth, host privilege escalation, and network-service
trust. The API lane hunts the most common serious API bugs (reaching an object
or writing a field the caller was never entitled to, and the surface a query API
exposes). The host and network lanes fill well-worn but under-packaged sysadmin
and netsec ground: local escalation through privileged binaries, jobs, and the
dynamic loader, and the trust a mail setup, a management service, or a
secure-shell session extends.

Added, API and object-authorization depth:

- `hunting-broken-object-level-authorization`
- `hunting-mass-assignment-and-property-authz`
- `auditing-graphql-attack-surface`

Added, host privilege escalation:

- `hunting-setuid-and-capability-escalation`
- `hunting-scheduled-job-and-search-path-hijacks`
- `hunting-dynamic-linker-hijacks`

Added, network-service trust:

- `testing-smtp-smuggling-and-email-spoofing`
- `enumerating-snmp-exposure`
- `auditing-ssh-trust-and-agent-forwarding`

Changed:

- README badge, hero, skills table, and roadmap updated for the three new lanes.
- Plugin and marketplace manifests updated to version 0.6.0, with
  `api-security`, `authorization`, `idor`, `sysadmin`, `privilege-escalation`,
  and `network-security` keywords.

## 0.5.0

Added a skill and MCP supply-chain trust lane, growing the library from 33 to 36
skills. These vet the skills and MCP servers you install into an agent, before
you trust them.

Added, skill and MCP supply-chain trust:

- `auditing-skill-and-mcp-instructions`
- `auditing-declared-vs-used-permissions`
- `vetting-skills-before-install`

Changed:

- README badge, hero, skills table, and roadmap updated for the new lane.
- Plugin and marketplace manifests updated to version 0.5.0.

## 0.4.0

Added a cloud identity and CI/CD trust lane, growing the library from 30 to 33
skills across six lanes.

Added, cloud identity and CI/CD trust:

- `hunting-iam-privilege-escalation-paths`
- `auditing-cicd-oidc-trust`
- `hunting-non-human-identity-and-secret-reachability`

Changed:

- README badge, hero, skills table, and roadmap updated for the cloud lane.
- Plugin and marketplace manifests updated to version 0.4.0 with a
  `cloud-security` keyword.

## 0.3.0

Grew the library from 13 to 30 skills and aligned the install identity to the
repository name.

Added, AI-agent and LLM red-teaming:

- `red-teaming-multi-agent-systems`
- `auditing-ai-agent-permissions`
- `testing-rag-and-memory-poisoning`
- `testing-llm-insecure-output-handling`
- `auditing-ml-model-supply-chain`
- `evaluating-model-guardrails`
- `reviewing-ai-generated-code`

Added, supply chain and business logic:

- `hunting-supply-chain-risks`
- `hunting-business-logic-flaws`

Added, appsec and access-control depth:

- `finding-crypto-misuse`
- `finding-fail-open-flaws`
- `auditing-declarative-authorization`
- `auditing-saml-and-oidc-flows`
- `testing-request-smuggling`
- `testing-client-side-dom-vulnerabilities`
- `exploiting-ssrf-to-cloud-metadata`
- `testing-web-cache-attacks`

Changed:

- Plugin and install identity renamed to `security-agent-skills` across the
  README, the plugin manifest, and the marketplace entry.
- README leads with a badge row and a one-line summary of scope and scale.

## 0.1.0

Initial release with 13 skills across five lanes.

- White-box: `hunting-bugs-with-a-code-graph`, `adjudicating-taint-paths`,
  `auditing-guard-gaps`, `detecting-memory-safety-bugs`,
  `detecting-race-conditions`.
- Research: `hunting-bug-variants`, `extracting-nday-from-a-patch`,
  `adjudicating-dependency-cve-reachability`.
- Agent: `auditing-the-lethal-trifecta`,
  `testing-agents-for-indirect-prompt-injection`,
  `auditing-mcp-tool-integrations`.
- Black-box: `mapping-attack-surface`.
- Reporting: `writing-vuln-reports`.
- Shared finding schema so results from any skill are consistent and
  deduplicable.
