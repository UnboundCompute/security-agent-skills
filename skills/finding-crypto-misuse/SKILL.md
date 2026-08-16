---
name: finding-crypto-misuse
description: >-
  Find exploitable cryptographic misuse, not theoretical weakness: reused nonces
  (stream and counter/GCM keystream reuse, ECDSA private-key recovery from a
  repeated per-signature secret), padding oracles that decrypt ciphertext, hash
  length-extension on naive MAC constructions, predictable or reused IVs and keys,
  and a hash chosen for the wrong job. Use when reviewing code that encrypts, signs,
  authenticates, or hashes, or when a protocol rolls its own crypto. The finding is
  a concrete recovery or forgery, not "weak algorithm."
license: MIT
---

# Finding crypto misuse: the finding is a recovery, not a weak primitive

Most crypto findings that matter are not "the algorithm is broken"; they are the
algorithm used wrong in a way that hands you plaintext, a key, or a forgery. A cipher
is only as safe as its nonce discipline, a signature only as safe as its randomness,
a MAC only as safe as its construction. Finding crypto misuse means looking for the
specific misuse patterns that collapse to a concrete attack, and proving the attack
rather than flagging the primitive.

## When to use

- You are reviewing code that encrypts, decrypts, signs, verifies, MACs, or hashes.
- A protocol or service rolls its own crypto or composes primitives by hand.
- You need to separate an exploitable misuse from a cosmetic "weak crypto" alert.

## Scope check

Analyze crypto in code you own or are authorized to test. Demonstrate recovery or
forgery only against your own keys and data. If you can't name the authorization,
stop.

## The loop

1. **Inventory the crypto operations and their inputs.** List every place the code
   encrypts, decrypts, signs, verifies, MACs, or hashes, and for each capture the
   primitive, the mode, and where the key, nonce or IV, and randomness come from. The
   misuse lives in the inputs, not the primitive name.

2. **Hunt nonce and IV reuse.** For any stream or counter/GCM mode, does a (key,
   nonce) pair ever repeat across messages (a fixed IV, a counter that resets, a
   random nonce in too small a space)? Reused keystream lets an attacker combine two
   ciphertexts to strip the key, and GCM nonce reuse additionally leaks the
   authentication key, enabling forgery. This is the highest-yield check.

3. **Hunt signature-randomness reuse.** For ECDSA or DSA, is the per-signature secret
   drawn fresh from a strong source every time, or can it repeat or be predictable?
   Two signatures under the same secret reveal the private key by simple algebra. Look
   for fixed, derived, or low-entropy per-signature values in signing code.

4. **Hunt oracle and construction flaws.** Does decryption or unpadding reveal,
   through an error, a timing difference, or a distinct response, whether padding was
   valid? That is a padding oracle, and it decrypts ciphertext without the key. Is a
   MAC built as hash(secret then message) on a length-extendable hash? That is
   forgeable by length extension. Check for these constructions specifically.

5. **Hunt key and randomness hygiene.** Are keys hardcoded, reused across contexts,
   derived from low-entropy material, or generated from a non-cryptographic random
   source? Is a fast hash used where a slow password hash is required, or an unsalted
   hash where reversal matters? Each is a concrete downgrade, not a style issue.

6. **Prove the concrete attack and record.** A crypto finding is confirmed by naming
   the recovery or forgery it enables (recover this key, decrypt this ciphertext,
   forge this token) and, where possible, demonstrating it. Rate by that impact. Kill
   the lead if the input discipline holds: unique nonces, fresh randomness,
   constant-time comparison, sound construction.

## Where crypto collapses

- **The nonce is the whole security of the mode.** Repeat it once and confidentiality,
  and for GCM integrity, is gone.
- **Signature randomness is a secret.** Reused or predictable per-signature randomness
  is a private-key disclosure.
- **An error message is an oracle.** Any observable difference on decrypt can be a
  decryption oracle. Make failures indistinguishable and constant-time.
- **"It uses AES" says nothing.** The mode, the nonce, and the key handling decide
  security, not the cipher name.

## Worked example (a confirm and a kill)

> **Confirm.** A service encrypts each session token with a counter mode and a fixed,
> hardcoded IV, the same key for all users. Two tokens share keystream; combining them
> cancels the keystream and, with one known token, recovers the other's plaintext.
> **Confirmed** keystream reuse, `high`, remediation = a unique random nonce per
> message, never a fixed IV, and separate keys per context.
>
> **Kill.** A service signs with ECDSA drawing a fresh per-signature secret from a
> cryptographic source, uses a distinct random nonce per encryption, compares MACs in
> constant time, and returns an identical error on any decrypt failure. Repeated
> inspection finds no reuse or oracle. **Killed**, `kill_reason` = "unique nonces and
> fresh signing randomness, constant-time verification, indistinguishable decrypt
> failures; no recovery or forgery path."

## Rationalizations to reject

- *"It uses a strong algorithm."* → Strength is in the usage. A strong cipher with a
  reused nonce is broken.
- *"A random-nonce collision is astronomically unlikely."* → Only with a large space
  and a strong source. Check both; small or weak spaces collide.
- *"It's just an error message."* → Distinguishable decrypt errors are an oracle.
  Unify them and make timing constant.
- *"We hash the password."* → With a fast, unsalted hash that is reversible at scale.
  Use a slow, salted password hash.

## Executing this in practice

You need to see every crypto call site with its key, nonce or IV, and randomness
source, and to reason about whether any of those repeat or leak. A call graph and
dataflow into the nonce and key arguments help you prove reuse; the misuse-pattern
checklist and the concrete-attack requirement are the method.

## Related

- `adjudicating-taint-paths` - tracing whether attacker-controlled or repeated values
  reach a nonce or key.
- `writing-vuln-reports` - turning a recovery or forgery into a reproducible writeup.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the misused key, nonce, or
  randomness, sink = the recovery, decryption, or forgery it enables.
