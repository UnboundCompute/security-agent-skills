---
name: auditing-electron-ipc-trust
description: >-
  Audit an Electron desktop app for untrusted renderer content that reaches a Node or operating-system
  capability, after the webPreferences and the preload bridge surface are resolved. Covers nodeIntegration
  enabled with contextIsolation off, a preload that exposes raw ipcRenderer or a generic invoke
  passthrough, an ipcMain handler that trusts renderer input as a path, command, or URL, remote or
  attacker-influenced content loaded through loadURL with navigation unlocked, shell.openExternal called
  on a renderer-controlled string, and a custom protocol or deeplink routed into a privileged action
  without validation. Use when reviewing webPreferences, the preload and contextBridge surface, IPC
  handlers, and remote-content loading, not renderer-side markup injection the client-side DOM skill owns.
  Untrusted content in a renderer is the source, a Node or operating-system capability is the sink, and
  input crossing the bridge without validation while isolation is off is the bug.
license: MIT
---

# Auditing Electron IPC trust: when renderer content reaches a native capability

An Electron app is dangerous where content in a renderer, whether remote, injected through a
cross-site-scripting, or arriving over a custom protocol, reaches a Node or operating-system capability.
The marquee chain is a renderer bug escalating to code execution because the renderer can require Node
directly, or because the preload bridge hands untrusted input to a native call. You audit it by resolving,
per native sink, whether context isolation and the sandbox stand between the renderer and Node and whether
the preload and IPC surface validate what they pass. The single most important false-positive killer is
that context isolation and the sandbox being on make an otherwise alarming exposed symbol harmless, so
resolve those two settings before judging any bridge. Renderer-side markup injection itself belongs to the
client-side DOM skill; this skill owns the bridge and the IPC substrate.

## When to use

- You are reviewing an Electron app's window configuration, its preload scripts, and its IPC handlers.
- You see webPreferences, a contextBridge exposure, an ipcMain handler, or a loadURL or openExternal call.
- You want to know whether untrusted renderer content can reach a Node or operating-system capability.

## Scope check

Audit only apps you own or are authorized to assess, and drive a renderer-to-main path only against a
build in scope, an IPC handler that reaches a shell or the filesystem acts on your host. Adjudicate on the
window configuration and the handlers. If you can't name the authorization, stop.

## The loop

1. **Resolve context isolation and the sandbox first.** Read the webPreferences for each window: is
   nodeIntegration off, is contextIsolation on, is the sandbox on. These settings decide whether the
   renderer can reach Node at all, so an exposed require or a Node symbol is only exploitable when they are
   off. Settle them before judging any exposed surface, because they are the killer for most leads.

2. **Check node integration and isolation.** Look for nodeIntegration enabled with contextIsolation off,
   which lets any renderer, including one hosting an attacker's script, require a Node module such as the
   child-process module and execute a command directly.

3. **Check the preload bridge surface.** Look for a preload that exposes the raw ipcRenderer object, or a
   generic invoke-any-channel passthrough, or the filesystem or child-process modules over the
   contextBridge. A bridge that hands the renderer an open channel to every IPC handler, rather than named
   argument-typed functions, gives the renderer the whole main-process surface.

4. **Check the IPC handlers.** Look for an ipcMain handler that trusts renderer-supplied input as a
   filesystem path, a command, or a URL (path traversal, argument injection, server-side request forgery)
   without validating it against an allowlist or a normalized base.

5. **Check remote content, navigation, and protocol handlers.** Look for loadURL or loadFile pointed at
   remote or attacker-influenced content, navigation left unlocked (no will-navigate or window-open
   guard), shell.openExternal called on a renderer-controlled string (a file or protocol URL reaching a
   local handler), and a custom protocol or deeplink routed into a privileged action without parsing and
   validating the URL.

6. **Confirm and record.** Confirm by showing renderer-reachable content drives a Node or operating-system
   sink, because isolation or the sandbox is off, or because the preload or IPC surface passes untrusted
   input to a native capability. Kill the lead if contextIsolation and the sandbox are on so an exposed
   require or Node symbol is unreachable from the renderer, if the preload exposes only named typed
   functions over the contextBridge with no raw ipcRenderer and no filesystem or child-process module, if
   the IPC handler validates or normalizes the path or allowlists the channel and action, if the renderer
   only ever loads bundled local content and navigation is locked, if the openExternal argument is a
   constant or scheme-allowlisted to a safe scheme, or if the app runs a current Electron with the sandbox
   on by default and no legacy remote module. Record the renderer source, the native sink, and the
   settings that gate the path.

## Where IPC trust leaks

- **Isolation and the sandbox are the killer, so resolve them first.** With contextIsolation and the
  sandbox on, an exposed require or Node symbol is harmless; with them off, a renderer bug reaches Node.
- **A raw ipcRenderer bridge exposes every handler.** Handing the renderer the raw object or a generic
  invoke passthrough gives it the whole IPC surface; named typed functions expose only what you intend.
- **An IPC handler that trusts a renderer path is a native sink.** Renderer-supplied paths, commands, and
  URLs must be validated in the main process; the renderer is the untrusted side of the bridge.
- **Remote content in a renderer is the on-ramp.** loadURL of remote content, or unlocked navigation, puts
  attacker-influenced script one bug away from the bridge; bundled local content with locked navigation does
  not.
- **openExternal and custom protocols route to the OS.** A renderer-controlled string handed to
  openExternal or a deeplink handler reaches a local protocol handler; the argument has to be validated or
  scheme-allowlisted.

## Worked example (a confirm and a kill)

> **Confirm.** A window is created with nodeIntegration on and contextIsolation off, and it loads a remote
> page for a live feature. A cross-site-scripting in that page runs `require('child_process').exec` with an
> attacker command. **Confirmed** renderer-to-Node code execution through a non-isolated window loading
> remote content, `critical`, remediation = enable contextIsolation and the sandbox, disable
> nodeIntegration, expose only named typed functions over a preload bridge, and stop loading remote content
> in a privileged window.
>
> **Kill.** Another window has contextIsolation and the sandbox on, nodeIntegration off, and a preload that
> exposes a single `readConfig()` function returning a fixed file's contents; no raw ipcRenderer, no
> child-process or filesystem module crosses the bridge, and it loads only bundled local content with
> navigation locked. A renderer bug reaches no Node capability. **Killed**, `kill_reason` = "context
> isolation and the sandbox are on, the preload exposes only a typed read function, and no untrusted input
> reaches a native sink."

## Rationalizations to reject

- *"We expose require, but it is behind the bridge."* -> Are contextIsolation and the sandbox on? With them
  on, the renderer cannot reach that require; with them off, the bridge is not a barrier.
- *"The preload just forwards IPC."* -> Forwarding raw ipcRenderer or a generic invoke exposes every
  handler; expose named typed functions, not an open channel.
- *"The handler takes a path from the renderer."* -> Then validate it in the main process; the renderer is
  untrusted, and an unvalidated path is traversal or argument injection.
- *"It only opens a link."* -> With openExternal on a renderer-controlled string, a file or protocol URL
  reaches a local handler; constant or scheme-allowlist the argument.
- *"It is our own deeplink."* -> Is the URL parsed and validated before it drives an action? An unvalidated
  custom-protocol route is the desktop analogue of an open redirect into a privileged call.

## Executing this in practice

You need each window's webPreferences (nodeIntegration, contextIsolation, sandbox), the preload scripts and
exactly what they expose over the contextBridge, the ipcMain handlers and what they do with renderer input,
the loadURL and loadFile targets and the navigation guards, and the openExternal and custom-protocol
handlers. For each native sink, decide whether isolation and the sandbox gate the renderer and whether the
bridge validates what it passes. Reading webPreferences tells you whether the barrier is up; reading the
preload and handlers tells you what crosses it.

## Related

- `auditing-browser-extension-trust` - the browser-extension sibling with the same untrusted-content-to-
  privileged-capability shape, across a message boundary rather than the Electron preload bridge.
- `auditing-editor-extension-workspace-trust` - the editor-extension sibling, where the untrusted source is
  a cloned workspace rather than renderer content.
- `auditing-mobile-deeplink-trust` - the mobile analogue of the custom-protocol and deeplink bug shape; the
  parsing logic rhymes, the Electron IPC and native substrate is owned here.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = untrusted content in a renderer, sink = a Node or
  operating-system capability, evidence = the isolation and sandbox settings, the bridge surface, and the
  input crossing to the native call.
