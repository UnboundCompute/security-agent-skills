---
name: auditing-observability-pipeline-collector-trust
description: >-
  Audit telemetry collectors and observability pipelines for trust they should not extend: a collector
  endpoint that ingests metrics, logs, or traces without authenticating the sender, a processor that
  executes or forwards based on attacker-controllable telemetry fields, a collector running with broad
  credentials whose exporters reach sensitive destinations, and an ingestion path where log or trace content
  becomes a command, a query, or a downstream request. Covers agents and gateway collectors for logs,
  metrics, and traces, where the pipeline reads data from many sources and acts on it. Use when a telemetry
  collector ingests from workloads or the network and forwards, transforms, or stores that data. The
  unauthenticated or attacker-shaped telemetry is the source, the collector processor or exporter is the
  sink, and the unauthenticated ingestion or the acted-upon field is the bug.
license: MIT
---

# Auditing observability pipeline collector trust: when telemetry is an untrusted input

A telemetry collector sits in a trusted spot, workloads and infrastructure send it their logs, metrics, and
traces, and it forwards, transforms, and stores that data with credentials that reach real backends. That
position makes it an under-examined trust boundary. The data it ingests is often accepted without
authenticating the sender, so anyone who can reach the ingestion endpoint can inject telemetry. The
processors that transform that data can be driven by fields the sender controls, and a processor that
executes, routes, or queries based on a telemetry field acts on attacker input. The collector's own
credentials and exporters can reach sensitive destinations, so compromising the pipeline reaches the
backends it writes to. You audit these by checking whether ingestion is authenticated and whether any
processor or exporter acts on attacker-controllable content.

## When to use

- A telemetry collector or agent ingests logs, metrics, or traces from workloads or the network.
- The collector transforms, routes, or stores telemetry, and processors may read sender-controlled fields.
- The collector runs with credentials whose exporters reach databases, object stores, or other backends.

## Scope check

Test collectors and pipelines only in environments you own or are authorized to assess, on non-production
telemetry. Injecting telemetry or exercising a processor touches a live pipeline, so keep any payload benign
and inside the authorized environment. If you can't name the authorization, stop.

## The loop

1. **Establish the intended senders first.** Name who is supposed to send telemetry to each ingestion
   endpoint: which workloads, which agents. This is the false-positive killer: an endpoint reachable only by
   authenticated intended senders, whose processors treat telemetry as inert data, is correct. Name the
   senders, then check ingestion and processing against that.

2. **Check ingestion authentication and reachability.** Determine whether each ingestion endpoint
   authenticates the sender or accepts telemetry from anyone who can reach it. An unauthenticated endpoint
   reachable beyond the intended senders lets an attacker inject arbitrary logs, metrics, or traces, which is
   the entry point for everything downstream. Confirm the network reachability and the auth requirement.

3. **Find processors that act on telemetry fields.** Read the pipeline's processors and ask which read
   sender-controlled fields to make a decision: routing on a field value, executing or shelling out,
   evaluating an expression, or building a query or a downstream request from field content. A processor that
   acts on an attacker-controllable field is the injection sink inside the pipeline.

4. **Trace telemetry content into command, query, and request sinks.** Follow log lines, trace attributes,
   and metric labels into any processor or exporter that turns them into a command, a database query, a
   search-engine query, or an outbound request. Telemetry is attacker-shaped when ingestion is
   unauthenticated, so content reaching these sinks is injection with the collector's privileges.

5. **Scope the collector's credentials and exporters.** Read what the collector's exporters can reach and with
   what credentials. A collector that can write to sensitive stores, query backends, or reach internal
   services holds a blast radius equal to those exporters. Confirm the credentials are scoped to the
   destinations the pipeline actually needs.

6. **Confirm and record.** Confirm by injecting benign telemetry into an unauthenticated endpoint and showing
   it is ingested, or by supplying a field that a processor acts on (a routing change or a benign downstream
   effect) in scope, without a destructive payload. Kill the lead if every ingestion endpoint authenticates
   intended senders, no processor acts on attacker-controllable fields as command/query/request structure, and
   the collector's exporters are scoped. Record the ingestion source, the processor or exporter sink, and the
   unauthenticated ingestion or acted-upon field.

## Where collector trust leaks

- **Unauthenticated ingestion accepts attacker telemetry.** An endpoint reachable beyond intended senders lets
  anyone inject logs, metrics, and traces the pipeline then trusts.
- **Processors that act on fields turn telemetry into input.** Routing, execution, or query construction from a
  sender-controlled field is injection inside the pipeline.
- **Telemetry content reaches command and query sinks.** A log line or trace attribute forwarded into a shell,
  a query, or a request carries attacker content with the collector's privileges.
- **The collector's exporters set the blast radius.** Broad exporter credentials mean a compromised pipeline
  reaches every backend they can write to.
- **Telemetry is treated as trusted by habit.** Because it comes from internal systems by design, the fact that
  it is attacker-injectable when unauthenticated is easy to overlook.

## Worked example (a confirm and a kill)

> **Confirm.** A gateway collector exposes an unauthenticated trace-ingestion endpoint reachable from the
> workload network, and a processor routes and enriches traces by executing a lookup built from a trace
> attribute. Injecting a trace with a crafted attribute drives the processor to make a benign outbound request
> the tester observes. **Confirmed** unauthenticated telemetry ingestion driving a processor request, `high`,
> remediation = authenticate senders on the ingestion endpoint, treat all telemetry fields as inert data in
> processors, and never build a command or request from a sender-controlled field.
>
> **Kill.** Every ingestion endpoint requires sender authentication and is reachable only from intended
> workloads, processors treat all telemetry fields as inert data with no execution, routing-by-content, or
> query construction, and the collector's exporters are scoped to the two destinations the pipeline writes to.
> Injected telemetry from an unauthorized sender is rejected. **Killed**, `kill_reason` = "ingestion
> authenticated to intended senders, processors treat telemetry as inert data, and exporters are scoped; no
> unauthenticated injection and no field acted upon as structure."

## Rationalizations to reject

- *"Only our workloads send telemetry."* → Confirm the endpoint authenticates them; an unauthenticated endpoint
  accepts telemetry from anyone who can reach it.
- *"Telemetry is just data we store."* → If a processor routes, executes, or queries on a field, that data is an
  input; trace it to the sink.
- *"It is on the internal network."* → Internal reachability is not authentication; an attacker on the workload
  network injects telemetry directly.
- *"The collector just forwards."* → Its exporters carry credentials to real backends; a compromised or injected
  pipeline reaches everything they can write.
- *"Log content is harmless."* → A log line forwarded into a shell, a query, or a request is injection with the
  collector's privileges.

## Executing this in practice

You need each ingestion endpoint's authentication and reachability, the intended senders, the processors that
read sender-controlled fields, the command/query/request sinks telemetry reaches, and the collector's exporter
credentials. For each endpoint, confirm ingestion is authenticated; for each processor, whether it acts on an
attacker field. Reading the pipeline config shows the intended trust; injecting benign telemetry shows whether
ingestion and processing hold.

## Related

- `auditing-serverless-event-source-trust` - the same "does the pipeline trust its trigger" question for
  event-driven functions; telemetry ingestion is an event source too.
- `auditing-security-logging-completeness` - the defensive companion; this skill guards the pipeline that
  logging depends on, so a poisoned collector undermines detection.
- `hunting-non-human-identity-and-secret-reachability` - the collector's exporter credentials are machine
  identities; that skill scopes what a compromised pipeline reaches.
- `adjudicating-taint-paths` - use it to confirm a telemetry field reaches a command, query, or request sink
  inside a processor.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the unauthenticated or attacker-shaped telemetry, sink
  = the collector processor or exporter, evidence = the unauthenticated ingestion or the acted-upon field.
