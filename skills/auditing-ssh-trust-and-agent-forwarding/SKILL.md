---
name: auditing-ssh-trust-and-agent-forwarding
description: >-
  Audit secure-shell trust hygiene, not cipher hardening: a forwarded authentication agent a
  remote host can abuse to log in as you elsewhere, client configuration or a proxy-command
  directive influenced by an untrusted source, host-key verification disabled or blind-accepted
  so a machine-in-the-middle succeeds, and authorized-key entries whose forced command can be
  escaped or whose source and forwarding are unrestricted. Covers agent-socket exposure on
  multi-user or untrusted hosts, config and proxy-command injection from attacker-controlled
  data, trust-on-first-use gaps, and permissive key options. Use when auditing how hosts and
  users establish secure-shell trust and what a compromised endpoint can reach. The forwarded
  socket, injected directive, or unverified key is the source, authentication or command
  execution as an unintended identity is the sink.
license: MIT
---

# Auditing SSH trust and agent forwarding: what a connected host can turn into your identity

Secure-shell security is usually discussed as ciphers and key lengths, which are largely solved.
The unsolved part is trust: who can use your authentication, whose host key you accept without
checking, and whose data shapes your client's behavior. A forwarded agent lets whatever host you
land on authenticate as you to everything your key opens; a proxy-command or config entry pulled
from an untrusted place runs on your machine or redirects your session; a disabled host-key check
turns any network position into a man in the middle; a loose authorized-key entry hands more than
intended. You find these by tracing where authentication and configuration flow and asking what a
compromised or untrusted endpoint can do with them.

## When to use

- You are auditing how users or automation establish secure-shell trust and what it reaches.
- Agents are forwarded, host keys are accepted, or client configuration comes from shared sources.
- A compromised or multi-tenant endpoint could abuse forwarded authentication or injected config.

## Scope check

Audit secure-shell trust only for hosts, accounts, and automation you own or are authorized to
assess. Using a forwarded agent or a key to reach systems outside scope is not part of the audit.
If you can't name the authorization, stop.

## The loop

1. **Map where authentication is forwarded and to which hosts.** Inventory the connections and
   automation that forward the authentication agent, and to what hosts the socket is exposed. A
   forwarded agent on a single-user, trusted bastion is contained; one exposed on a multi-user,
   internet-facing, or lower-trust host means anyone who can reach that socket authenticates as you
   to everything your key opens. Determine the trust of every host the agent reaches.

2. **Check who can use an exposed agent socket.** Where the agent is forwarded, check the socket's
   permissions and the host's other users and processes. A root or co-tenant on the far host can use
   the socket to sign challenges as you for the duration of the session, with no access to your key
   itself. An agent reachable by anyone but you on an untrusted host is the finding.

3. **Trace client configuration and proxy directives to their source.** Determine where the client's
   configuration, host aliases, and proxy-command directives come from, and whether any is influenced
   by an untrusted source: a configuration file checked into a shared repository, read from a
   world-writable path, or a host or option value built from attacker-controlled input. A proxy-command
   runs a local command when connecting; if its content or selection is attacker-influenced, connecting
   runs their command on your machine.

4. **Check host-key verification.** Determine how host keys are verified: strict checking against a
   trusted store, blind trust-on-first-use, or verification disabled entirely. Automation that disables
   checking or auto-accepts unknown keys accepts any machine in the middle; the first connection to a
   never-seen host over a hostile path is trusted forever. A disabled or blindly-accepting check is a
   standing interception exposure.

5. **Audit authorized-key options.** For keys that grant access, read the restrictions on each entry: a
   forced command that can be escaped through its arguments or an interactive option, a missing source
   restriction so the key works from anywhere, and agent, port, or session forwarding left enabled where
   it is not needed. A key meant for one narrow task that permits a shell, arbitrary forwarding, or use
   from any source grants far more than intended.

6. **Confirm and record.** Confirm by exercising the gap in scope: use an exposed agent socket to reach a
   host you should not, trigger an injected proxy-command, accept a substituted host key that a strict
   check would refuse, or escape a forced command. Kill the lead if agents are forwarded only to trusted,
   single-user hosts with a protected socket, configuration and proxy directives come only from trusted
   sources, host keys are strictly verified, and keys carry tight command, source, and forwarding
   restrictions. Record with the trust source and what it reached.

## Where secure-shell trust leaks

- **A forwarded agent is your identity on the far host.** Whoever can reach the socket authenticates as
  you everywhere your key works, without ever holding the key. Forward it only to hosts you fully trust.
- **A proxy-command is local execution.** Configuration or a directive from an untrusted source runs their
  command on your machine, or silently reroutes your session, the moment you connect.
- **Trust-on-first-use trusts the attacker if they are first.** A disabled or blindly-accepting host-key
  check makes any network position a viable man in the middle.
- **A forced command is only as strong as its escapes.** Argument injection, an interactive option, or
  permitted forwarding turns a restricted key back into a general one.
- **A key without a source restriction works from anywhere it leaks to.** Restrict where a key may be used,
  not only what it may do.

## Worked example (a confirm and a kill)

> **Confirm.** A deployment pipeline forwards its authentication agent to every host it touches, including a
> shared build host where other tenants have a shell. A co-tenant on that host uses the exposed agent socket
> to authenticate as the pipeline identity to a production host the pipeline's key also opens. **Confirmed**
> agent-forwarding abuse to lateral movement, `high`, remediation = stop forwarding the agent to shared hosts,
> use per-host constrained keys or a short-lived certificate, and confine any needed forwarding to a
> single-tenant jump host.
>
> **Kill.** The agent is forwarded only to a single-tenant, strictly-controlled jump host with a
> user-only socket; client configuration and proxy directives come only from root-owned, non-writable paths
> with no attacker-influenced values; host keys are verified strictly against a managed store with no
> auto-accept; and every authorized key carries a non-escapable forced command, a source restriction, and
> forwarding disabled. Every abuse attempt fails. **Killed**, `kill_reason` = "agent confined to a trusted
> single-tenant host, config and proxy from trusted sources only, strict host-key verification, keys restricted
> by command, source, and forwarding."

## Rationalizations to reject

- *"Forwarding the agent is convenient."* → It is your identity on every host you land on. Convenient and, on
  any shared or lower-trust host, a full lateral-movement primitive.
- *"They would need my private key."* → No. A forwarded agent signs challenges as you without the key ever
  leaving your machine; reaching the socket is enough.
- *"We disable host-key checking so automation does not hang."* → Then automation trusts any machine in the
  middle. Pin the keys and manage the store instead.
- *"The key is locked to a forced command."* → Can the command be escaped through its arguments or an
  interactive option, and is forwarding still allowed? A forced command with an escape is not a restriction.
- *"The config is just our standard file."* → From where - a shared repository, a writable path? A
  proxy-command from an untrusted source runs on your machine when you connect.

## Executing this in practice

You need the map of where agents are forwarded and to which hosts, the trust and multi-tenancy of those hosts
and the permissions on any forwarded socket, the sources of client configuration and proxy directives, the
host-key verification posture, and the option set on every authorized key. The audit is tracing trust: for
each, ask what an untrusted or compromised endpoint on the far side can do with it. Reading the intended setup
tells you the design; following the trust tells you the exposure.

## Related

- `hunting-non-human-identity-and-secret-reachability` - the keys and forwarded identities here are machine
  credentials; trace how far each actually reaches.
- `auditing-declared-vs-used-permissions` - a key or forwarded agent that grants more than its task needs is an
  over-grant in this form.
- `hunting-iam-privilege-escalation-paths` - agent-forwarding lateral movement is the on-host analog of chained
  identity assumption in the cloud.
- `enumerating-snmp-exposure` - a sibling network-service trust audit in the same lane.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the forwarded socket, injected directive, or unverified
  key, sink = authentication or command execution as an unintended identity, evidence = what it reached.
