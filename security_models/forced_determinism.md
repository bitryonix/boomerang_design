# Forced determinism

> **Last change — 2026-11-15:** included device-loss exposure without backup.

Forced determinism occurs when the Boomerang path becomes unavailable and the
participants must wait for the normal, deterministic withdrawal period. The
known paths are device loss, a late start, and peer non-cooperation.

## Attack vectors

### 1. Loss of a Boomlet card

**Scenario:** Without a Boomletwo backup, losing one peer's Boomlet can block
the Boomerang path, leaving timelocked normal-key recovery. The normal mnemonic
cannot reconstruct the lost Boomlet share.

**Potential guard:** Keep a Boomletwo backup.

**Remaining vulnerability:** Procedures for activating or revoking Boomletwo
and preventing both devices from signing concurrently are undefined. The backup cannot be treated as an available
recovery device until that protocol exists; it can also fail, be lost, or be
destroyed.

**Rationale:** Losing two separately stored cards should be less likely than
losing one, provided that the holder stores the backup more securely.

### 2. Late withdrawal

**Scenario:** Each Boomlet samples a fresh `mystery` only when an approved
withdrawal enters `DIGGING`. The peers know the configured range but not the
five sampled values or their maximum, so they cannot know whether a late
ceremony will finish before a fallback milestone. Narrowing the range reveals
more about completion time.

**Potential guard:** Choose milestones with enough withdrawal runway and roll
funds into a fresh setup before fallback.

**Remaining vulnerability:** Users may miss the rollover deadline.

### 3. Peer non-cooperation

**Scenario:** If one peer stops cooperating during withdrawal, all participants
must wait for the normal period.

**Guard:** The protocol assumes cooperation among all users, as does a 5-of-5
multisig. A 4-of-5 Boomerang path would weaken that commitment: four users could
bypass the fifth, so one honest user could no longer preserve the protocol's
guarantees.

**Remaining vulnerability:** Non-cooperation remains an open risk.
Reached peers must keep sending fresh pings and processing placeholders until
WT distributes a valid all-reached collection, so refusal can stall the
ceremony even after a peer reaches its own threshold.

## Existing safeguards

- Generate `mystery` once per withdrawal at `DIGGING` entry, retain it across
  retries in that ceremony, and erase it with active withdrawal state. A
  threshold inferred by WT cannot carry into a later withdrawal.
- Plan milestone runway and roll funds into a fresh setup before fallback.


## Earlier discussion

[Historical design discussion](forced_determinism_history.md)
