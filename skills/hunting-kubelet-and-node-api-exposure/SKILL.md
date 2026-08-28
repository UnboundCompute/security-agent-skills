---
name: hunting-kubelet-and-node-api-exposure
description: >-
  Hunt for node-level Kubernetes endpoints that are reachable and under-authenticated: a kubelet API that
  allows anonymous or unauthenticated requests to list pods, read logs, or exec into containers, a read-only
  kubelet port exposing pod and node data, a node-local metadata or debug endpoint reachable from a pod, and
  a kubelet authorization mode that authenticates but does not restrict what a caller can do. Covers
  Kubernetes nodes where the kubelet and other node-local services expose control over the pods on that node.
  Use when node endpoints may be reachable from pods or the network and their authentication is the only
  thing standing between a caller and node-level control. The reachable caller is the source, the kubelet or
  node endpoint is the sink, and the anonymous or unauthorized access it permits is the bug.
license: MIT
---

# Hunting kubelet and node API exposure: the control plane on every node

Every Kubernetes node runs a kubelet, and the kubelet is a powerful local API: it can list the pods on the
node, stream their logs, and exec into their containers. That makes each node a small control plane, and the
kubelet's authentication and authorization are the only thing between a caller who can reach it and node-level
control over every pod scheduled there. The exposures are concrete: a kubelet configured to allow anonymous
requests, a read-only port that serves pod and node data without auth, an authorization mode that
authenticates the caller but then allows the action anyway, and reachability from a pod or the network that
should never have existed. A caller who reaches an under-authenticated kubelet can read secrets from other
pods' environments, exec into neighbors, and pivot. You hunt this by finding node endpoints that are reachable
and testing whether their authentication actually refuses an unauthorized caller.

## When to use

- Kubelet or other node-local endpoints may be reachable from pods or the surrounding network.
- The kubelet may allow anonymous requests or expose a read-only port without authentication.
- The kubelet's authorization mode may authenticate a caller without restricting the action.

## Scope check

Test node endpoints only on clusters and nodes you own or are authorized to assess, on non-production nodes.
Reaching a kubelet can read other pods' data and exec into containers, so use a non-production node and stop
at proof rather than pivoting into neighboring pods. If you can't name the authorization, stop.

## The loop

1. **Establish who should reach node endpoints first.** Name what is supposed to talk to the kubelet and node
   services: the control plane, specific node agents, nothing from workload pods. This is the false-positive
   killer: a kubelet that requires authentication, uses an authorization mode that restricts actions, disables
   anonymous access and the read-only port, and is unreachable from pods is correctly closed. Name the intended
   callers, then find endpoints reachable beyond them.

2. **Find reachable node endpoints.** Determine which node-local endpoints are reachable and from where: the
   kubelet API port, the read-only kubelet port, and any node-local metadata or debug service, tested from a
   pod and from the network. Reachability from a workload pod is the key finding, since a pod compromise then
   reaches the node control surface.

3. **Test anonymous and unauthenticated access.** For each reachable endpoint, test whether it serves requests
   without credentials: anonymous kubelet access, the read-only port, or an unauthenticated debug endpoint. An
   endpoint that returns pod listings, logs, or node data to an anonymous caller is exposed regardless of any
   later authorization.

4. **Check the authorization mode, not just authentication.** Where the kubelet authenticates callers, confirm
   its authorization mode actually restricts actions rather than allowing any authenticated request. An
   authenticated-but-always-allowed configuration lets any caller who can present a credential exec into
   containers and read logs. Authentication without enforced authorization is not a control.

5. **Assess what the endpoint grants.** For each exposed endpoint, determine what a caller obtains: reading
   other pods' environments and secrets, streaming logs, exec-ing into containers, or retrieving node
   credentials. This sets the severity, and it is usually node-wide, since the kubelet governs every pod on
   the node.

6. **Confirm and record.** Confirm by reaching a node endpoint from an unintended position, an anonymous
   request that returns pod data, a read-only port serving node state, or an authenticated request the
   authorization mode should deny, on a non-production node and without pivoting into neighbor pods. Kill the
   lead if node endpoints require authentication, the authorization mode restricts actions, anonymous access
   and the read-only port are disabled, and the kubelet is unreachable from workload pods. Record the caller
   source, the node-endpoint sink, and the anonymous or unauthorized access it permitted.

## Where node endpoint exposure leaks

- **Anonymous kubelet access is node-level control.** A kubelet that serves anonymous requests hands pod
  listings, logs, and exec to anyone who reaches it.
- **The read-only port leaks pod and node data.** An enabled read-only kubelet port serves node and pod
  information without authentication.
- **Authentication without authorization is not enough.** An authorize-always mode lets any authenticated
  caller take any kubelet action, including exec.
- **Reachability from a pod bridges the escape.** A kubelet reachable from a workload pod turns a pod
  compromise into node-wide control.
- **The kubelet governs the whole node.** Its exposure is not one pod's problem; it reaches every pod scheduled
  on that node.

## Worked example (a confirm and a kill)

> **Confirm.** From a workload pod, the node's kubelet API port is reachable and configured to allow anonymous
> requests. An anonymous call lists every pod on the node and streams a neighboring pod's logs, and an exec
> request into a neighbor succeeds. **Confirmed** anonymous kubelet access granting node-level control,
> `critical`, remediation = disable anonymous authentication on the kubelet, set the authorization mode to
> enforce restrictions rather than always-allow, disable the read-only port, and block pod access to the
> kubelet port with network policy.
>
> **Kill.** The kubelet requires authentication, uses an authorization mode that restricts each action, has
> anonymous access and the read-only port disabled, and is unreachable from workload pods by network policy. An
> anonymous request is refused, an authenticated-but-unauthorized exec is denied, and pods cannot reach the
> port at all. **Killed**, `kill_reason` = "kubelet requires authentication with enforced authorization,
> anonymous access and read-only port disabled, and unreachable from pods; no caller obtains node-level control."

## Rationalizations to reject

- *"The kubelet needs a credential."* → Confirm the authorization mode restricts actions; an authenticate-then-
  always-allow kubelet grants exec to any caller with any credential.
- *"The read-only port is just metrics."* → It serves pod and node data without auth; disable it or treat it as
  an exposed information source.
- *"Only the control plane reaches the node."* → Test reachability from a pod; a kubelet reachable from a
  workload turns a pod compromise into node control.
- *"Anonymous access is off by default."* → Confirm it on this node; defaults drift across versions and
  provisioning, and one node with it on is enough.
- *"It is on the node network, not exposed."* → The node network is reachable from pods and often the cluster
  network; node-local is not the same as unreachable.

## Executing this in practice

You need the reachable node endpoints and from where (pod and network), each endpoint's authentication and, for
the kubelet, its authorization mode, whether anonymous access and the read-only port are enabled, and what each
exposed endpoint grants. For each endpoint, test anonymous and unauthorized access. Reading the kubelet
configuration shows the intended posture; a node endpoint that answers an unintended caller shows whether it
holds.

## Related

- `hunting-container-escape-surface` - a node escape and kubelet reach compound; escape reaches the node,
  the kubelet then grants control over every pod on it.
- `mapping-pod-to-cloud-credential-reach` - the kubelet and node role together are how a pod compromise becomes
  node and cloud access; map both reaches.
- `auditing-network-policy-segmentation-gaps` - blocking pod access to the kubelet port is a segmentation
  control; the two audits reinforce each other.
- `mapping-attack-surface` - use it to enumerate reachable node-local endpoints before testing each one's
  authentication.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the reachable caller, sink = the kubelet or node
  endpoint, evidence = the anonymous or unauthorized access it permitted.
