---
name: auditing-ble-and-gatt-authorization
description: >-
  Audit a Bluetooth Low Energy device for missing authorization on its GATT attributes: a characteristic
  performing a sensitive action or revealing sensitive data readable or writable by any peer, pairing or bonding
  that is not required or falls back to an unauthenticated Just Works mode with no link encryption, an
  authorization decision the device pushes to the mobile app instead of enforcing on the peripheral, and a
  replayable command a sniffer can capture and resend. Covers BLE peripherals, wearables, locks, medical and IoT
  devices, and their GATT services where a connected peer reads or writes characteristics. Use when a peer can
  connect over BLE and the peripheral's own enforcement of who may read or write each characteristic is the
  boundary. The unauthorized connected peer or captured command is the source, the sensitive read, write, or
  action on the peripheral is the sink, and the missing pairing, characteristic-level authorization, or replay
  protection is the bug.
license: MIT
---

# Auditing BLE and GATT authorization: the peripheral must enforce access, not the app

A Bluetooth Low Energy peripheral exposes its functionality as GATT characteristics that any peer in radio
range can attempt to connect to and read or write, and the security question is entirely on the peripheral:
does it enforce who may touch each characteristic, or does it assume the only client is its own trusted mobile
app. The common failure is trusting the app. The app shows a login and a permission model, but the peripheral
itself accepts reads and writes from any connected peer, so a peer that skips the app and talks to the GATT
server directly performs the sensitive action, unlocking, dumping data, changing a setting, with no check.
Pairing and bonding, which establish link encryption and a persistent trust relationship, are frequently not
required, or fall back to the unauthenticated Just Works association that gives encryption without
authenticating who is on the other end, so a sniffer or a man in the middle reads the traffic. And commands
sent without authentication can be captured over the air and replayed. The audit connects to the peripheral as
an arbitrary peer, enumerates its GATT attributes, and checks whether the device, not the app, enforces
authorization on each. You audit this by talking to the GATT server directly and seeing what it lets an
unauthorized peer do.

## When to use

- A BLE peripheral (wearable, lock, sensor, medical or IoT device) exposes GATT characteristics a connected
  peer can read or write.
- Pairing or bonding may not be required, or may fall back to unauthenticated Just Works with no MITM
  protection.
- Authorization may be enforced in the mobile app rather than on the peripheral, or commands may be replayable.

## Scope check

Test BLE devices only on hardware you own or are explicitly authorized to assess, in a controlled RF
environment. Connecting to a peripheral, writing characteristics, and sniffing radio traffic exercise a real
device and a shared spectrum, so use your own devices and never connect to, sniff, or actuate a device that is
not yours or affect nearby devices. If you can't name the authorization, stop.

## The loop

1. **Establish which characteristics are sensitive and how the device should protect them first.** Enumerate
   the GATT services and characteristics and name which perform sensitive actions or reveal sensitive data.
   Then name the intended protection: which require authenticated pairing/bonding, and which the peripheral
   itself authorizes per read/write. This is the false-positive killer: a peripheral that requires authenticated
   pairing, enforces characteristic-level authorization on the device for every sensitive attribute, and
   protects commands against replay is behaving correctly. Name the sensitive attributes and the intended
   protection, then connect as an arbitrary peer.

2. **Connect as an unauthorized peer and enumerate GATT.** Connect to the peripheral without going through its
   mobile app, enumerate all services and characteristics and their permissions, and attempt to read and write
   the sensitive ones. A characteristic that a bare peer can read or write is protected only by whatever the
   device enforces, so this reveals the real boundary.

3. **Test pairing and bonding requirements.** Determine whether sensitive operations require authenticated
   pairing and bonding, or whether the peripheral serves them to an unpaired peer. Check the pairing method: an
   unauthenticated Just Works association gives link encryption without authenticating the peer, so it does not
   stop a man in the middle. Missing or unauthenticated pairing means the link is not trustworthy.

4. **Test where authorization is enforced.** Compare what the app blocks against what the peripheral blocks.
   Perform a sensitive action by writing the characteristic directly, bypassing the app's checks entirely. If
   the action succeeds, authorization lives in the app, not on the device, which is no authorization at all
   against a direct peer.

5. **Test replay and command authentication.** Capture a legitimate command over the air (or from a prior
   session) and resend it to the peripheral. Confirm whether commands are authenticated and bound so a captured
   one cannot be replayed. A replayable, unauthenticated command lets a sniffer actuate the device without ever
   pairing.

6. **Confirm and record.** Confirm on your own device by performing a sensitive read, write, or action as an
   unauthorized peer, over an unauthenticated link, or by replaying a captured command, in a controlled RF
   environment and without touching a device that is not yours. Kill the lead if the peripheral requires
   authenticated pairing/bonding, enforces characteristic-level authorization on the device for every sensitive
   attribute, and protects commands against replay. Record the unauthorized peer or captured command, the
   sensitive read, write, or action, and the missing pairing, device-side authorization, or replay protection.

## Where BLE authorization leaks

- **World-accessible characteristics.** A sensitive characteristic any connected peer can read or write is
  protected only by whatever the device itself enforces, which is often nothing.
- **No required pairing/bonding.** Serving sensitive operations to an unpaired peer means there is no
  authenticated link at all.
- **Just Works association.** Unauthenticated pairing gives encryption without authenticating the peer, so a man
  in the middle still reads and relays the traffic.
- **Authorization in the app, not the device.** Access checks the mobile app performs are bypassed by a peer
  that talks to the GATT server directly; enforcement must be on the peripheral.
- **Replayable commands.** An unauthenticated command captured over the air and resent actuates the device with
  no pairing and no live authorization.

## Worked example (a confirm and a kill)

> **Confirm.** A BLE smart lock exposes an "unlock" characteristic. The mobile app requires the user to log in
> before showing the unlock button, but the peripheral accepts a write to the unlock characteristic from any
> connected peer without pairing. Connecting as a bare peer and writing the characteristic directly unlocks the
> device, bypassing the app's login entirely. **Confirmed** unauthorized actuation via device-side missing
> authorization, `critical`, remediation = require authenticated pairing and bonding, enforce authorization on
> the peripheral for the unlock characteristic (not in the app), and authenticate commands so a direct or
> replayed write cannot actuate.
>
> **Kill.** The peripheral requires authenticated (MITM-protected) pairing and bonding before any sensitive
> operation, enforces authorization on the device for every sensitive characteristic so a bare peer's read or
> write is refused, and authenticates commands with a freshness binding so a captured command cannot be
> replayed. A direct write from an unauthorized peer, an unauthenticated link, and a replayed command all fail.
> **Killed**, `kill_reason` = "peripheral requires authenticated pairing, enforces characteristic-level
> authorization on the device, and binds commands against replay; an unauthorized peer performs no sensitive
> read, write, or action."

## Rationalizations to reject

- *"Only our app talks to it."* → Any peer in range can connect to the GATT server directly; authorization the
  app enforces is no authorization against a peer that skips the app.
- *"It is paired."* → Confirm the pairing is authenticated (MITM-protected), not Just Works; unauthenticated
  pairing encrypts the link without authenticating who is on it.
- *"The signal is short range."* → Range is extendable with better antennas, and an attacker only needs to be in
  range once; proximity is not authorization.
- *"Commands are encrypted."* → Encryption without replay protection still lets a captured ciphertext be
  resent; bind commands to a nonce or counter so a replay is rejected.
- *"It is just a sensor."* → Read access to a sensor can reveal sensitive data (presence, health, location), and
  the same missing enforcement usually applies to its writable settings too.

## Executing this in practice

You need the device's GATT service and characteristic map with permissions, its pairing and bonding
requirements and method, where each sensitive action's authorization is actually enforced (app versus
peripheral), and whether commands are replay-protected. Connect as a bare peer, enumerate and exercise the
sensitive characteristics, attempt actions bypassing the app, and replay a captured command. Reading the
device's GATT configuration shows the intended protection; a sensitive action performed by an unauthorized peer
shows whether the peripheral enforces it.

## Related

- `auditing-ota-and-firmware-update-channel-trust` - the update channel of the same class of device; GATT
  authorization and firmware-update trust are the two main remote surfaces of a BLE peripheral.
- `hunting-firmware-secrets-and-debug-interfaces` - a shared key or credential baked into the device firmware
  often underlies weak BLE authentication; the two failures compound.
- `mapping-attack-surface` - use it to enumerate a device's radio and network interfaces before auditing the
  GATT server specifically.
- `hunting-broken-object-level-authorization` - the device-side enforcement this skill checks is the same
  authorization-at-the-resource discipline applied over a BLE link instead of an API.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the unauthorized connected peer or captured command,
  sink = the sensitive read, write, or action on the peripheral, evidence = the missing pairing, device-side
  authorization, or replay protection.
