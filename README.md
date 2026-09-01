# security-agent-skills

![license](https://img.shields.io/badge/license-MIT-blue?style=flat-square)
![skills](https://img.shields.io/badge/skills-142-2ea44f?style=flat-square)
![Claude Code plugin](https://img.shields.io/badge/Claude%20Code-plugin-8A63D2?style=flat-square)
![method](https://img.shields.io/badge/method-tool--agnostic-orange?style=flat-square)

**142 tool-agnostic security-testing skills for AI coding agents: white-box bug
hunting, AI-agent and LLM red-teaming, code-interpreter sandbox and
model-inference-endpoint abuse, server-side injection and deserialization
depth across SQL, NoSQL, LDAP, expression-language, and object streams, cloud
identity, secrets, and KMS trust, infrastructure-as-code and container/Kubernetes
and host workload trust, network, wire-protocol, and multiplexing trust across
HTTP/2, gRPC, message brokers, and mutual TLS, federation and token trust across
SAML, OIDC, OAuth, and JWT, account-recovery and payment-and-commerce integrity,
client-app trust surfaces across browser and editor extensions and Electron,
mobile app security, smart-contract and DeFi review across EVM and Move plus web3
account-abstraction, cross-chain bridge, MEV, signature-replay, and wallet-drainer
trust, browser third-party-script and clickjacking trust, IoT and edge BLE and
firmware-update trust, skill and supply-chain trust, API and object-authorization
depth, firmware and embedded-software trust, web and transport trust boundaries,
authentication and session depth, rate-limiting and multi-tenant isolation,
file-upload and resource-exhaustion depth, defensive detection and logging, and
appsec depth.**

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
| [hunting-cicd-workflow-injection](skills/hunting-cicd-workflow-injection) | cloud | Untrusted event data into a run step, privileged pull_request_target checkout, mutable-tag actions, cache and runner trust |
| [hunting-non-human-identity-and-secret-reachability](skills/hunting-non-human-identity-and-secret-reachability) | cloud | Machine credentials that are live, over-privileged, and actually reachable |
| [auditing-infrastructure-as-code-exposures](skills/auditing-infrastructure-as-code-exposures) | iac | Effective resource state after variables and defaults: public storage, open ingress, wildcard policies, plaintext secrets |
| [auditing-kubernetes-workload-and-rbac-hardening](skills/auditing-kubernetes-workload-and-rbac-hardening) | iac | What the binding graph and admission actually allow: cluster-admin grants, escape-primitive workloads |
| [auditing-container-image-build-hardening](skills/auditing-container-image-build-hardening) | iac | What survives into the shipped image: baked secrets, root user, unpinned bases, dangerous runtime requests |
| [hunting-broken-object-level-authorization](skills/hunting-broken-object-level-authorization) | api | BOLA/IDOR: a client-supplied object reference reaching data with no owner binding |
| [hunting-mass-assignment-and-property-authz](skills/hunting-mass-assignment-and-property-authz) | api | Payload fields the client should never write: role, owner, price, verified |
| [auditing-graphql-attack-surface](skills/auditing-graphql-attack-surface) | api | Introspection, depth and cost, batching limits, per-field and per-mutation authz |
| [hunting-redos-and-complexity-dos](skills/hunting-redos-and-complexity-dos) | appsec | Single-request DoS: backtracking regex, quadratic loops, and hash flooding with no bound |
| [hunting-unsafe-archive-extraction](skills/hunting-unsafe-archive-extraction) | appsec | Archive path escape, symlink escape, decompression bombs, and post-extract execution |
| [hunting-orm-and-query-builder-injection](skills/hunting-orm-and-query-builder-injection) | appsec | Raw-query escape hatches, identifier and sort injection, filter-object operator injection |
| [auditing-file-upload-and-content-handling](skills/auditing-file-upload-and-content-handling) | appsec | When the type one layer trusts another contradicts: active-content serving, parser exploitation, polyglots, path escape |
| [auditing-android-component-exposure](skills/auditing-android-component-exposure) | mobile | What another app can reach and drive: exported components, weak permission gates, trusted intent extras |
| [auditing-mobile-deeplink-trust](skills/auditing-mobile-deeplink-trust) | mobile | When an attacker URL drives a trusted action: scheme hijack, unvalidated params, WebView and JS-bridge trust |
| [hunting-mobile-secret-and-storage-exposure](skills/hunting-mobile-secret-and-storage-exposure) | mobile | A real credential a real reader can reach: embedded secrets, unsafe storage, credential-versus-identifier |
| [auditing-browser-extension-trust](skills/auditing-browser-extension-trust) | client-app | What another page can reach through the extension: external messaging, content-script channels, host permissions, DOM sinks |
| [auditing-editor-extension-workspace-trust](skills/auditing-editor-extension-workspace-trust) | client-app | What an untrusted repository makes the editor do on open: auto-run tasks, debug launches, workspace-sourced tool paths |
| [auditing-electron-ipc-trust](skills/auditing-electron-ipc-trust) | client-app | When renderer content reaches a native capability: contextIsolation, nodeIntegration, the preload bridge, IPC handlers |
| [hunting-setuid-and-capability-escalation](skills/hunting-setuid-and-capability-escalation) | host | Setuid/setgid and capability carriers paired with a reachable exec, read, write, or load primitive |
| [hunting-scheduled-job-and-search-path-hijacks](skills/hunting-scheduled-job-and-search-path-hijacks) | host | Writable job scripts, PATH hijack, and wildcard/argument injection (filenames as flags) |
| [hunting-dynamic-linker-hijacks](skills/hunting-dynamic-linker-hijacks) | host | Preload across a boundary, writable library paths, rpath and $ORIGIN into a writable dir |
| [testing-smtp-smuggling-and-email-spoofing](skills/testing-smtp-smuggling-and-email-spoofing) | network | SMTP smuggling boundary desync, SPF/DKIM/DMARC alignment and enforcement gaps, open relay |
| [enumerating-snmp-exposure](skills/enumerating-snmp-exposure) | network | Default community strings, downgrade, sensitive read views, and writable device state |
| [auditing-ssh-trust-and-agent-forwarding](skills/auditing-ssh-trust-and-agent-forwarding) | network | Forwarded-agent abuse, proxy-command injection, host-key trust gaps, loose key options |
| [auditing-tls-and-certificate-validation](skills/auditing-tls-and-certificate-validation) | network | When the client accepts a certificate it should reject: disabled verification, trust-all managers, downgrade |
| [auditing-grpc-service-authorization](skills/auditing-grpc-service-authorization) | wire-trust | The method the interceptor does not cover: unary-versus-streaming gaps, per-method authz, reflection, transcoding gateways |
| [auditing-websocket-connection-trust](skills/auditing-websocket-connection-trust) | wire-trust | Trust checked once at the upgrade, never per message: origin gates, per-message authz, message-as-transport sinks |
| [auditing-jwt-verification-trust](skills/auditing-jwt-verification-trust) | wire-trust | When a signed token is verified on its own terms: header-chosen algorithm, key confusion, kid/jku injection, unchecked claims |
| [auditing-security-logging-completeness](skills/auditing-security-logging-completeness) | defense | Whether security decisions are recorded, plus secrets-in-logs and log injection |
| [reviewing-detection-rules-for-evasion](skills/reviewing-detection-rules-for-evasion) | defense | Variant, anchor, and self-defeating-exclusion bypasses of detection-as-code rules |
| [auditing-serverless-event-source-trust](skills/auditing-serverless-event-source-trust) | defense | Event handlers that trust the payload or its source; event injection and blast radius |
| [auditing-secure-boot-and-firmware-signing](skills/auditing-secure-boot-and-firmware-signing) | firmware | Does a verified signature gate every path to flash and boot |
| [hunting-firmware-secrets-and-debug-interfaces](skills/hunting-firmware-secrets-and-debug-interfaces) | firmware | Embedded secrets, backdoor credentials, debug consoles, and unauthenticated command paths |
| [auditing-webhook-authenticity-and-callback-trust](skills/auditing-webhook-authenticity-and-callback-trust) | web-trust | Proving the webhook caller and pinning the callback target against SSRF |
| [auditing-cors-and-cross-origin-trust](skills/auditing-cors-and-cross-origin-trust) | web-trust | Reflected origins, null origin, substring allowlists, and postMessage origin checks |
| [reviewing-content-security-policy](skills/reviewing-content-security-policy) | web-trust | Whether a policy actually stops injected script or only looks strict |
| [hunting-subdomain-takeover-and-dangling-dns](skills/hunting-subdomain-takeover-and-dangling-dns) | web-trust | Who can claim the name you still point at: dangling records, delegation takeover, teardown ordering |
| [auditing-webauthn-and-passkey-flows](skills/auditing-webauthn-and-passkey-flows) | auth | Passkey and WebAuthn ceremonies: challenge binding, origin, user-verification and counter checks |
| [auditing-device-code-and-pkce-flows](skills/auditing-device-code-and-pkce-flows) | auth | Proof-key and device-code grants: a code becoming a token without proof |
| [auditing-randomness-and-nonce-quality](skills/auditing-randomness-and-nonce-quality) | auth | Guessable-by-construction secrets: weak generators, short or reused nonces, low entropy |
| [auditing-session-lifecycle-and-fixation](skills/auditing-session-lifecycle-and-fixation) | auth | Whether a session is rotated, scoped, and destroyed: fixation, post-logout replay, endless lifetime |
| [reviewing-rate-limiting-and-abuse-controls](skills/reviewing-rate-limiting-and-abuse-controls) | abuse | Missing, spoofable-keyed, or per-instance limits on sensitive and expensive endpoints |
| [auditing-multi-tenant-isolation](skills/auditing-multi-tenant-isolation) | abuse | Whether every data operation is scoped to the caller's tenant |
| [hunting-smart-contract-reentrancy](skills/hunting-smart-contract-reentrancy) | web3 | Acting on state not yet updated: interaction-before-effects, cross-function and read-only reentrancy |
| [auditing-smart-contract-access-control](skills/auditing-smart-contract-access-control) | web3 | Who may call the privileged function: missing modifiers, unprotected initializers, attacker delegatecall |
| [hunting-defi-economic-and-oracle-flaws](skills/hunting-defi-economic-and-oracle-flaws) | web3 | Profiting by moving a price nobody bounded: spot-oracle and flash-loan manipulation, rounding, share math |
| [auditing-move-resource-ownership](skills/auditing-move-resource-ownership) | web3 | Move (Aptos and Sui): acting on a resource without proving ownership, capability leaks, ability misuse |
| [hunting-blind-and-second-order-sql-injection](skills/hunting-blind-and-second-order-sql-injection) | injection | Boolean, time, and error blind paths, plus stored input that executes on a later read |
| [hunting-nosql-operator-and-where-injection](skills/hunting-nosql-operator-and-where-injection) | injection | A value arriving as a query operator or a server-side where/JS expression |
| [hunting-ldap-injection-and-bind-trust](skills/hunting-ldap-injection-and-bind-trust) | injection | Filter injection and bind logic that lets input decide authentication |
| [hunting-expression-language-injection](skills/hunting-expression-language-injection) | injection | Template and expression contexts that evaluate a string as a program |
| [hunting-search-engine-injection](skills/hunting-search-engine-injection) | injection | Query-DSL and script-query injection into a search backend |
| [hunting-connection-string-and-jdbc-url-injection](skills/hunting-connection-string-and-jdbc-url-injection) | injection | Attacker-set connection targets and driver properties that load or exfiltrate |
| [hunting-server-side-prototype-pollution](skills/hunting-server-side-prototype-pollution) | injection | One request mutating a base prototype to change every object's behavior |
| [hunting-java-deserialization-gadget-chains](skills/hunting-java-deserialization-gadget-chains) | deserialization | Untrusted object streams reaching a gadget chain that runs code on read |
| [hunting-python-unsafe-deserialization](skills/hunting-python-unsafe-deserialization) | deserialization | pickle, YAML, and loaders that call a function while loading data |
| [hunting-php-object-injection-pop-chains](skills/hunting-php-object-injection-pop-chains) | deserialization | unserialize reaching a property-oriented chain through magic methods |
| [hunting-dotnet-deserialization-type-injection](skills/hunting-dotnet-deserialization-type-injection) | deserialization | Type-name-carrying formatters that instantiate an attacker-chosen type |
| [auditing-datastore-exposure-and-abuse](skills/auditing-datastore-exposure-and-abuse) | datastore | Unauthenticated caches and databases reachable as a command surface |
| [auditing-presigned-url-scope-abuse](skills/auditing-presigned-url-scope-abuse) | cloud | Signed URLs granting more path, method, or lifetime than the request intended |
| [auditing-cross-account-role-trust-boundaries](skills/auditing-cross-account-role-trust-boundaries) | cloud | Trust policies that admit the wrong external principal without a condition |
| [mapping-service-account-impersonation-chains](skills/mapping-service-account-impersonation-chains) | cloud | Token, impersonation, and key grants that let one identity become another |
| [auditing-ecs-task-metadata-boundaries](skills/auditing-ecs-task-metadata-boundaries) | cloud | Containers reaching task or instance credentials they should not |
| [reviewing-secrets-manager-access-policy-trust](skills/reviewing-secrets-manager-access-policy-trust) | cloud | Who can actually read a secret across resource, identity, and key policy |
| [auditing-kms-key-policy-and-envelope-encryption](skills/auditing-kms-key-policy-and-envelope-encryption) | cloud | Who can decrypt: key-policy grants, grants, and envelope-key reach |
| [auditing-observability-pipeline-collector-trust](skills/auditing-observability-pipeline-collector-trust) | cloud | Telemetry collectors trusting the payload or its source as input |
| [auditing-s3-object-ownership-trust](skills/auditing-s3-object-ownership-trust) | cloud | When the object, not the bucket, decides who can read or overwrite |
| [auditing-iac-module-and-provider-supply-chain](skills/auditing-iac-module-and-provider-supply-chain) | iac | Third-party modules and providers running code at plan and apply |
| [auditing-terraform-state-and-backend-trust](skills/auditing-terraform-state-and-backend-trust) | iac | State files and backends that expose secrets or accept tampering |
| [hunting-helm-template-and-values-injection](skills/hunting-helm-template-and-values-injection) | iac | Chart values that render into manifests as privilege or injection |
| [auditing-ansible-become-and-vault-trust](skills/auditing-ansible-become-and-vault-trust) | iac | Privilege escalation, vault handling, and control-node trust in playbooks |
| [hunting-container-escape-surface](skills/hunting-container-escape-surface) | k8s | Privileged, capability, and mount configurations that reach the host |
| [auditing-admission-control-policy-gaps](skills/auditing-admission-control-policy-gaps) | k8s | Admission policies that fail open, miss resources, or can be bypassed |
| [mapping-pod-to-cloud-credential-reach](skills/mapping-pod-to-cloud-credential-reach) | k8s | The cloud blast radius of one compromised pod's identity |
| [auditing-network-policy-segmentation-gaps](skills/auditing-network-policy-segmentation-gaps) | k8s | Default-open pod networking and policies that do not actually segment |
| [auditing-service-mesh-mtls-and-authz-trust](skills/auditing-service-mesh-mtls-and-authz-trust) | k8s | Mesh mTLS and authorization asserting a trust it does not enforce |
| [auditing-workload-secret-exposure-surface](skills/auditing-workload-secret-exposure-surface) | k8s | Every reader of a workload secret: env, volume, RBAC, and node |
| [hunting-kubelet-and-node-api-exposure](skills/hunting-kubelet-and-node-api-exposure) | k8s | Kubelet and node APIs exposing exec, logs, and workload control |
| [auditing-namespace-as-tenant-boundary](skills/auditing-namespace-as-tenant-boundary) | k8s | What a namespace does not isolate when treated as a tenant boundary |
| [auditing-init-and-sidecar-injection-trust](skills/auditing-init-and-sidecar-injection-trust) | k8s | Injected init and sidecar containers sharing a pod's trust |
| [auditing-container-image-provenance](skills/auditing-container-image-provenance) | container | Mutable tags, unverified signatures, and image-to-artifact drift |
| [auditing-container-runtime-and-socket-exposure](skills/auditing-container-runtime-and-socket-exposure) | container | A runtime socket or API handle that is a handle on the host |
| [auditing-host-mount-and-device-exposure](skills/auditing-host-mount-and-device-exposure) | container | Host path and device mounts opening a route into the node filesystem |
| [hunting-http-request-smuggling-and-desync](skills/hunting-http-request-smuggling-and-desync) | network | Front-end and back-end parsers disagreeing on where a request ends |
| [hunting-dns-rebinding-and-ssrf-pivots](skills/hunting-dns-rebinding-and-ssrf-pivots) | network | SSRF, DNS rebinding, and redirect pivots turning the server into a proxy |
| [auditing-http2-and-grpc-multiplexing-trust](skills/auditing-http2-and-grpc-multiplexing-trust) | wire-trust | h2c downgrade, pseudo-header forgery, and per-stream versus per-connection trust |
| [auditing-message-broker-topic-authorization](skills/auditing-message-broker-topic-authorization) | wire-trust | Topic ACLs, wildcard subscribes, and cross-tenant reach on MQTT, Kafka, and AMQP |
| [hunting-mutual-tls-and-service-identity-gaps](skills/hunting-mutual-tls-and-service-identity-gaps) | wire-trust | Certificates requested not required, chain validity mistaken for identity |
| [auditing-oauth-token-audience-and-scope-trust](skills/auditing-oauth-token-audience-and-scope-trust) | auth | Audience, issuer, and scope confusion accepting a token outside its domain |
| [auditing-saml-and-oidc-federation-trust](skills/auditing-saml-and-oidc-federation-trust) | auth | Signature wrapping, issuer/audience/nonce, and unbound subject in federation |
| [auditing-jwt-verification-and-key-trust](skills/auditing-jwt-verification-and-key-trust) | auth | Algorithm confusion, attacker-chosen keys, decode-without-verify, unenforced claims |
| [auditing-account-recovery-and-reset-trust](skills/auditing-account-recovery-and-reset-trust) | auth | Weak reset tokens, multi-factor bypass, and host-poisoned reset links |
| [auditing-payment-state-machine-and-idempotency](skills/auditing-payment-state-machine-and-idempotency) | payment | Out-of-order and replayed transitions releasing value before settlement |
| [auditing-payment-callback-and-amount-integrity](skills/auditing-payment-callback-and-amount-integrity) | payment | Forged callbacks and unreconciled amounts marking an order paid |
| [hunting-price-and-coupon-manipulation](skills/hunting-price-and-coupon-manipulation) | payment | Client-set prices, negative quantities, and coupon stacking that lower the total |
| [hunting-code-interpreter-and-tool-sandbox-escape](skills/hunting-code-interpreter-and-tool-sandbox-escape) | ai-app | Model-generated code reaching network, host FS, credentials, or the host |
| [auditing-system-prompt-and-context-leakage](skills/auditing-system-prompt-and-context-leakage) | ai-app | Secrets in the prompt, unscoped retrieval, and session or tenant memory bleed |
| [auditing-ml-inference-endpoint-abuse](skills/auditing-ml-inference-endpoint-abuse) | ai-app | Denial-of-wallet, model extraction, and membership inference on a served model |
| [hunting-mev-and-transaction-ordering-exposure](skills/hunting-mev-and-transaction-ordering-exposure) | web3 | Sandwichable swaps, sequencing races, and mempool intent leakage |
| [auditing-account-abstraction-and-paymaster-trust](skills/auditing-account-abstraction-and-paymaster-trust) | web3 | ERC-4337 validation gaps and drainable paymaster sponsorship |
| [auditing-cross-chain-bridge-and-message-trust](skills/auditing-cross-chain-bridge-and-message-trust) | web3 | Forged proofs, spoofable quora, replay, and unbalanced lock/mint accounting |
| [hunting-signature-replay-and-eip712-domain-trust](skills/hunting-signature-replay-and-eip712-domain-trust) | web3 | Missing nonce or domain binding letting a signature replay or cross contexts |
| [hunting-wallet-drainer-and-dapp-approval-abuse](skills/hunting-wallet-drainer-and-dapp-approval-abuse) | web3 | Unlimited approvals, overbroad permits, and blind-signing that drain a wallet |
| [auditing-third-party-script-and-sri-trust](skills/auditing-third-party-script-and-sri-trust) | browser | Unpinned external script and permissive CSP enabling in-page skimming |
| [auditing-clickjacking-and-ui-redressing](skills/auditing-clickjacking-and-ui-redressing) | browser | Framing and overlay redressing a sensitive one-click action |
| [auditing-ble-and-gatt-authorization](skills/auditing-ble-and-gatt-authorization) | iot-edge | GATT characteristics without device-side authorization, pairing, or replay protection |
| [auditing-ota-and-firmware-update-channel-trust](skills/auditing-ota-and-firmware-update-channel-trust) | iot-edge | Unsigned, swapped, or rolled-back firmware installing on a device |
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
      OIDC-trust abuse, CI/CD workflow injection (untrusted data into a
      privileged run step), non-human identity and secret reachability
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
      injection, file-upload and content handling
- [x] Defensive detection and logging: security-logging completeness,
      detection-rule evasion review, serverless event-source trust
- [x] Firmware and embedded-software trust: secure-boot and firmware-signing
      audit, embedded-secret and debug-interface hunting
- [x] Web trust boundaries: webhook authenticity and callback-SSRF, CORS and
      cross-origin trust, content-security-policy review, subdomain takeover
      and dangling DNS
- [x] Authentication and session depth: passkey and WebAuthn ceremonies,
      proof-key and device-code grants, randomness and nonce quality, session
      lifecycle and fixation
- [x] Abuse and isolation: rate-limiting and abuse-control coverage,
      multi-tenant isolation
- [x] Infrastructure-as-code and container hardening: cloud resource
      exposures, Kubernetes workload and RBAC hardening, container image build
      hardening
- [x] Transport trust: TLS and certificate validation
- [x] Mobile app security: Android component exposure, mobile deep-link trust,
      mobile secret and storage exposure
- [x] Smart-contract and DeFi review: reentrancy, access control, economic and
      oracle manipulation, Move resource-ownership and capability safety (Aptos
      and Sui)
- [x] Client-app trust surfaces: browser-extension message and permission trust,
      editor-extension workspace trust, Electron renderer-to-native IPC trust
- [x] Wire-protocol and token trust: gRPC service authorization, WebSocket
      connection trust, JWT verification trust
- [x] Server-side injection and deserialization depth: blind and second-order
      SQL, NoSQL operator and where injection, LDAP injection and bind trust,
      expression-language injection, search-engine injection, connection-string
      and JDBC-URL injection, server-side prototype pollution, and Java,
      Python, PHP, and .NET deserialization gadget and object-injection chains,
      plus unauthenticated datastore exposure
- [x] Cloud identity, secrets, and IaC trust: presigned-URL scope, cross-account
      role trust boundaries, service-account impersonation chains, ECS task
      metadata boundaries, secrets-manager and KMS access trust, S3
      object-ownership trust, observability-collector trust, IaC module and
      provider supply chain, Terraform state and backend trust, Helm
      template/values injection, and Ansible become and vault trust
- [x] Kubernetes, container, and host workload trust: container escape surface,
      admission-control policy gaps, image provenance, pod-to-cloud credential
      reach, network-policy segmentation, service-mesh mTLS and authz, workload
      secret exposure, kubelet and node-API exposure, namespace-as-tenant
      boundary, runtime and socket exposure, init/sidecar injection trust, and
      host mount and device exposure
- [x] Network, multiplexing, and service-identity trust: HTTP request smuggling
      and desync, DNS rebinding and SSRF pivots, HTTP/2 and gRPC multiplexing
      trust, message-broker topic authorization, and mutual-TLS service identity
- [x] Federation, token, and recovery trust: OAuth audience and scope, SAML and
      OIDC federation, JWT verification and key trust, and account-recovery and
      reset trust
- [x] Payment and commerce integrity: payment state machine and idempotency,
      payment callback and amount integrity, and price and coupon manipulation
- [x] Emerging surfaces: AI-application code-interpreter sandbox escape,
      system-prompt and context leakage, and inference-endpoint abuse; web3
      MEV, account abstraction, cross-chain bridge, signature replay, and
      wallet-drainer trust; browser third-party-script and clickjacking trust;
      and IoT/edge BLE and firmware-update trust

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
