---
name: mapping-service-account-impersonation-chains
description: >-
  Map service-account impersonation and token-generation paths that let a principal act as a
  more-privileged identity: a role granting impersonation or token creation on a service account, an actAs
  or token-creator permission that chains one identity into another, and a sequence of such grants that
  reaches a highly privileged account from a low-privileged start. Covers cloud service-account
  impersonation, short-lived-token generation, and the transitive graph where each impersonation grant is an
  edge from one identity to another. Use when principals can impersonate service accounts or mint tokens for
  them and you need to know where those edges lead. The principal holding an impersonation grant is the
  source, the impersonate or token-generation call is the sink, and the reachable privileged identity at the
  chain's end is the bug.
license: MIT
---

# Mapping service-account impersonation chains: when one identity can become another

Cloud identity models let a principal act as a service account through impersonation or short-lived-token
generation, and each such grant is an edge in a graph: whoever holds it can become the target identity and
inherit its permissions. Individually these grants look benign, a deployment principal impersonating a
deploy account, an automation user minting a token for a job account. The risk is transitive. If the target
account can itself impersonate a third, and that a fourth, a low-privileged starting principal can walk the
chain into an administrative identity no single grant intended. You find these by treating impersonation and
token-creator grants as edges and mapping where a starting principal can reach.

## When to use

- Principals can impersonate service accounts or generate short-lived tokens for them.
- An actAs, token-creator, or equivalent grant lets one identity act as another.
- You need to know the transitive reach of a starting principal through chained impersonation.

## Scope check

Test impersonation only in projects and organizations you own or are authorized to assess. A confirming
impersonation obtains credentials for the target identity, so stay inside the authorized projects and never
impersonate an account outside the engagement. If you can't name the authorization, stop.

## The loop

1. **Establish the starting principal and its intended reach first.** Name the principal you are analyzing
   from, a user, a workload identity, a CI principal, and what identities it is legitimately supposed to act
   as. This is the false-positive killer: an impersonation grant that lets a principal act only as the one
   account its job requires is correct even in a large project. Name the intended reach, then map the actual
   reach against it.

2. **Enumerate the impersonation and token-creator edges.** Inventory every grant that lets one identity act
   as another: impersonation roles, token-creator or actAs permissions, and any binding that produces
   credentials for a service account. Each is a directed edge from the grantee to the target account. Record
   the edges, not just the roles, because the graph is what matters.

3. **Build the reachability graph from the start.** Follow the edges: the starting principal can become each
   account it may impersonate, and then each account those accounts may impersonate, and so on. The reachable
   set is every identity the start can eventually act as. A chain of two or three benign-looking grants often
   reaches far beyond any single grant's intent.

4. **Locate the privileged endpoints.** Within the reachable set, identify the accounts that hold sensitive
   permissions: project or organization administration, security or logging control, broad data access. An
   edge into one of these from a low-privileged start is the finding, because it collapses the privilege
   separation the grants appeared to maintain.

5. **Check the guards that should bound the chain.** Scoping impersonation to the specific account a job
   needs, avoiding grants of token-creator on highly privileged accounts, and separating deployment identities
   from administrative ones each cut edges. Determine whether any binding grants impersonation broadly (on a
   folder, a project, or with a wildcard) rather than on a single account, since broad grants create many
   edges at once.

6. **Confirm and record.** Confirm by impersonating along the chain within owned projects to show the start
   reaches a privileged endpoint, stopping at proof and not exercising the endpoint's power destructively.
   Kill the lead if every impersonation grant is scoped to the single intended account, no chain from the
   start reaches a privileged endpoint, and no broad binding creates unintended edges. Record the starting
   principal, the impersonation sink, the edge chain, and the privileged endpoint reached.

## Where impersonation chains leak

- **Each grant is an edge, and edges compose.** A path of individually reasonable grants can reach an
  administrative identity the grants never intended to connect.
- **Broad-scope impersonation creates many edges at once.** A token-creator grant on a project or folder,
  rather than one account, lets the grantee act as every account in that scope.
- **Deployment and automation accounts are common bridges.** They hold impersonation grants by design, so
  reaching one often opens further edges.
- **A privileged endpoint collapses the separation.** Once the start can become an admin account, every
  lower boundary in between is moot.
- **The graph is invisible in per-grant review.** Reading one binding never reveals the chain; only the
  reachability map does.

## Worked example (a confirm and a kill)

> **Confirm.** A CI principal may impersonate a deploy account, which holds token-creator on an automation
> account, which can impersonate an account with project-admin. Mapping the edges shows the CI principal
> reaches project-admin in three hops. Impersonating along the chain in an owned project confirms it obtains
> project-admin credentials. **Confirmed** impersonation-chain privilege escalation, `high`, remediation =
> scope each impersonation grant to the single account its job needs, remove token-creator on the
> automation account for the deploy account, and separate the admin-capable account from the deployment path.
>
> **Kill.** Each principal may impersonate only the one account its workload requires, no automation account
> holds token-creator on another, and the admin-capable accounts are reachable from no deployment or CI
> principal. Mapping from every low-privileged start reaches no privileged endpoint. **Killed**, `kill_reason`
> = "impersonation grants scoped to single intended accounts with no chain from a low-privileged start to a
> privileged endpoint; the reachability graph reaches no admin identity."

## Rationalizations to reject

- *"Each grant is scoped to one account."* → Confirm the whole chain; single-account grants still compose into
  a multi-hop path to a privileged endpoint.
- *"That account is only for deployment."* → Deployment accounts are common bridges; check what further
  identities they can impersonate.
- *"No one would chain those."* → The chain is a reachability fact, not an intent; if the edges exist, the
  path is exploitable.
- *"It is just token generation."* → Minting a short-lived token for an account is acting as it; the token
  carries that identity's permissions.
- *"The grant is on the folder for convenience."* → A folder or project-scoped impersonation grant creates an
  edge to every account in that scope at once.

## Executing this in practice

You need the starting principal and its intended reach, every impersonation and token-creator edge in the
environment, the reachability graph those edges form, the privileged endpoints, and the scope of each grant.
Build the graph from the start and look for a path to any privileged account. Reading a single binding shows
one edge; mapping the transitive reach shows whether a low-privileged start can become an administrator.

## Related

- `hunting-iam-privilege-escalation-paths` - the permission-level escalation companion; impersonation edges
  are one class of escalation this mapping makes transitive.
- `auditing-cross-account-role-trust-boundaries` - the assume-role analog across accounts, where trust
  policies rather than impersonation grants form the edges.
- `hunting-non-human-identity-and-secret-reachability` - impersonated service accounts are machine
  identities; that skill governs what each reachable identity can then access.
- `adjudicating-taint-paths` - use it to trace a starting principal through the impersonation edges to a
  privileged endpoint.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the principal holding an impersonation grant, sink =
  the impersonate or token-generation call, evidence = the reachable privileged identity at the chain's end.
