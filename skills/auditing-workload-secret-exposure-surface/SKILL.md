---
name: auditing-workload-secret-exposure-surface
description: >-
  Audit how a workload holds its secrets for the exposure that outlives the secret's intent: a secret passed
  as an environment variable that any process, crash dump, or child inherits and that debug endpoints echo, a
  secret volume mounted where a sidecar or a shared process can read it, a secret written into logs or an
  error, and a Kubernetes secret readable by more service accounts than the one workload that needs it.
  Covers containerized workloads where secrets reach the process through environment, mounted files, or the
  orchestrator's secret store. Use when a workload consumes secrets and the question is who or what else can
  read them. The workload secret is the source, the process, sidecar, log, or extra reader that can see it is
  the sink, and the exposure beyond the intended consumer is the bug.
license: MIT
---

# Auditing workload secret exposure surface: who else can read this secret

A secret a workload needs is safe only if nothing beyond the intended consumer can read it, and containerized
delivery quietly widens that set. An environment variable is inherited by every child process, echoed by many
debug and introspection endpoints, and captured in crash dumps and process listings, so an env secret is
readable by far more than the code that meant to use it. A mounted secret file is readable by every container
in the pod that shares the mount, so a sidecar becomes a reader. A secret that lands in a log line or an error
message is exposed to whoever reads logs, which is usually a much broader audience than whoever holds the
secret. And the orchestrator's secret object is readable by every identity its access rules admit, often more
service accounts than the one workload. The audit is not whether the secret exists but who else can see it.
You audit this by tracing each secret from its store into the workload and enumerating every other reader.

## When to use

- Workloads consume secrets through environment variables, mounted files, or the orchestrator's secret store.
- Sidecars, child processes, debug endpoints, or logs may be able to read a secret meant for one consumer.
- A secret object may be readable by more identities or service accounts than the single workload that needs it.

## Scope check

Audit secret exposure only for workloads and clusters you own or are authorized to assess, on non-production
secrets. Confirming exposure means reading a real secret through the exposing path, so use non-production
credentials and never exfiltrate or reuse a secret you surface. If you can't name the authorization, stop.

## The loop

1. **Establish the intended consumer first.** Name, for each secret, the single process or workload that is
   supposed to read it and nothing else. This is the false-positive killer: a secret delivered as a file
   mounted only into the one container that needs it, with the secret object readable by only that workload's
   service account, is correctly scoped. Name the intended consumer, then enumerate every other reader.

2. **Check the delivery mechanism.** Determine how each secret reaches the workload: an environment variable, a
   mounted file, or a fetched value. Environment delivery is the widest exposure, inherited by children and
   exposed by introspection; a file mounted only where needed is narrower. Note the mechanism, since it sets the
   default exposure.

3. **Enumerate in-pod readers.** For each secret, find what else in the pod can read it: sidecar and init
   containers sharing a mounted secret volume, child processes inheriting an env secret, and any process able
   to read another's environment. A multi-container pod turns one secret into a secret every container can see
   unless the mount is scoped.

4. **Trace leakage into logs, errors, and endpoints.** Follow whether the secret appears in log lines, error
   messages, stack traces, crash dumps, or debug and introspection endpoints that echo the environment. A
   secret in a log is exposed to everyone with log access; an endpoint that dumps the environment exposes every
   env secret to whoever can reach it.

5. **Scope the secret object's readers.** For the orchestrator's secret store, compute which identities and
   service accounts can read each secret object through RBAC and namespace access. Compare that to the single
   intended workload. A secret readable by a broad role, many service accounts, or a shared namespace is
   exposed to all of them regardless of how narrowly the pod consumes it.

6. **Confirm and record.** Confirm by reading a secret through an unintended path, a sidecar reading the mount,
   a debug endpoint echoing an env secret, a log line containing it, or an extra service account fetching the
   object, in scope and without exfiltrating. Kill the lead if each secret is delivered only to its one
   consumer, no other in-pod reader or log or endpoint exposes it, and the secret object is readable only by the
   intended workload's identity. Record the secret source, the extra-reader sink, and the exposure beyond the
   intended consumer.

## Where secret exposure surface leaks

- **Environment variables spread the secret.** An env secret is inherited by children, shown in process
  listings and crash dumps, and echoed by introspection endpoints.
- **A shared mount makes sidecars readers.** A secret volume mounted into a multi-container pod is readable by
  every container that shares it, not just the consumer.
- **Logs and errors carry secrets to a wider audience.** A secret in a log line or stack trace is exposed to
  everyone with log access, which is broader than the secret's holders.
- **Debug endpoints dump the environment.** An introspection or debug endpoint that echoes the environment
  exposes every env secret to whoever can reach it.
- **A broadly readable secret object exposes to all readers.** A secret the orchestrator lets many identities
  read is exposed to all of them regardless of the pod's narrow use.

## Worked example (a confirm and a kill)

> **Confirm.** A workload receives a database password as an environment variable, the pod runs a logging
> sidecar, and the application exposes a debug endpoint that prints its environment. The debug endpoint returns
> the password, and because it is an env variable a diagnostic child process also captures it in a crash dump.
> **Confirmed** secret exposure beyond the intended consumer via environment and debug endpoint, `high`,
> remediation = deliver the secret as a file mounted only into the consuming container, disable or gate the
> environment-dumping endpoint, and scrub secrets from logs and crash output.
>
> **Kill.** Each secret is delivered as a file mounted only into the one container that needs it, no sidecar or
> init container shares the mount, no log line or error contains a secret, no endpoint echoes the environment,
> and the secret object is readable only by that workload's dedicated service account. Nothing beyond the
> intended consumer can read the secret. **Killed**, `kill_reason` = "secret file-mounted to its sole consumer
> with no shared mount, no log or endpoint leakage, and a single-identity-readable secret object; no reader
> exists beyond the intended workload."

## Rationalizations to reject

- *"It is just an environment variable."* → Env secrets are inherited by children, shown in process listings
  and crash dumps, and echoed by debug endpoints; that is a wide exposure, not a convenience.
- *"Only our container is in the pod."* → Confirm no sidecar or init container shares the secret mount, and that
  no future sidecar will; a shared mount makes every container a reader.
- *"We do not log secrets."* → Confirm error messages, stack traces, and crash dumps do not either; secrets leak
  into logs through errors more often than through deliberate logging.
- *"The debug endpoint is internal."* → Internal reachability is still exposure; an environment-dumping endpoint
  hands every env secret to anyone who reaches it.
- *"The secret is in the orchestrator's secret store."* → Storage is not scope; compute which identities can read
  the object and compare to the one intended workload.

## Executing this in practice

You need each secret's delivery mechanism, the other containers and child processes that can read it, whether it
appears in logs, errors, or introspection endpoints, and the identities that can read the secret object. For
each secret, enumerate every reader beyond the intended consumer. Reading the workload spec and RBAC shows the
granted exposure; reading the secret through an unintended path shows whether it holds.

## Related

- `hunting-non-human-identity-and-secret-reachability` - the machine-credential reachability discipline; this
  skill is its in-workload delivery half.
- `reviewing-secrets-manager-access-policy-trust` - the managed-store companion; moving secrets there with a
  scoped policy is often the durable remediation.
- `auditing-observability-pipeline-collector-trust` - the logging side where a leaked secret travels; a secret
  in a log reaches wherever the pipeline forwards it.
- `mapping-pod-to-cloud-credential-reach` - what an exposed secret then reaches in the cloud; exposure and reach
  compound.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the workload secret, sink = the extra process,
  sidecar, log, or reader that can see it, evidence = the exposure beyond the intended consumer.
