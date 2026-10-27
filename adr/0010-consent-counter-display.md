# ADR 0010. Consent Counter Display During Rotation and Backup Activation

- **Status:** Accepted
- **Recorded:** 2026-10-14

## Context

Boomlet and Boomletwo retain a five-country consent set and `consent_counter`
for replay protection. A backup can retain an older set after interrupted
rotation. Its activation check needs a version cue for the user.

Routine counter display reveals state advancement to a coercer who may already
know an answer, giving them reason to demand the current answer.

## Decision

Omit the counter from Boomlet's current-set rotation query. During input, begin
display only after ST authenticates the new-set query for its approved rotation,
which Boomlet releases after accepting the first answer and recording any
duress. Show the next counter as pending during selection and confirmation.
Display it as applied with authenticated completion after Boomletwo's receipt,
including identical countries and cancellation. Status and recovery follow the
same restrictions.

Display Boomletwo's stored counter with its first consent query during activation.
Keep it visible through retries and recovery until that answer is
accepted. Ordinary withdrawal and later backup consent queries omit it.

Keep the counter inside authenticated rotation and reset messages. Transfer the
new set and duress flag together and require Boomletwo's durable acknowledgment.

## Rationale

New set entry gives the user a version reference after the first duress check.
A coercer cannot inspect the initial query's counter before that opportunity
to record duress. Activation shows which stored version the backup checks without
another country-entry round during rotation. Authorization and replay protection
retain the internal counter.

## Consequences

Any valid five-country answer reaches selection and reveals the counter; a wrong
answer must already have set `under_duress`. The pending counter also reveals its
predecessor by subtraction. Advancement can reflect identical countries,
cancellation, or reset, so it does not prove the countries changed.

The first activation query exposes the backup's version before answering.
Recovery must preserve rotation display gating and distinguish the pending
first activation query from later queries across restart.

## Rejected Alternatives

- Displaying the counter on every query exposes state changes during ordinary use.
- Displaying it with Boomlet's current-set rotation query reveals advancement
  before the first duress check.
- Displaying it only during rotation provides no version cue during backup activation.
- Displaying it only during backup activation omits the version reference during rotation.
- Asking the user to confirm the new set on Boomletwo during every rotation adds
  an input round and recovery handling for interrupted confirmation.
