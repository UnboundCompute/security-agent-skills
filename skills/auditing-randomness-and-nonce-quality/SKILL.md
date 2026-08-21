---
name: auditing-randomness-and-nonce-quality
description: >-
  Audit security-sensitive values for weak randomness: a non-cryptographic generator, a predictable or
  constant seed, a reused nonce or initialization vector, or an output too short to resist guessing,
  feeding a value whose only defense is unpredictability. Covers session and authentication tokens,
  password-reset and verification links, cross-site-request tokens and one-time codes, and
  cryptographic nonces or initialization vectors, drawn from a statistical generator instead of a
  cryptographic one, seeded from a timestamp or a constant, reused across messages under one key, or
  truncated into a space small enough to brute-force. Scoped to the randomness, seed, nonce lifecycle,
  and entropy length, not the choice of cipher, mode, or hash, which a separate skill covers. Use when
  a generated value gates access or protects a message. The generator is the source, the
  security-sensitive value is the sink, and predictability between them is the bug.
license: MIT
---

# Auditing randomness and nonce quality: when the secret is guessable by construction

A whole class of secrets is broken not by a flaw in the algorithm around them but by where their bytes
came from. A password-reset token from a statistical generator can be reconstructed from a few prior
outputs. A generator seeded from the clock produces the same token twice. A nonce reused across two
messages under one key can collapse a mode's confidentiality or leak its authentication key. A token
that is unpredictable but only thirty-two bits long is brute-forceable anyway. In every case the
cipher, mode, and hash can be perfectly chosen; the value is guessable because of the randomness, the
seed, the nonce lifecycle, or the length. You find these by tracing each security-sensitive value back
to the generator that produced it and asking whether an attacker can predict or repeat it.

## When to use

- A generated value is the only thing standing between an attacker and an account, a message, or a request.
- Tokens, links, codes, session identifiers, nonces, or initialization vectors are produced somewhere in the code.
- You want to separate a value that is unpredictable by construction from one that only looks random.

## Scope check

Assess randomness only in code you own or are authorized to review, and reproduce predictability only
against test data. Demonstrating that a real token is guessable can expose live accounts, so treat a
confirmed finding as sensitive and coordinate. If you can't name the authorization, stop.

## The loop

1. **Map security-sensitive values to their generator.** Inventory the values whose security property is
   unpredictability: session and authentication tokens, password-reset and email-verification links,
   cross-site-request tokens, one-time and device codes, API keys, and cryptographic nonces, salts, and
   initialization vectors. For each, trace back to the call that produced its bytes. The trace, not the
   variable name, tells you what the value actually is.

2. **Find non-cryptographic generators feeding a secret.** Flag any security-sensitive value whose bytes
   come from a statistical or linear-congruential generator, the kind meant for simulation and sampling,
   rather than a cryptographic random source. These generators are deterministic from an internal state
   that a handful of outputs can reconstruct, so their tokens are predictable to anyone who has seen a
   few.

3. **Find predictable or constant seeds.** Flag a generator explicitly seeded from a timestamp, a process
   identifier, a counter, a constant, or any value an attacker can derive. A seeded generator produces a
   reproducible stream, and seeding even a cryptographic generator from a low-entropy source reintroduces
   the predictability the generator was meant to remove.

4. **Find nonce and initialization-vector reuse.** Flag a nonce or initialization vector that is a
   constant, is derived deterministically from the message, or is not regenerated for every message under
   a given key. Reuse under one key is catastrophic for stream and counter constructions and can leak an
   authentication key outright, and here the defect is the nonce lifecycle, not the cipher or mode around it.

5. **Find insufficient entropy and predictable identifiers.** Flag a value that is generated correctly but
   truncated into a small space, a short numeric token used as a security token without a compensating
   throttle and expiry, or a sequential, time-based, or otherwise guessable identifier used where
   unguessability is required. Correct source, too few bits, is still brute-forceable.

6. **Confirm and record.** Confirm by reading the generator's source to establish it is non-cryptographic,
   seeded, reused, or too short, and by tracing the value to a sink that depends on unpredictability;
   where feasible, predict or collide a value from observed outputs on test data. Kill the lead if the
   generator is a cryptographic source behind an unfamiliar name, the value is not security-sensitive, or
   a short code is genuinely defended in depth by a strict throttle and short expiry. Record the
   generator, the sink, and the predictability shown.

## Where randomness leaks

- **The variable name lies; the generator tells the truth.** A value called a token can come from a
  simulation generator, and a plain-looking string can be cryptographic bytes. Read the source of the call.
- **A statistical generator is reconstructable, not just biased.** Its whole future and past follow from a
  few outputs, so one leaked token can predict the next.
- **Seeding is the back door into a good generator.** A cryptographic source seeded from the clock is as
  predictable as the clock. Never manually seed what is meant to seed itself.
- **A nonce is a lifecycle, not a value.** Freshness per message under a key is the property; a constant or
  a reset counter breaks it even when every other choice is correct.
- **Unpredictable and long are different requirements.** A perfectly random thirty-two-bit token is still
  brute-forceable; entropy length is its own axis.

## Worked example (a confirm and a kill)

> **Confirm.** A password-reset token is built from a statistical generator's output, stored, and emailed
> as a link. The generator is non-cryptographic and unseeded per request, so its state is recoverable from
> a few tokens an attacker requests for their own account, and the victim's token is then predictable
> within the reset window. **Confirmed** predictable password-reset token, `critical`, remediation =
> generate the token from a cryptographic random source with at least a hundred and twenty-eight bits, and
> bind it to a short expiry and single use.
>
> **Kill.** A helper with an unremarkable name returns a token that, read to its source, is a base-encoded
> block of bytes from the platform's cryptographic random source; a short numeric code elsewhere is
> cryptographically generated, single-use, expires in minutes, and is rate-limited to a few attempts. No
> value traces to a statistical generator, a fixed seed, or a reused nonce. **Killed**, `kill_reason` =
> "security-sensitive values derive from a cryptographic source with adequate length; the short code is
> defended by a strict throttle and short expiry; no reuse or predictable seed on any path."

## Rationalizations to reject

- *"It looks random to me."* -> Looking random and being unpredictable are different. A statistical
  generator's output looks random and is fully predictable from a few samples.
- *"It is only used for a token, not encryption."* -> A token is exactly a value whose security is
  unpredictability. A weak generator there is an account-takeover primitive.
- *"We seed it so it is different each run."* -> Seeding from the clock or a constant makes the stream
  reproducible. A cryptographic source needs no manual seed and is weakened by one.
- *"The nonce is generated, so it is fine."* -> Generated once and reused across messages under one key is
  the bug. Confirm it is fresh per message inside the encryption path.
- *"The code is random and short is convenient."* -> Short is brute-forceable unless a strict throttle and
  a short expiry carry the weight the length does not.

## Executing this in practice

You need the set of security-sensitive generated values, the exact generator call behind each, any seed
passed to it, the per-message path for nonces and initialization vectors, and the length of each output.
For each value, decide whether an attacker can predict it from prior outputs, repeat it, or brute-force
its space. Reading the generator's source settles what kind of randomness it is; tracing the value to its
sink settles whether unpredictability is the property that matters there.

## Related

- `finding-crypto-misuse` - the sibling scoped to cipher, mode, and hash selection; this skill deliberately
  covers the orthogonal axis of where the random bytes came from and how the nonce is used.
- `auditing-webauthn-and-passkey-flows` - relies on unguessable challenges; a weak generator there is a
  randomness finding this skill names.
- `auditing-device-code-and-pkce-flows` - a weak user code or authorization code is a grant bug rooted in
  the entropy this skill assesses.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the generator, sink = the security-sensitive
  value, evidence = the predicted, repeated, or brute-forced value from observed outputs.
