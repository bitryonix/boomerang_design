# Observed-consent decision proposal

| Item | Value |
| --- | --- |
| Status | Draft, not adopted |
| Phase 1 priority | 2 |
| Normative baseline | [`SPEC.md`](../../spec/SPEC.md) at `ed91233a4216be0c96f8f8d081276b7c94a39751` |
| Related gaps | DG-38, AR-35, AR-79, FM-30, R-12, R-21 |
| Security review | 2026-09-19; Boomletwo interactions rechecked 2026-09-22 |

## Problem and scope

The five-country `duress_consent_set` is copied from Boomlet to Boomletwo and
remains safe for the setup's lifetime. Fresh permutations and nonces prevent
transcript replay, but an attacker who learns a safe selection can recognize or
dictate it later. Consent replacement and backup activation remain unresolved.

The decision is whether to replace the setup, change the protocol, or explicitly
accept exposure. Observation includes recording, dictated input, and device
extraction. It does not grant signing keys, suppress other peers' signals, or
cancel rescue already triggered. Recovery protects later ceremonies; it cannot
undo disclosure during a coerced interaction.

## Decision map

| Goal | Candidate | Decision gate |
| --- | --- | --- |
| Keep an enrolled backup ready | Option 3, 4A, 5, or 7 | Commit one current consent epoch across devices and recovery authority. |
| Replace a stale backup before rotation | Variant 4D | Prove the old backup is excluded from every activation path. |
| Protect later ceremonies after unnoticed observation | Option 6 or 10 | Validate one-time answers or private cues under active coercion. |
| Add independent review | Option 11 | Define enforceable clearance and escalation. |

Rejected: ~~Option 1~~ requires action from other peers; ~~Option 2~~ retains an
exposed safe answer; ~~Variant 4B~~ requires two memorized sets; ~~Variant 4C~~
contacts the unavailable Boomlet during activation. ~~Variant 4E~~ permits
coerced enrollment at activation. ~~Options 8 and 9~~ permit coerced changes to
the consent authority used by an offline backup. The remaining options are
alternatives; none is adopted.

## Required properties

For fixed-set recovery, success requires these properties. Quarantine begins on
known or suspected disclosure; its enforcement on offline devices must be defined.

| ID | Property |
| --- | --- |
| OC-SR-01 | No eligible device classifies an observed set as safe after recovery. |
| OC-SR-02 | Only one set is safe per active Boomlet; no overlap preserves the exposed answer. |
| OC-SR-03 | Crashes, retries, and rollback cannot restore invalidated authority. Ambiguity blocks ordinary withdrawal. |
| OC-SR-04 | Backup activation cannot restore exposed consent or bypass required enrollment. |
| OC-SR-05 | Consent maintenance cannot grant signing authority or weaken the one-active-Boomlet invariant. |
| OC-SR-06 | Fixed-set enrollment uses trusted ST, two fresh nonce-bound confirmations, and a private environment. |
| OC-SR-07 | Transcripts and public records provide no offline test for guessing the secret. |
| OC-SR-08 | In-place maintenance occurs outside withdrawal and preserves SAR observability rules. |
| OC-SR-09 | A trusted interface identifies the current device, ceremony, and consent state. |
| OC-SR-10 | Device loss, stale backups, interrupted maintenance, and unavailable authorities have recovery actions. |

Broader designs must specify replacements for the fixed-set and unchanged-wire
requirements. Option 11 explicitly trades concealment for independent review.
Every option retains the signing-authority invariant.

## Common security limits

- Restore compromised devices and assess exposed keys before recovery. A new set
  cannot repair a device that leaks answers or ignores invalidation.
- The exposed set cannot be the sole maintenance credential. Authentication and
  two matching confirmations establish neither privacy nor freedom from coercion
  [R3]; the device cannot verify those environmental assumptions.
- After rotation, an old but valid set must follow the ordinary duress path.
  A distinctive rejection or retry would disclose classification.
- At Boomletwo activation, the original Boomlet is permanently offline and
  unresponsive. Any backup option must prove source exclusion and current state
  without contacting it; otherwise activation stays blocked. Signed old records
  do not prove freshness. Counter storage must resist rollback.
- Exclude known exposed sets using protected history. Lost history limits proof
  of lifetime non-reuse. Public hashes permit guessing; properly hiding
  commitments need not [R2]. Neither commitments nor equality proofs establish
  durable invalidation.
- Preserve pending rescue during maintenance. Bound retries and writes to limit
  denial of service. Protocol-controlled timing and errors must conceal
  classification, although human maintenance activity may reveal suspicion.
- Quarantine must be an authenticated durable transition ordered against active
  withdrawal, signing, and rescue duties. A stalled ceremony is still active;
  blocking spending must not strand a pending placeholder or acknowledgment.
- An external classifier must bind its result and SAR payload to the setup,
  device, withdrawal, exact outstanding check nonce and phase, authority, and
  consent epoch. One check consumes one result; exact delivery retries remain
  idempotent. Restart and epoch cutover must preserve pending-check state.
- Scheduled rotation limits unnoticed exposure only when later enrollment is
  private. Continuous disclosure of each replacement defeats fixed-set rotation.

Message sketches use `Active` for the current Boomlet and `Backup` for
Boomletwo. Arrows show candidate order, not assigned wire types.

## ~~Option 1. Full setup replacement~~

**Rejected.** Rollover requires action from other peers.

~~Quarantine, enroll a fresh setup, move funds to its descriptor, and retire the old
consent-bearing devices. Permit only a restricted rollover bound to the verified
new descriptor. Destination matching and peer approval alone do not authorize
signing. Transaction review, SAR acknowledgment, digging, and final signing
gates remain required unless a replacement guarantee is specified. Validate all
outputs, change, fees, replacement keys, milestones, and fallback paths.~~

**~~Message sequence~~**

- ~~User -> Active (quarantine request)~~
- ~~User -> Iso -> new Boomlet (fresh setup)~~
- ~~User -> ST -> new Boomlet (fresh consent enrollment)~~
- ~~Active -> WT -> peers (restricted rollover review and approvals)~~
- ~~WT -> SAR -> WT (placeholder and exact acknowledgment)~~
- ~~WT -> chain (signed rollover); chain -> User (confirmation)~~
- ~~User -> old devices (retirement)~~

### ~~Pros~~

- ~~Avoids paired consent epochs and reuses setup enrollment.~~
- ~~Confirmed rollover removes transferred funds from old spending paths and can
  replace other exposed keys.~~

### ~~Cons~~

- ~~Requires peers, fees, and confirmations while exposure persists. Competing
  spends, outstanding signatures, reorganizations, and fallback constrain timing.~~
- ~~Still needs durable quarantine and device retirement. Fresh setup creation
  alone cannot recover funds without existing spending authority.~~
- ~~Reusing exposed sets or compromised devices defeats recovery. Stop issuing old
  deposit addresses and define handling of later deposits to them.~~
- ~~If the rescue root or upload key was exposed, a fresh custody setup using the
  same password and SAR leaves that authority intact. Recovery needs a new
  doxing password and registration, while preserving old setup rescue duties.~~

## ~~Option 2. Immutable set with accepted exposure~~

**Rejected.** The exposed safe answer remains valid, so a coercer can dictate it
to suppress this user's duress signal in later ceremonies.

~~Continue accepting the same set after disclosure; replacement remains voluntary.~~

**~~Message sequence~~**

- ~~User -> ST -> Active (existing consent answer)~~
- ~~Active -> WT -> SAR (ordinary withdrawal and placeholder)~~
- ~~SAR -> WT -> Active (exact acknowledgment); withdrawal continues~~

### ~~Pros~~

- ~~Preserves availability without consent-update state or authority.~~
- ~~Retains signing checks, peer approvals, delays, and other peers' duress signals.~~

### ~~Cons~~

- ~~A coercer controlling input can force the affected user's checks to remain safe.~~
- ~~Nonces and shielding cannot restore the disclosed secret. This fails OC-SR-01
  and requires an explicit reduction in the security claim.~~

## Option 3. Paired rotation

Transfer a fresh set to the inactive backup and durably commit the same epoch on
both devices. Bind updates and receipts to setup, identities, epoch, and
transition. An uncertain participant blocks until decision evidence or recovery
is available. Two-phase commit can preserve safety while blocking [R1]; a witness
is a recovery and availability choice, not universally required for safety.

**Message sequence**

```text
User -> ST -> Active (fresh set, two confirmations)
Active -> Backup (encrypted set and proposed epoch)
Backup -> Active (durable prepare receipt)
Active -> Backup (commit); Backup -> Active (durable commit receipt)
```

### Pros

- Preserves one memorized answer, the descriptor, and service registrations.
- A completed update leaves backup consent current, subject to activation checks.

### Cons

- Requires both devices and stores the secret twice, increasing correlated exposure.
- Partial commits, rollback, and missing evidence can block recovery. A witness
  adds outage, privacy, and equivocation risks.
- Aborting after disclosure must return to quarantine, never the exposed answer.
  Updating consent must not activate signing authority.

## Option 4. Independent device consent state

### Variant 4A. Same set enrolled separately

Enter one fresh set independently on both devices. Authenticated private equality
checking can detect mismatch without publishing a guessable hash [R2].

**Message sequence**

```text
User -> ST -> Active (fresh set and confirmations)
User -> ST -> Backup (same set and confirmations)
Active <-> Backup (private equality and matching epoch receipts)
```

#### Pros

- Avoids transmitting the set between devices while retaining one memorized answer.

#### Cons

- Repeats exposure and error opportunities while retaining the coordination problem.
- Equality does not prove durable commit. A partial update must exclude the stale
  device; accepting both answers to mask mismatch violates OC-SR-02.

### ~~Variant 4B. Distinct pre-enrolled sets~~

**Rejected.** The user would need to memorize two separate sets.

~~Memorize one independent set per device and rotate each exposed set before that
device next activates.~~

**~~Message sequence~~**

- ~~User -> ST -> Active (set A); User -> ST -> Backup (set B)~~
- ~~Activation procedure -> Backup (proof of source exclusion)~~
- ~~Backup -> ST (challenge); User -> ST -> Backup (answer from set B)~~

#### ~~Pros~~

- ~~Disclosure of one independently chosen set need not expose the other.~~
- ~~Active rotation need not update backup consent; backup enrollment is already done.~~

#### ~~Cons~~

- ~~Two sets, including a rarely used one, increase confusion and false duress.~~
- ~~Related sets and common ST compromise weaken separation. Backup eligibility
  still requires compromise tracking; wrong-device hints must not reveal answers.~~

### ~~Variant 4C. Enrollment on backup activation~~

**Rejected as drawn.** Its message sequence places an Active Boomlet exchange in
the activation flow, when Boomlet is permanently offline. Variant 4E moves this
exchange to backup creation but leaves coerced enrollment at activation.

~~Introduce a versioned protected `CONSENT_REENROLL_REQUIRED` mode for the backup.
After excluding the old signing authority, require private ST enrollment before
withdrawal. Rotate active consent locally through durable quarantine,
confirmation, and commit. The current backup schema requires a five-element set;
an empty or dummy set cannot encode this mode.~~

**~~Message sequence~~**

- ~~Active -> Backup (protected reenrollment-required mode)~~
- ~~Activation procedure -> Backup (proof of source exclusion)~~
- ~~User -> ST -> Backup (fresh set and two confirmations)~~
- ~~Backup (commit enrollment before withdrawal)~~

#### ~~Pros~~

- ~~Removes shared consent and its synchronization; the user remembers one set.~~
- ~~Backup activation cannot restore a copied exposed answer.~~

#### ~~Cons~~

- ~~Recovery requires private enrollment when the user may be rushed or coerced.
  The device cannot detect an attacker dictating the replacement.~~
- ~~Requires enforceable revocation of a missing active device. Lost history limits
  non-reuse guarantees; enrollment delays backup readiness.~~
- ~~Activation and legacy import must enforce the mode. A local flag does not
  retire an offline device holding the old set and signing share.~~

### Variant 4D. Retire the backup before local rotation

Erase a reachable trustworthy backup or revoke it through every future activation
path. Rotate locally, then provision a replacement backup through a lifecycle
transition that excludes the former target.

**Message sequence**

```text
Retirement procedure -> Active (proof old Backup is excluded)
User -> ST -> Active (fresh set and two confirmations)
Active -> replacement Backup (target-bound protected state)
```

#### Pros

- Avoids paired commit and supports migration from copied consent state.
- Temporarily limits consent storage to the active device.

#### Cons

- Losing the remaining device may leave only fallback spending paths.
- Declaring a missing device retired is insufficient. Erasure cannot retract
  extracted secrets; replacement must not revive old signing authority.

### ~~Variant 4E. Preprovisioned reenrollment for offline activation~~

**Rejected.** A coercer can observe or dictate the fresh safe set during
activation. Source exclusion and two confirmations do not establish private,
uncoerced enrollment.

~~During backup creation, Active provisions Backup with a versioned, target-bound
`CONSENT_REENROLL_REQUIRED` mode instead of a copied safe set. After Active is
permanently offline, Backup verifies independent source-exclusion and current-state
evidence, then enrolls one fresh set through trusted ST before any withdrawal.
If either proof is unavailable, Backup remains inactive.~~

**~~Message sequence~~**

- ~~Active -> Backup (reenrollment-required state during backup creation)~~
- ~~Exclusion mechanism -> Backup (source-exclusion proof after permanent loss)~~
- ~~Recovery mechanism -> Backup (current-state and pending-duty evidence)~~
- ~~User -> ST -> Backup (fresh set and two confirmations)~~
- ~~Backup (commit enrollment; activate only after all gates pass)~~

#### ~~Pros~~

- ~~Activation needs no message from the offline Boomlet and retains one memorized set.~~

#### ~~Cons~~

- ~~Independent source exclusion and current-state recovery remain unsolved; a loss
  report or timeout cannot supply either proof.~~
- ~~Requires a versioned backup schema and private enrollment during recovery,
  when coercion or lack of ST access may prevent completion.~~

## Option 5. Derived sets with a witnessed epoch

Both devices derive consent from a protected high-entropy root and a committed
epoch. A witness records the current epoch; the user privately learns and confirms
each set before commitment.

**Message sequence**

```text
Active -> ST -> User (privately show set derived for new epoch)
User -> ST -> Active (two confirmations)
Active -> Witness (commit epoch); Witness -> Backup (current epoch proof)
Backup (derive set only under the committed epoch)
```

### Pros

- Avoids transferring each replacement set to an offline backup.
- A protected pseudorandom derivation can isolate observed outputs across epochs.

### Cons

- Static-root compromise exposes all covered epochs; advancing the counter cannot
  repair it. Sampling, repeated sets, and protected history need explicit rules.
- Witness records can be stale or conflicting. Replication needs a fault model
  and quorum protocol, with outage and metadata costs.
- User confirmation, commitment, and backup activation still need crash recovery.

## Option 6. Challenge-dependent responses

Replace the reusable answer with a ceremony-specific response. One concrete
candidate privately pre-enrolls independent answers, allocates one durably before
each ceremony, repeats it only for that ceremony's retries, and never reuses a
spent entry. Other candidates compute responses from a protected secret.

**Message sequence**

```text
Active (durably allocate unused answer for this ceremony)
Active -> ST (fresh check); ST -> User (challenge)
User -> ST (answer); ST -> Active (nonce-bound result)
Active (durably consume answer; exact retries use the same allocation)
```

### Pros

- Observing a spent answer need not enable a later ceremony, even without detection.
- Reduces reliance on discretionary rotation after disclosure.

### Cons

- Public challenge-response pairs may reveal a low-entropy root through offline
  guessing. Fresh nonces alone do not prevent this.
- An authenticator that automatically returns safe gives the coercer that path;
  the user still needs a concealed duress choice.
- Consumption, exhaustion, backup rollback, and replenishment add state. Stored
  sequences can be seized; memorization and calculation increase errors.
- Exposure remains within the ceremony while its allocated answer stays valid.

## Option 7. Witnessed encrypted recovery

Enroll independent fresh sets and store randomized, target-bound recovery
envelopes externally. A witness binds the committed epoch to the exact envelope;
Boomletwo imports only the current one during authorized recovery.

**Message sequence**

```text
User -> ST -> Active (fresh set and confirmations)
Active -> Store (target-bound encrypted envelope)
Active -> Witness (epoch and envelope digest)
Witness -> Backup (current epoch proof); Store -> Backup (matching envelope)
```

### Pros

- Permits rotation with an offline backup without one permanent derivation root.
- A ciphertext digest can identify the envelope without exposing a plaintext
  guessing test; storage and epoch authority can be separate.

### Cons

- A compromised recipient key exposes captured envelopes, including future ones
  until that recipient is revoked or replaced.
- Storage deletion, stale records, and equivocation threaten recovery. Encryption
  proves neither freshness nor availability.
- Confirmation, storage, witness commitment, and activation require coordinated
  recovery, service availability, and additional key management.

## ~~Option 8. Consent authority in ST~~

**Rejected as drawn.** Local rotation or replacement ST reenrollment can make a
coercer-chosen safe set current while Boomlet is offline. No independent gate
prevents this change from taking effect on Boomletwo.

~~Move persistent consent and comparison into ST; neither Boomlet stores the set.
ST returns a result bound to each outstanding check and a SAR-encrypted
classification with uniform public behavior. Boomlet requires acknowledgment
of that exact payload. Replacement STs reenroll instead of importing stale
consent.~~

**~~Message sequence~~**

- ~~User -> ST (rotate or reenroll safe set); ST (make it current)~~
- ~~Active -> ST (outstanding nonce, phase, and withdrawal)~~
- ~~User -> ST (answer); ST -> Active (per-check receipt and SAR ciphertext)~~
- ~~Active -> WT -> SAR (authenticated relay requiring a new channel contract)~~
- ~~SAR -> WT -> Active (exact payload acknowledgment)~~

### ~~Pros~~

- ~~One consent authority serves both devices, removing their consent synchronization.~~
- ~~Rotation can remain local and preserve the descriptor.~~

### ~~Cons~~

- ~~Concentrates durable consent in ST. Giving it `doxing_key_for_sar` also grants
  historical rescue-data decryption and `dynamic_update_auth_key`, which can
  authenticate uploads under any device ID. Choose a narrower signaling
  capability or explicitly accept that expanded authority.~~
- ~~Moves backup, rollback, and revocation problems into ST. Old identities and
  pending receipts need cutover rules.~~
- ~~ST ciphertext cannot be inserted into the Boomlet-to-SAR placeholder channel
  unchanged. Define its authenticated relay or replacement channel, replay key,
  and exact acknowledgment coverage without sharing Boomlet channel secrets.~~

## ~~Option 9. Online consent verifier~~

**Rejected as drawn.** A coerced update of the verifier's authoritative record
can make a known safe answer valid for Boomletwo without the original Boomlet.
No independent gate for such updates is specified.

~~ST submits encrypted, authenticated responses to an authoritative service on each
check. The verifier binds its consent epoch and verdict to the exact outstanding
check nonce and phase, producing a uniform receipt and SAR-confidential
classification. Boomlet consumes only that check's result and requires SAR's
acknowledgment; backups cannot substitute cached consent.~~

**~~Message sequence~~**

- ~~User -> ST -> Verifier (rotate safe set); Verifier (commit new epoch)~~
- ~~Active -> ST (outstanding nonce, phase, and withdrawal)~~
- ~~User -> ST (answer); ST -> Verifier (authenticated encrypted response)~~
- ~~Verifier -> Active (per-check verdict and SAR-confidential payload)~~
- ~~Active -> WT -> SAR (payload over a defined authenticated channel)~~
- ~~SAR -> WT -> Active (exact acknowledgment)~~

### ~~Pros~~

- ~~Rotation updates one record and reaches offline backups at their next required
  check without distributing the new set to them.~~
- ~~Protected history can survive local-device loss.~two ph~

### ~~Cons~~

- ~~Adds service dependence, censorship, classification privacy, and database risks.
  A compromised verifier can suppress duress; replication needs explicit trust bounds.~~
- ~~Wrong valid sets must signal duress, not authentication failure. Ordinary OTP
  success receipts expose classification and can permit relay [R3].~~
- ~~Must define rescue authority and pending-verdict cutover. Revocation cannot undo
  completed signing or a delivered rescue signal.~~
- ~~If the verifier originates SAR ciphertext, it needs an authenticated channel
  contract distinct from the current Boomlet-to-SAR placeholder channel.~~

## Option 10. Private randomized cues

Privately show the user a fresh safe button position for each round. All fixed
rounds must match for safe; any valid mismatch means duress, with uniform public
behavior. Bind and erase cue state. This implements Option 6 through a private
visual or tactile channel rather than a memorized sequence.

**Message sequence**

```text
Active -> ST (fresh check and round)
ST -> User (private cue); User -> ST (selection)
ST -> Active (nonce-bound round result); repeat fixed rounds, then erase cue
```

### Pros

- Removes the reusable set and its backup synchronization.
- Recording public inputs need not reveal classification or future private cues.

### Cons

- Assumes the coercer cannot observe, demand, or take over the private cue channel.
  Cameras, physical leakage, and forced demonstrations challenge that assumption.
- Small choice spaces permit safe guesses. More independent rounds reduce guessing
  but increase user error; retries and independence require justification.
- Changes hardware and accessibility needs. Motor behavior may reveal deviation;
  observation studies alone do not establish active-coercion resistance [R4].

## Option 11. Mandatory independent rescue review

Every withdrawal opens a durable SAR case before progress. Independent guardians
review intent; signing requires clearance, while unresolved cases escalate under
a deadline policy. The exposed set or local user alone cannot cancel review.
Define clearance and escalation thresholds, rescue-data access, and enforcement
by both devices, including fallback limits.

**Message sequence**

```text
Active -> WT -> SAR (open withdrawal case)
SAR -> Guardians (review request); Guardians -> SAR (independent decisions)
SAR -> WT -> Active (clearance or escalation under the deadline policy)
Active (sign only after required clearance)
```

### Pros

- A safe local answer cannot suppress all review; rescue consideration can begin
  without a covert signal.
- Distributes judgment beyond the locally coerced user.

### Cons

- Requires available, honest, independently situated guardians. Collusion or
  coerced clearance defeats review.
- Routine review exposes more information; outages can cause false escalation.
  Visible refusal may alert the coercer and increase physical danger.
- Replaces SAR release and concealment rules. A veto does not establish timely
  rescue; guardian replacement and fallback remain attack paths. Adoption must
  replace the incompatible observability requirement in the Boomletwo lifecycle.

## Integration findings and decision gates

These findings apply only to the affected choices. Struck choices retain their
findings for review. OCI-01 has a candidate check-binding rule, pending
implementation evidence. The active gates require a selected design or migration
procedure.

| ID | Choice | Decision or evidence required |
| --- | --- | --- |
| OCI-01, high | ~~8~~, ~~9~~ | If reconsidered, verify per-check nonce, phase, epoch, and payload binding across restart and exact retry. |
| OCI-02, high | ~~8~~ | If reconsidered, decide whether ST may gain rescue decryption and upload authority from `doxing_key_for_sar`. |
| OCI-03, high | ~~1~~ | If reconsidered, specify complete rollover spending gates and destination policy. |
| OCI-04, high | Fixed-set recovery, ~~1~~ | Specify durable quarantine with pending rescue and stalled withdrawals; test crashes and races. |
| OCI-05, high | ~~4C~~, 4D, ~~4E~~ | Version the backup consent mode and enforce source and former-target exclusion; test legacy imports and delayed receipts. |
| OCI-06, medium | ~~8~~, possibly ~~9~~ | If reconsidered, define the classifier-to-SAR channel, replay scope, and exact acknowledgment without weakening Boomlet authentication. |
| OCI-07, high | ~~1~~ after rescue-key compromise | If reconsidered, use a new doxing password and registration, with a transition for old setup obligations. |
| OCI-08, medium | ~~2~~, 11 | Define a versioned acceptance profile. Option 2 fails OC-SR-01; Option 11 conflicts with BW-SR-08. |

Synthetic checks showed that a ceremony-only binding cannot distinguish two
checks in one withdrawal, a rescue-root holder can authenticate uploads and
decrypt retained data, and the current backup schema has no consent-mode field.
No consent implementation or physical device was tested.

## Validation and adoption

- Exercise every durable transition under message loss, crash, replay, rollback,
  device loss, stale backup activation, and attempted duplicate authority.
- Test unauthorized or coerced maintenance, exposed-set reuse, stolen keys,
  guessing from transcripts, exhaustion, and false-safe or false-duress rates.
  Preserve queued rescue; old valid sets must not produce distinctive errors.
- For witnesses, ST authority, and verifiers, test stale or conflicting records,
  substituted payloads, outages, compromised authority, and pending-receipt cutover.
- For sequences and cues, test repeated observation, dictated input, physical
  leakage, retries, exhaustion, accessibility, and user error. For guardians,
  test collusion, coerced clearance, silence, false escalation, and rescue timing.

Broader changes require versioned messages, authority and observability rules,
and migration tests. Retire or revoke old devices capable of bypassing new gates;
if that cannot be enforced, the in-place recovery path is unavailable. Updating
new devices alone leaves old spending authority intact.

Adoption requires an explicit choice, documented lost properties, operator
recovery actions, and reproducible evidence. Tests cover exercised cases;
model checks establish properties only within their stated assumptions.

Update `spec/SPEC.md`, setup and withdrawal documentation, `DESIGN.md`, `README.md`,
`GLOSSARY.md`, security models, an ADR, and operator runbooks for the selected
option. Change PlantUML only for adopted protocol changes; do not regenerate SVGs.

## Review basis

Protocol reasoning uses SPEC Sections 13.3, 16.1–16.6, 18.5, and 19.1. External
sources support the cited technical points, not the proposed designs as a whole.

- [R1] Gray and Lamport, *Consensus on Transaction Commit* (2006).
  DOI `10.1145/1132863.1132867`.
- [R2] Metere and Dong, *Automated Cryptographic Analysis of the Pedersen
  Commitment Scheme* (2017). DOI `10.1007/978-3-319-65127-9_22`.
- [R3] NIST SP 800-63B-4, Sections 3.1.3, 3.2.5, and 3.2.8.
  DOI `10.6028/NIST.SP.800-63B-4`.
- [R4] Bošnjak and Brumen, *Shoulder surfing: From an experimental study to a
  comparative framework* (2019). DOI `10.1016/j.ijhcs.2019.04.003`.
