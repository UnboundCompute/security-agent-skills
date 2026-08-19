# Changelog

Notable changes to this skill library. Versions follow the plugin version in
`.claude-plugin/plugin.json`.

## 0.6.0

Added an API and object-authorization depth lane, growing the library from 36 to
39 skills. These hunt the most common serious API bugs: reaching an object or
writing a field the caller was never entitled to, and the surface a query API
exposes.

Added, API and object-authorization depth:

- `hunting-broken-object-level-authorization`
- `hunting-mass-assignment-and-property-authz`
- `auditing-graphql-attack-surface`

Changed:

- README badge, hero, skills table, and roadmap updated for the new lane.
- Plugin and marketplace manifests updated to version 0.6.0, with
  `api-security`, `authorization`, and `idor` keywords.

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
