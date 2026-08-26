---
name: hunting-connection-string-and-jdbc-url-injection
description: >-
  Hunt injection into database connection strings and JDBC or driver URLs where untrusted input sets the
  host, a driver property, or a URL parameter, turning a data connection into a request to an attacker
  server or an unsafe driver feature. Covers a tenant, hostname, or option taken from input and spliced
  into a connection URL, driver properties that enable local file reads, arbitrary command execution, or
  class loading, and multi-attribute connection strings where an extra property overrides a security
  setting. Use when an application builds a database or service connection string from user or tenant input
  rather than from fixed configuration. The untrusted value that becomes a connection host or property is
  the source, the connect call is the sink, and the dangerous driver feature or redirected endpoint it
  reaches is the bug.
license: MIT
---

# Hunting connection-string and JDBC-URL injection: when the connection target is attacker-set

Connection strings and driver URLs look like inert configuration, but they are richly featured: they carry
the host to connect to and a set of driver properties, and many drivers expose properties that read local
files, load classes, run initialization commands, or disable transport security. When an application builds
one of these from untrusted input, a tenant identifier, a hostname, a region, or an options blob, the
attacker can redirect the connection to a server they control, or append a property that flips a dangerous
driver feature on. The result ranges from credential capture at a rogue server to local file reads and code
execution through the driver. You find it by locating every place a connection string or URL is built from
input and asking which host or property the attacker can set.

## When to use

- An application builds a database or service connection string or driver URL from user or tenant input.
- A hostname, port, tenant, region, or driver option in a connection comes from a request or stored value.
- A multi-tenant or dynamic-datasource design selects the connection target at runtime from input.

## Scope check

Test connection-string handling only against applications and infrastructure you own or are authorized to
assess, on non-production systems. A confirming payload can cause the application to connect to a server you
control or trigger a driver feature, so stay inside the authorized boundary and use only endpoints you own
as the redirect target. If you can't name the authorization, stop.

## The loop

1. **Establish where connection strings and URLs are built.** Inventory every place a connection string,
   JDBC or driver URL, or datasource is constructed at runtime rather than read whole from fixed
   configuration. This is the false-positive killer: a connection built entirely from static configuration
   with no input-derived host or property is not injectable. Name the runtime-built connections first.

2. **Trace untrusted input into the host and the properties.** For each runtime-built connection, separate
   what the attacker can set: the host and port, the database or tenant name, and any driver property or URL
   parameter. Follow request values, tenant identifiers, and stored settings into each part. A value that
   sets the host can redirect the connection; a value that sets a property can enable a driver feature.

3. **Identify the dangerous driver features reachable.** Read which driver is in use and which of its
   properties are dangerous: local-file-read options, initialization or startup commands, class-loading or
   plugin properties, and transport-security toggles. An attacker who can append a property reaches whichever
   of these the driver exposes. Confirm the specific driver's property surface rather than assuming.

4. **Check for property override and redirection.** A multi-attribute connection string can let an appended
   property override an earlier security setting (disabling TLS, changing the auth mechanism, or adding an
   init command). A host taken from input can point at an attacker server that captures the credentials the
   application presents on connect. Determine whether the construction allows extra properties or an
   arbitrary host.

5. **Check the defenses that actually stop it.** Building the connection from fixed configuration and
   selecting only among a server-side allowlist of known targets, validating any input-derived host against
   that allowlist, and constructing the string through an API that sets known properties rather than
   concatenating an options blob each remove the vector. Escaping does not help, because the danger is a
   legitimate property, not a metacharacter. Determine which stands here.

6. **Confirm and record.** Confirm by supplying an in-scope input that redirects the connection to a server
   you own (observing the inbound connection) or that appends a benign, observable driver property, without
   pointing at any third-party host or triggering a destructive feature. Kill the lead if every connection is
   built from fixed configuration or a server-side allowlist of targets, no input-derived host or property
   reaches the string, and the construction cannot append arbitrary properties. Record the input, the
   connect sink, and the redirected host or enabled feature.

## Where connection-string injection leaks

- **The host is a redirect primitive.** An input-derived host points the connection at an attacker server
  that captures the credentials the app sends on connect.
- **Driver properties are the dangerous payload.** Local-file-read, init-command, and class-loading
  properties turn a data connection into file disclosure or code execution.
- **An options blob concatenated in is an override channel.** Appending a property can disable TLS or change
  the auth mechanism set earlier in the string.
- **Multi-tenant datasource selection is the common entry.** Choosing the connection by tenant or region from
  input is exactly where an unvalidated host or property slips in.
- **Escaping does not apply.** The attack uses legitimate properties and hosts, so there is no metacharacter
  to escape; only an allowlist of targets and known properties holds.

## Worked example (a confirm and a kill)

> **Confirm.** A multi-tenant service builds its datasource URL by inserting a tenant-supplied host into a
> JDBC URL template. A tenant sets the host to a server the tester controls, and the application connects and
> presents its database credentials to that server, which the tester observes. **Confirmed** connection-string
> host injection to credential capture, `high`, remediation = never take the connection host from input;
> select the datasource from a server-side allowlist of known tenant targets keyed by an authenticated tenant
> identifier, and build the URL through a driver API rather than string insertion.
>
> **Kill.** Connection strings are built from fixed configuration, the tenant identifier selects among a
> server-side allowlist of preconfigured datasources, and properties are set through the driver's typed API
> with no input-derived options blob. A tenant-supplied host or property never reaches the connect call.
> **Killed**, `kill_reason` = "connections built from fixed config and a server-side target allowlist, no
> input-derived host or property, and no arbitrary options appended; nothing attacker-set reaches the URL."

## Rationalizations to reject

- *"It is just a hostname."* → An input-set host redirects the connection to an attacker server that captures
  the credentials the application sends; the host is a redirect primitive.
- *"We escape the connection string."* → The attack uses legitimate properties and hosts, not metacharacters;
  escaping does nothing. Allowlist targets and properties instead.
- *"Only the tenant sets it."* → A tenant is an attacker for these purposes; select the datasource from a
  server-side allowlist keyed by an authenticated identity, not from a tenant-supplied value.
- *"The driver would not do that."* → Confirm the specific driver's property surface; many expose file-read,
  init-command, or class-loading properties that an appended option enables.
- *"It is internal configuration."* → If any part of the string is built from input at runtime, it is an
  injection sink regardless of how static the rest looks.

## Executing this in practice

You need every runtime-built connection string or URL, which parts (host, database, properties) are
input-derived, the driver's dangerous property surface, and whether construction allows an arbitrary host or
appended properties. For each connection, ask whether the attacker can set the host or a property and which
feature that reaches. Reading the construction shows what is input-derived; a redirect to a host you own or a
benign observable property shows whether the boundary holds.

## Related

- `exploiting-ssrf-to-cloud-metadata` - a redirected connection host is a server-side request primitive;
  both reach an attacker-chosen endpoint from inside the trust boundary.
- `hunting-non-human-identity-and-secret-reachability` - the credentials a redirected connection leaks are
  exactly the machine identities that skill inventories.
- `auditing-multi-tenant-isolation` - tenant-driven datasource selection is where this injection enters;
  the tenant boundary and the connection allowlist are the same control.
- `adjudicating-taint-paths` - use it to confirm a tenant or request value reaches the connect call as a host
  or property through configuration indirection.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the untrusted value that becomes a connection host
  or property, sink = the connect call, evidence = the redirected host or the enabled driver feature.
