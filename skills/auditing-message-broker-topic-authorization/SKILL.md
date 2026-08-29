---
name: auditing-message-broker-topic-authorization
description: >-
  Audit message-broker topic and queue authorization for reach a client should not have: a wildcard
  subscription that receives another tenant's messages, a publish permission broad enough to inject into a
  control or command topic, a shared broker where topic naming is the only separation between tenants, and a
  client authenticated to the broker but not authorized per topic so any connection can read or write any
  topic. Covers MQTT, Kafka, AMQP, and similar brokers where publish and subscribe permissions on topics or
  queues are the boundary between producers, consumers, and tenants. Use when a broker carries messages across
  trust boundaries and topic-level authorization is the control. The client publishing or subscribing is the
  source, the topic or queue it reaches is the sink, and the wildcard or missing per-topic authorization that
  admits it is the bug.
license: MIT
---

# Auditing message-broker topic authorization: when a wildcard subscribes to everyone

A message broker is a shared channel that many producers and consumers, often many tenants, talk through, and
topic authorization is what keeps one client's messages from reaching another's. That boundary is thinner than
it looks. Authenticating to the broker is not the same as being authorized per topic: a broker that checks the
connection but not the topic lets any authenticated client read or write any topic. Wildcards make it worse,
a subscription with a wildcard can match another tenant's topics and receive their messages, and a broad
publish permission can inject into a control or command topic that consumers act on. And on a shared broker,
topic naming is frequently the only thing separating tenants, which is a convention, not an enforced boundary.
The audit asks, for each client, exactly which topics it can publish to and subscribe to, and whether that
matches its role. You audit this by enumerating effective topic permissions and testing cross-tenant reach.

## When to use

- A broker (MQTT, Kafka, AMQP, or similar) carries messages across producer, consumer, or tenant boundaries.
- Clients authenticate to the broker but topic-level publish and subscribe authorization may be missing.
- Wildcard subscriptions or broad publish grants may cross topics or tenants on a shared broker.

## Scope check

Test broker authorization only on brokers you own or are authorized to assess, on non-production topics.
Subscribing or publishing exercises a live broker and can receive or inject real messages, so use
non-production topics and never read another tenant's real messages or inject into a live command topic. If
you can't name the authorization, stop.

## The loop

1. **Establish each client's intended topics first.** Name, per client or tenant, exactly which topics it
   should publish to and which it should subscribe to. This is the false-positive killer: a client authorized
   to precisely its own topics for precisely the direction it needs, with no wildcard crossing tenants, is
   correctly scoped. Name the intended topic map, then compare effective permissions to it.

2. **Confirm per-topic authorization exists.** Determine whether the broker authorizes publish and subscribe
   per topic or merely authenticates the connection. A broker that admits any authenticated client to any topic
   has no topic boundary at all, so the first check is whether topic-level access control is enforced, not just
   connection authentication.

3. **Enumerate wildcard and broad grants.** For each client, read its publish and subscribe permissions and
   flag wildcards and broad prefixes. A wildcard subscription can match other tenants' topics and receive their
   messages; a broad publish grant can reach control, command, or other tenants' topics. Compute what each
   wildcard actually matches across the topic space, not what it was intended to match.

4. **Check the direction of each grant.** Confirm publish and subscribe are scoped separately and correctly: a
   consumer that only needs to read should not be able to publish, and a producer that only needs to write
   should not be able to subscribe to responses it should not see. A grant that conflates the two directions
   widens reach beyond the client's role.

5. **Check tenant separation on a shared broker.** Where tenants share a broker, determine whether topic naming
   is the only separation or whether per-tenant authorization enforces it. If naming is the boundary, test
   whether a client can publish or subscribe to another tenant's topic by naming it, since a naming convention
   without enforcement is not isolation.

6. **Confirm and record.** Confirm by subscribing to or publishing on a topic a client should not reach, a
   wildcard receiving another tenant's non-production message, or a publish into a control topic, without
   reading real messages or injecting into a live command path. Kill the lead if the broker enforces per-topic
   authorization, no wildcard or broad grant crosses tenants or reaches control topics, publish and subscribe
   are separately scoped, and tenant separation is enforced rather than by naming alone. Record the client
   source, the topic or queue sink, and the wildcard or missing per-topic authorization that admitted it.

## Where topic authorization leaks

- **Connection auth is not topic auth.** A broker that authenticates the client but not the topic lets any
  connection read or write any topic.
- **A wildcard subscription catches other tenants.** A subscription wildcard that matches beyond the client's
  topics receives messages it was never meant to see.
- **A broad publish grant reaches control topics.** A wide publish permission injects into command or control
  topics that consumers act on.
- **Publish and subscribe conflated widen reach.** A grant that does not separate the two directions lets a
  producer read responses or a consumer inject messages.
- **Naming is not isolation.** On a shared broker, topic naming as the only tenant separation is a convention a
  client can step outside by naming another tenant's topic.

## Worked example (a confirm and a kill)

> **Confirm.** A shared MQTT broker authenticates clients but authorizes subscriptions loosely, and a tenant's
> client holds a wildcard subscription intended for its own device topics. The wildcard also matches another
> tenant's device topics under a shared prefix, so the client receives the other tenant's non-production
> telemetry, and a broad publish grant lets it publish into a command topic those devices act on. **Confirmed**
> cross-tenant reach via wildcard subscription and broad publish, `high`, remediation = enforce per-topic
> authorization scoped to each tenant's own topics, replace wildcards with explicit per-tenant topic grants,
> and separate publish and subscribe permissions so no client can inject into command topics it only consumes.
>
> **Kill.** The broker authorizes every publish and subscribe per topic, each client is granted only its own
> tenant's topics with no wildcard that crosses tenants, publish and subscribe are scoped separately to the
> client's role, and tenant separation is enforced by authorization rather than topic naming. A client naming
> another tenant's topic is refused for both directions. **Killed**, `kill_reason` = "per-topic authorization
> enforced with no cross-tenant wildcard, publish and subscribe separately scoped, and enforced tenant
> separation; no client reaches a topic outside its role."

## Rationalizations to reject

- *"Clients authenticate to the broker."* → Authentication is not per-topic authorization; confirm the broker
  checks the topic, or any authenticated client reads and writes any topic.
- *"The wildcard is just for our topics."* → Compute what the wildcard actually matches across the whole topic
  space; a shared prefix makes it catch other tenants.
- *"They only consume, they cannot publish."* → Confirm publish and subscribe are separately scoped; a
  conflated grant lets a consumer inject into command topics.
- *"Tenants use their own topic names."* → Naming is a convention, not a boundary; test whether a client can name
  and reach another tenant's topic.
- *"It is an internal broker."* → Internal brokers still carry cross-tenant and control messages; topic
  authorization is the boundary regardless of network placement.

## Executing this in practice

You need each client's effective publish and subscribe permissions, whether the broker enforces per-topic
authorization, what every wildcard actually matches, the separation of the two directions, and how tenants are
separated. For each client, compare its effective topic reach to its role and test a cross-topic or cross-tenant
access. Reading the broker's authorization rules shows the intended map; subscribing or publishing to a
should-be-unreachable topic shows whether it holds.

## Related

- `auditing-observability-pipeline-collector-trust` - telemetry often flows over a broker; unauthenticated
  ingestion and broad topic reach are the same trust gap at different points.
- `auditing-namespace-as-tenant-boundary` - the same tenant-separation-by-convention failure, here on broker
  topics instead of Kubernetes namespaces.
- `auditing-serverless-event-source-trust` - broker topics are event sources for consumers; whether the consumer
  trusts the message is the paired question.
- `mapping-attack-surface` - use it to enumerate the brokers, topics, and clients crossing trust boundaries
  before auditing each grant.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the client publishing or subscribing, sink = the topic
  or queue it reaches, evidence = the wildcard or missing per-topic authorization that admitted it.
