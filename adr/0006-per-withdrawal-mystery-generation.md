# ADR 0006: Generate Mystery Per Withdrawal at Digging Entry

- **Status:** Accepted
- **Recorded:** 2026-07-11

## Context

Boomerang's withdrawal delay depends on each Boomlet keeping a secret
`mystery` threshold. During the digging game, Boomlet increments `counter` as
valid progress rounds complete. When `counter >= mystery`, Boomlet sets
`reached_mystery_flag = true` in signed pings.

WT sees those pings and can observe when the reached flag first becomes true.
Even though `counter` is not a wire field, WT can often infer the threshold
from the ceremony transcript under clean progress conditions. After a mystery
has been reached, it should be treated as disclosed to WT for security
analysis.

If `mystery` is generated during setup and kept as long-lived Boomlet state,
then an aborted withdrawal can preserve a threshold that WT already observed or
inferred. WT could probe the threshold, cause or wait for abandonment, and use
the known value to predict timing in a later withdrawal from the same setup.

## Decision

`mystery` is per-withdrawal volatile state. Setup does not generate it, backup
does not export it, and Boomletwo does not create it during setup import.

Each Boomlet generates a fresh `mystery` only when an approved withdrawal
ceremony enters `DIGGING`:

```text
mystery =
  random_integer(
    MIN_TRIES_FOR_DIGGING_GAME_IN_BLOCKS,
    MAX_TRIES_FOR_DIGGING_GAME_IN_BLOCKS
  )
```

Boomlet erases the ceremony `mystery` with the rest of active withdrawal state
after signed PSBT export, explicit abort, or unrecoverable active-withdrawal
failure.

## Rationale

Generating at `DIGGING` entry gives `mystery` the narrowest useful lifetime.
The withdrawal identity and unanimous approval context are already fixed, and
the secret is created only immediately before the protocol phase that can
reveal it.

This removes the need to prove that every possible setup, backup, abort,
stall-to-abandon, crash-recovery, and successful-withdrawal path regenerates a
long-lived secret correctly. A disclosed threshold cannot survive into the next
withdrawal because the next withdrawal creates a new threshold at its own
digging entry.

It also keeps backups simpler. Backup state represents long-lived setup
authority and replay memory; it should not carry a future timing secret whose
only purpose is one active withdrawal ceremony.

## Security Effect

- WT observing `reached_mystery_flag` does not compromise future withdrawal
  timing after abort or successful export.
- Local compromise before withdrawal has less time to learn a future ceremony's
  threshold, because no such threshold exists yet.
- Backup export and Boomletwo import cannot accidentally clone a future
  withdrawal threshold.
- Crash recovery and abandonment paths have a simpler requirement: erase active
  withdrawal state, including `mystery`; do not regenerate replacement secrets
  outside withdrawal.
- The decision does not hide the current ceremony's reached transition from
  WT. It limits the disclosure to that ceremony.

## Consequences

The setup state model must not include `mystery` as long-lived Boomlet state.
Diagrams and explanatory documents must show setup parameter agreement followed
by later withdrawal-time mystery generation.

Implementations must ensure that retries inside the same active withdrawal do
not accidentally create a new mystery. Fresh mystery generation occurs only
when entering `DIGGING` for a new approved withdrawal ceremony. Once in
`DIGGING`, retry behavior remains bound to the active withdrawal state.

## Rejected Alternatives

- **Generate during setup and regenerate after successful withdrawal:** rejected
  because an aborted or abandoned ceremony can preserve a threshold after WT
  has observed it.
- **Generate during setup and regenerate on every abort path:** rejected because
  it depends on exhaustively and correctly handling every failure, recovery,
  crash, replacement, and abandonment path.
- **Generate a replacement when the threshold is reached:** rejected because the
  reached transition is the disclosure event, and the replacement secret would
  again become future long-lived state unless additional lifecycle rules are
  added.
