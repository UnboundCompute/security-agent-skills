---
name: auditing-mfa-enrollment-and-reset-abuse
description: >-
  Audit multi-factor authentication enrollment, reset, and recovery for paths that let an attacker add their own
  factor or bypass the check: a first-factor session that can enroll a new authenticator without re-proving
  identity so a stolen password adds a second factor, an MFA reset or recovery flow guarded only by a weak signal
  (an email link, a knowledge question, a support request) that resets the factor to attacker control, a step-up
  prompt that can be skipped or is not enforced server-side on a sensitive action, backup codes that are weak,
  reusable, or issued without authentication, and an MFA-fatigue or push-bombing flow that approves on a single
  tap. Use when adding, resetting, or satisfying a second factor is the boundary between a stolen first factor
  and a full account takeover. The enrollment or reset request is the source, the attacker-controlled factor or
  bypassed check is the sink, and the missing identity proof or skippable step-up is the bug.
license: MIT
---

# Auditing MFA enrollment and reset abuse: the second factor is only as strong as how you enroll and reset it

Multi-factor authentication assumes the attacker who has the password still cannot pass the second factor, and
that assumption lives or dies in three places most audits skip: how a factor is enrolled, how it is reset or
recovered, and how a step-up is enforced. If an account with only the first factor satisfied can enroll a new
authenticator without re-proving identity, an attacker with a stolen password simply adds their own second
factor and now passes MFA legitimately. If the reset or recovery flow is guarded only by a weak signal, an
email link to an inbox the attacker controls, a knowledge question, a support request that does not verify hard,
then MFA resets back to attacker control and the second factor is decorative. If a step-up prompt on a sensitive
action can be skipped, or backup codes are weak, reusable, or handed out without authentication, or a push
approval lands on a single tap so fatigue or bombing gets an accidental yes, the factor is bypassed rather than
broken. The audit treats enrollment and reset as privileged actions that must re-prove identity, and checks that
every step-up is actually enforced. You audit this by holding only the first factor and trying to add, reset, or
skip the second.

## When to use

- An authentication system lets users enroll a second factor, reset or recover it, use backup codes, or satisfy
  a step-up prompt.
- Enrollment or reset may not re-prove identity, or recovery may be guarded only by a weak signal (email link,
  knowledge question, support request).
- Step-up may be skippable, backup codes may be weak or unauthenticated, or push approval may succeed on a
  single tap (MFA fatigue).

## Scope check

Test MFA enrollment and reset only against accounts and systems you own or are authorized to assess, with test
accounts. Enrolling, resetting, and bypassing factors changes real authentication state, so use test identities
and never add a factor to, reset, or take over an account that is not yours. If you can't name the
authorization, stop.

## The loop

1. **Establish what enrolling and resetting a factor should require first.** Name the identity proof each
   privileged authentication action demands: what re-authentication enrolling a new factor requires, what
   resetting or recovering a factor requires, when a step-up is mandatory, and how backup codes are issued. This
   is the false-positive killer: a system that requires a fresh strong proof to enroll or reset a factor,
   enforces step-up on every sensitive action, issues strong single-use backup codes only after authentication,
   and requires deliberate push approval is behaving correctly. Name the intended proof, then test each path.

2. **Test factor enrollment from a first-factor-only session.** With only the password satisfied (no second
   factor yet passed on this session), attempt to enroll a new authenticator. Confirm the system requires a
   fresh, strong identity proof before adding a factor. If a stolen-password session can add its own second
   factor without re-proving identity, the attacker owns MFA from that point on.

3. **Test the reset and recovery flow.** Exercise every way a factor can be reset or recovered and identify what
   guards each: an email or SMS link, a knowledge question, a support-driven reset, a recovery code. Confirm the
   guard is a strong proof, not a weak signal an attacker with the first factor also controls. A reset guarded
   only by an email link to a takeable inbox resets the factor back to the attacker.

4. **Test step-up enforcement.** For each sensitive action that should demand a step-up (changing security
   settings, viewing secrets, high-value operations), attempt it without satisfying the step-up, by skipping the
   prompt, replaying a prior state, or calling the action directly. Confirm the step-up is enforced server-side
   on the action, not just shown in the UI. A skippable step-up is no step-up.

5. **Test backup codes and push approval.** Check how backup or recovery codes are generated (strength,
   single-use, count) and whether they can be issued or viewed without authentication. Separately, test push
   approval: whether repeated prompts (fatigue or bombing) can be sent and whether a single tap approves without
   context or number matching. Weak or unauthenticated backup codes and single-tap push are both factor
   bypasses.

6. **Confirm and record.** Confirm with test accounts by enrolling an attacker factor from a first-factor-only
   session, resetting a factor through a weak recovery guard, skipping an unenforced step-up, or approving via
   backup-code or push weakness, without touching real accounts. Kill the lead if enrollment and reset require a
   fresh strong proof, step-up is enforced server-side, backup codes are strong single-use and post-auth, and
   push requires deliberate approval. Record the enrollment or reset request, the attacker-controlled factor or
   bypassed check, and the missing identity proof or skippable step-up.

## Where MFA enrollment and reset leak

- **Enrollment without re-proof.** A first-factor-only session that can add a new authenticator lets a stolen
  password enroll the attacker's own second factor.
- **Weak reset or recovery guard.** A factor reset guarded only by an email link, a knowledge question, or a
  soft support check resets MFA back to attacker control.
- **Skippable step-up.** A sensitive action that shows a step-up prompt but does not enforce it server-side is
  reachable without the second factor.
- **Weak or unauthenticated backup codes.** Backup or recovery codes that are guessable, reusable, or issued
  without authentication are a standing bypass of the factor.
- **Single-tap push and MFA fatigue.** A push that approves on one tap, with no number matching or context and
  no limit on repeated prompts, lets fatigue or bombing get an accidental approval.

## Worked example (a confirm and a kill)

> **Confirm.** After entering a valid password, but before satisfying any second factor, the account settings
> page lets the session enroll a new authenticator app and does not require the existing second factor or any
> fresh proof. On a test account, an attacker holding only the stolen password enrolls their own authenticator
> and from then on passes MFA legitimately, locking the real user's factor out of the decision. **Confirmed**
> attacker factor enrollment from a first-factor-only session, `high`, remediation = require a fresh strong proof
> (the existing second factor, or full re-authentication) before enrolling, changing, or removing any
> authenticator.
>
> **Kill.** Enrolling, changing, or removing a factor requires the existing second factor or full
> re-authentication; reset and recovery are guarded by a strong proof, not a weak email or knowledge signal;
> every sensitive action enforces step-up server-side; backup codes are strong, single-use, and issued only
> after authentication; and push requires number matching with limits on repeated prompts. A first-factor-only
> session cannot add a factor, a weak recovery signal does not reset one, and a step-up cannot be skipped.
> **Killed**, `kill_reason` = "enrollment and reset require a fresh strong proof, step-up is enforced server-side,
> backup codes are strong single-use post-auth, and push needs deliberate approval; a stolen first factor cannot
> add, reset, or bypass the second."

## Rationalizations to reject

- *"They already entered the password."* → The password is the factor assumed stolen; enrolling or resetting a
  second factor must re-prove identity beyond the password, or MFA protects nothing against a credential leak.
- *"Recovery has to be easy."* → Easy recovery guarded by a signal the attacker also controls is an attacker
  recovery path; require a strong proof to reset a factor even at the cost of some friction.
- *"The step-up prompt appears."* → A prompt shown in the UI is not enforcement; confirm the server rejects the
  sensitive action when the step-up is not satisfied, including on a direct call.
- *"Backup codes are for emergencies."* → Emergency codes are full bypasses; if they are weak, reusable, or
  viewable without authentication, the attacker uses them as the primary path.
- *"The user approved the push."* → A single-tap approval under a flood of prompts is not consent; require number
  matching and rate-limit prompts so fatigue and bombing cannot harvest an accidental yes.

## Executing this in practice

You need the enrollment flow's re-authentication requirement, every reset and recovery path and its guard, which
actions demand a step-up and how it is enforced, and how backup codes and push approvals work. With test
accounts, hold only the first factor and try to enroll a new factor, reset an existing one, skip a step-up, and
approve via backup code or push. Reading the enrollment and reset handling shows the intended proof; an attacker
factor added or a check bypassed from a first-factor-only session shows what the proof fails to require.

## Related

- `auditing-account-recovery-and-reset-trust` - the account-level recovery flow; MFA reset is a specific factor
  reset that shares the weak-recovery-signal failure mode.
- `auditing-webauthn-and-passkey-flows` - the enrollment and step-up mechanics for passkeys; adding a
  phishing-resistant factor still depends on proving identity at enrollment.
- `auditing-session-lifecycle-and-fixation` - the session that enrollment and step-up act on; whether a factor
  change re-issues or elevates the session is shared ground.
- `reviewing-rate-limiting-and-abuse-controls` - push-bombing and code-guessing are rate-limit failures; that
  skill covers the abuse controls this one depends on.
- [FINDING-SCHEMA.md](../../FINDING-SCHEMA.md) - source = the enrollment or reset request, sink = the
  attacker-controlled factor or bypassed check, evidence = the missing identity proof or skippable step-up.
