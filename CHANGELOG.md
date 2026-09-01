# Changelog

Notable changes to this skill library. Versions follow the plugin version in
`.claude-plugin/plugin.json`.

## 0.12.0

A wave across emerging trust surfaces, growing the library from 130 to 142
skills in one batch of twelve, spanning AI applications, web3, the browser, and
IoT and edge devices, each skill leading with the false-positive killer a
scanner skips.

The AI-application set covers the runtime of a served model. Code-interpreter
and tool sandbox escape treats model-generated code as hostile and checks
whether it reaches the network, the host filesystem, ambient credentials, or the
host itself; system-prompt and context leakage assumes the model will reveal what
it holds and checks that no secret and nothing cross-tenant is in the context;
inference-endpoint abuse treats a served model as both a cost meter and a
confidential asset, hunting denial-of-wallet, model extraction, and membership
inference.

The web3 set follows value through an on-chain stack. MEV and
transaction-ordering exposure finds sandwichable swaps and sequencing races that
a block producer controls; account abstraction and paymaster trust walks the
ERC-4337 user-operation lifecycle for over-permissive validation and drainable
sponsorship; cross-chain bridge and message trust checks that a destination
genuinely verifies the source event before it mints or releases; signature
replay and EIP-712 domain trust checks that a signature is bound to one action,
one chain, and one use; wallet-drainer and dApp approval abuse finds the
unlimited approvals, overbroad permits, and blind-signing that get a user to
authorize their own loss.

The browser set covers two front-end trust boundaries set by response headers:
third-party-script and SRI trust treats an external script tag as code you did
not write running with full page privileges, and clickjacking and UI redressing
finds sensitive one-click actions a framed overlay can steer. The IoT and edge
set covers a device's two remote surfaces: BLE and GATT authorization checks
that the peripheral, not the app, enforces access, and OTA and firmware-update
channel trust checks that a device verifies an image before it installs and runs
it forever.

## 0.11.0

A depth wave across the server-and-infrastructure trust surface, growing the
library from 82 to 130 skills in four batches of twelve, each batch a coherent
lane and each skill leading with the false-positive killer a scanner skips.

The first batch is server-side injection and deserialization. Blind and
second-order SQL injection covers the paths first-order testing walks past;
NoSQL operator and where injection catches a value arriving as an operator or a
server-side expression; LDAP injection and bind trust decides whether a filter
or a bind authenticates the wrong identity; expression-language and
search-engine injection find a string evaluated as a program or a query DSL;
connection-string and JDBC-URL injection catch an attacker-set connection
target; server-side prototype pollution catches one request changing every
object. The deserialization set covers Java gadget chains, Python pickle and
loader calls, PHP object injection and POP chains, and .NET type injection,
where reading an object runs code or picks the type. Datastore exposure and
abuse closes the batch on the unauthenticated cache or database reachable as a
command surface.

The second batch is cloud identity, secrets, and IaC trust: presigned-URL scope
abuse, cross-account role trust boundaries, service-account impersonation
chains, ECS task metadata boundaries, secrets-manager and KMS access-policy
trust, S3 object-ownership trust, observability-pipeline collector trust, IaC
module and provider supply chain, Terraform state and backend trust, Helm
template and values injection, and Ansible become and vault trust. Each answers
who can actually reach a credential, a secret, or a plan-time code path.

The third batch is Kubernetes, container, and host workload trust: container
escape surface, admission-control policy gaps, image provenance, pod-to-cloud
credential reach, network-policy segmentation, service-mesh mTLS and authz,
workload secret exposure, kubelet and node-API exposure, the namespace as a
tenant boundary, container runtime and socket exposure, init and sidecar
injection trust, and host mount and device exposure. The wedge throughout is
the difference between what a manifest declares and what the cluster and node
actually allow.

The fourth batch is network, identity, and payment trust: HTTP request
smuggling and desync, DNS rebinding and SSRF pivots, HTTP/2 and gRPC
multiplexing trust, message-broker topic authorization, OAuth audience and
scope trust, SAML and OIDC federation trust, JWT verification and key trust,
mutual-TLS service identity, account recovery and reset trust, payment state
machine and idempotency, payment callback and amount integrity, and price and
coupon manipulation. These trace a request from where two parsers disagree on
its boundary, through how a service proves who is calling, to how value moves
through checkout.

## 0.10.0

A niche-depth wave, growing the library from 73 to 82 skills across two clean new
lanes and three lane extensions, all chosen for high demand and thin packaged
presence. Client-app trust surfaces is the first new lane: the three places a
desktop or in-browser app trusts content it should not, a browser extension that
another page can reach through external messaging and content-script channels, an
editor extension that an untrusted repository can drive on open through auto-run
tasks and workspace-sourced tool paths, and an Electron app where renderer content
reaches a native capability. Each leads with the false-positive killer that the
scanners skip: the sender check, the workspace-trust state, and contextIsolation.
Wire-protocol and token trust is the second: gRPC service authorization finds the
method the interceptor does not cover (the unary-versus-streaming gap), WebSocket
connection trust catches trust checked once at the upgrade and never per message,
and JWT verification trust decides whether a signed token is verified on its own
terms, resolving algorithm pinning and key source before flagging.

Three extensions deepen existing lanes on the same adjudication wedge. CI/CD
workflow injection joins the cloud lane: attacker-controlled repository data
reaching a privileged run step, with the quoted-environment-variable fix as the
kill. File-upload and content handling joins the injection lane: the bug is a type
one layer trusts that another contradicts, with the no-execute origin and the
server-side re-encode as the killers. Move resource-ownership joins the
smart-contract lane, taking the resource-and-ability model of Aptos and Sui rather
than the EVM account model its siblings own.

Every new skill keeps the coverage-and-adjudication stance: leads are facts, the
false-positive killers are first-class and resolved before any flag, and every
finding lands in the shared schema.

Added, client-app trust surfaces:

- `auditing-browser-extension-trust`
- `auditing-editor-extension-workspace-trust`
- `auditing-electron-ipc-trust`

Added, wire-protocol and token trust:

- `auditing-grpc-service-authorization`
- `auditing-websocket-connection-trust`
- `auditing-jwt-verification-trust`

Added, cloud identity and CI/CD trust:

- `hunting-cicd-workflow-injection`

Added, injection and resource-exhaustion depth:

- `auditing-file-upload-and-content-handling`

Added, smart-contract and DeFi review:

- `auditing-move-resource-ownership`

Changed:

- README skill count 73 to 82, with the two new lanes in the hero, table, and
  roadmap, and the three extended lanes updated in place.
- `.claude-plugin/plugin.json` and `.claude-plugin/marketplace.json` bumped to
  0.10.0, with the two new lanes in the descriptions and new keywords (move,
  aptos, sui, browser-extension, electron, vscode, grpc, websocket, jwt, cicd,
  github-actions, file-upload).

## 0.9.0

A platform-coverage wave, growing the library from 61 to 73 skills across five
new lanes and reaching into three of the highest-demand review surfaces that the
earlier waves skipped. Transport and web trust closes the network-path gap:
subdomain takeover and dangling DNS is an infrastructure-state bug that
source-focused review never sees, and TLS and certificate validation catches the
client that accepts a certificate it should reject. Infrastructure-as-code and
container hardening moves the source-to-sink discipline onto declared cloud
resources, Kubernetes manifests, and image builds, where the durable edge over the
existing checklist scanners is the adjudication layer: the effective config after
variables and account defaults, the binding graph and admission verdict, and what
actually survives into a shipped image. Mobile app security scopes three
device-specific classes, cross-app component reach, deep-link and WebView trust,
and a real credential a real reader can reach, with the credential-versus-public-
identifier and reachability false-positive killers up front. Smart-contract and
DeFi review enters a crowded field on rigor, not novelty: a clean split between
reentrancy (reorder effects or add a mutex), access control (add or repair a
gate), and economic manipulation proven profitable on a fork.

Session lifecycle joins the authentication lane, and subdomain takeover joins web
trust. Every new skill keeps the coverage-and-adjudication stance: leads are
facts, the false-positive killers ("check the other layer, resolve the effective
state, prove it pays") are first-class, and every finding lands in the shared
schema.

Added, infrastructure-as-code and container hardening:

- `auditing-infrastructure-as-code-exposures`
- `auditing-kubernetes-workload-and-rbac-hardening`
- `auditing-container-image-build-hardening`

Added, mobile app security:

- `auditing-android-component-exposure`
- `auditing-mobile-deeplink-trust`
- `hunting-mobile-secret-and-storage-exposure`

Added, transport and web trust:

- `hunting-subdomain-takeover-and-dangling-dns`
- `auditing-tls-and-certificate-validation`

Added, authentication and session depth:

- `auditing-session-lifecycle-and-fixation`

Added, smart-contract and DeFi review:

- `hunting-smart-contract-reentrancy`
- `auditing-smart-contract-access-control`
- `hunting-defi-economic-and-oracle-flaws`

Changed:

- README hero, skill table, and roadmap updated for the 73-skill count and the
  new infrastructure-as-code, mobile, transport, and smart-contract lanes.
- Plugin and marketplace manifests bumped to 0.9.0 with expanded descriptions and
  new discovery keywords.

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
