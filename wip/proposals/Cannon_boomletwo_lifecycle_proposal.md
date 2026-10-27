# Boomletwo lifecycle proposal

| Item | Value |
| --- | --- |
| Status | Draft, not adopted or verified |
| Phase 1 priority | 3 |
| Baseline | Current working-tree [`SPEC.md`](../../spec/SPEC.md), Sections 7.1, 13.10, and 18.5 |
| Related gaps | DG-04, DG-30, DG-39; AR-27, AR-28, AR-80; FM-31 |

## Scope and security boundary

Boomletwo imports a normal-key-authorized, target-bound copy of long-lived
setup state and returns `BackupDone`. The copy includes the logical signing
identity and share, but no withdrawal mystery or live signing session. Import
alone cannot establish that the source has stopped signing. Activation,
revocation, and recovery of later security state remain open, including in the
[WT checkpoint proposal](wt_failover/boomlet_rollover.md).

The goal is at most one active signing device per logical peer and setup while
preserving withdrawal, consent, and rescue obligations. The attacker may control
Niso and transport, isolate a functional source, hide newer state and emitted
outputs, replay receipts, substitute targets, and interrupt writes or messages.
Trusted device code, protected storage, and setup-time Iso and ST remain
assumptions. A copied signing share or a source that ignores retirement needs
independently enforced exclusion. Authorization does not prove source exclusion
or freedom from coercion.

## Proposed lifecycle

Use `EMPTY`, `BACKUP_INACTIVE`, `ACTIVE`, and `RETIRED` as candidate device
states, separate from withdrawal phases. Keep a backup inactive until source
exclusion and current security state are proved. A distinct `REVOKED` state
depends on how revocation constrains an offline device.

### Import

Keep each device's management identity separate from the copied logical peer
identity. Bind the encrypted transfer to the setup, source, designated target,
and attempt, with normal-key authorization and trusted-device review. Reserve
the target before export. Commit inactive import and the original `BackupDone`
receipt together; exact retries return that receipt. An ambiguous import keeps
the backup slot occupied until resolved.

### Planned handover

A reachable, trustworthy source is the initial candidate.

1. Verify identities, authorization, and that no withdrawal, SAR duty, or
   signing handoff is unresolved.
2. Bring the target's protected setup and lifecycle state current; a valid old
   checkpoint alone is insufficient.
3. Durably retire the source and store a target-bound receipt before releasing
   it. Retirement blocks signing, conflicting lifecycle decisions, and further
   secret export; exact receipt retrieval survives restart.
4. Verify the receipt, current state, and consent requirements on the target;
   durably record activation before exercising active authority.

Interruption may leave both devices unable to sign. It must never enable both
or restore the source because the target's acknowledgment was lost. The receipt
format and enforceable retirement mechanism still need specification.

### Missing source and replacement

A missing or compromised source cannot supply trustworthy retirement evidence.
Keep the target inactive until an enforceable exclusion mechanism is selected;
otherwise declare this recovery path unsupported. Candidates are fresh
permission enforced by every signing device or migration through an available
spending path. Timeouts, peer assurances, and service blacklists cannot disable
an isolated device holding a share.

Exclude an old inactive target from every future activation path before
releasing another backup, including when it is unreachable or already has an
activation receipt. Active replacement follows the same source-exclusion rule.
Migration needs valid spending authority and chain confirmation; old signatures
and funds still reachable through the old setup remain relevant.

## Security requirements

These conditions apply to any selected mechanism. All are proposed and pending
validation.

### BW-SR-01 Exclusive active authority

At every reachable point, including interruption, rollback, isolation, and
reconnection, at most one device per peer and setup may sign or make
state-advancing protocol decisions. The gate must constrain devices holding
the copied authority, not just host status.

### BW-SR-02 Inactive backup confinement

Import, `BackupDone`, key possession, and backup authorization grant no active
authority. Inactive operations have distinct signatures. Activation requires
verified source exclusion, current security state, applicable consent, and a
durable target decision before active output.

### BW-SR-03 Bound lifecycle decisions

Authenticate operation, profile, setup, logical peer, physical source and
target, predecessor state, and unique attempt. Define canonical fields, domains,
and bounds. Reject replay, wrong context, and conflicting successors, including
concurrent requests and delayed receipts.

### BW-SR-04 Atomic import and exact recovery

Reserve the target before export and commit inactive import with its
original receipt. Exact retries recover that result without another grant;
lost replies, partial writes, and ambiguous state cannot free the slot or make
a second target eligible.

### BW-SR-05 Current state and obligations

Carry forward replay floors, consent restrictions, withdrawal identity,
commitments, pending SAR duties, signing records, and applicable WT decisions.
A valid old checkpoint cannot prove later duties absent. Recover hidden later
state or block activation; later consent cannot clear pending rescue delivery.

### BW-SR-06 Withdrawal and signing safety

Preserve transaction review, unanimous approval, SAR acknowledgment, digging,
and signing gates. Do not credit old traffic, redraw a mystery within one
withdrawal, reuse a MuSig2 nonce, erase emitted fragments, or restart uncertain
signing as unused. Block live handover until unresolved duties are safely closed
or a recovery procedure is specified.

### BW-SR-07 Consent continuity

Apply the selected [observed-consent decision](Cannon_observed_consent_decision_proposal.md).
A stale backup cannot restore an invalidated answer; required private enrollment
and trusted ST confirmation precede withdrawal. The consent set alone cannot
authorize lifecycle changes, and prompts or receipts cannot reveal or test it.
Any accepted exposure needs an explicit reduced claim.

### BW-SR-08 Confidentiality and duress observability

Transfer private material only in authenticated, target-bound encrypted
envelopes; keep keys, nonces, mysteries, consent answers, and rescue plaintext
out of public records and diagnostics. Preserve SPEC Sections 16.4–16.6 on
placeholder delivery, exact acknowledgment, deadlines, retries, and observable
errors. Lifecycle handling cannot disclose classification or cancel committed
rescue; visible maintenance activity is outside that protocol guarantee.

### BW-SR-09 Durable retirement and revocation

Before releasing activation evidence, durably disable the source's conflicting
authority and retain the exact receipt. Restart, restored storage, and delayed
messages cannot reverse retirement under the stated hardware assumptions.
Permit only named post-retirement operations. Revocation needs an enforcement
point beyond host metadata or a service blacklist.

### BW-SR-10 Missing or compromised source

Treat a reported loss as a possible partition of a functional source. Timeouts,
physical-loss reports, normal-key possession, and peer assurances do not prove
exclusion. Any external permission system must define freshness, conflicting
and cached grants, offline use, and device enforcement. Without exclusion or
current-state evidence, activation stays blocked.

### BW-SR-11 Backup replacement

At most one inactive backup may remain eligible. Exclude the old target before
replacement, including delayed imports, issued receipts, and restored state.
An unresolved import occupies its slot. Replenishment binds the new source,
target, and predecessor; otherwise block or specify setup migration.

### BW-SR-12 Reviewable authorization

Require normal-key authorization and trusted-device review bound to each exact
operation, setup, and device pair. Specify who initiates, approves, cancels,
and resumes after credential loss. Approval cannot substitute for retirement
or waive withdrawal and rescue duties.

### BW-SR-13 Resource and storage bounds

Set byte, nesting, proof, retry, and persistent-storage limits; reserve journal
and receipt capacity before irreversible steps. Invalid input, exact retries,
capacity exhaustion, interrupted writes, and counter overflow cannot change
authority or evict required evidence. Exact retries cannot consume new attempts
or cause unbounded writes. Measure endurance and availability limits.

### BW-SR-14 Device identity and compatibility

Keep physical management identity distinct from the copied peer key and bind
it through import; a copied-key signature alone cannot identify a device.
Reject unsupported profiles. Counters need protected state, not only wire fields.

### BW-SR-15 Emitted outputs and migration

Account for signatures, fragments, commitments, and receipts emitted before
retirement, including concealed outputs. If migrating, bind review to the new
descriptor and define spending authority, confirmation, competing-spend,
reorganization, and later-deposit rules. Track funds still reachable through
the old setup.

### BW-SR-16 Cancellation and completion

Define durable outcomes and safe cancellation at every interruption point.
Restore source use only when no target can activate and inherited duties remain
accounted for. Name the devices, services, storage, delivery, and user actions
needed for completion; missing dependencies may block progress, not safety.
The planned path needs an availability demonstration under those assumptions.

## Required validation evidence

All cases are pending. Results must pin protocol and implementation or model
revisions, assumptions, explored bounds, and remaining limits. WT checkpoint
experiments do not validate this lifecycle. Model durable device state,
in-flight evidence, stale snapshots, and adversarial delivery; test hardware
rollback and storage assumptions separately.

| Case | Exercise and required result | Requirements |
| --- | --- | --- |
| BW-T01 | Isolate a functional source, attempt activation, then reconnect both; preserve exclusive authority. | 01, 02, 09, 10 |
| BW-T02 | Substitute context fields and replay approvals; reject unauthorized transitions. | 03, 12, 14 |
| BW-T03 | Interrupt each import, retirement, and activation write or reply; recover the committed result or block. | 01, 04, 09, 16 |
| BW-T04 | Lose `BackupDone`, retry, conflict imports, and race targets; keep one reservation and successor. | 03, 04, 11 |
| BW-T05 | Hide later duress, WT, consent, or signing state behind a valid old snapshot; recover duties or block. | 05, 07, 15 |
| BW-T06 | Restore pre-retirement state or replay a receipt after replacement; excluded authority stays unusable. | 09, 11, 14 |
| BW-T07 | Try historical-pong credit, redraws, nonce reuse, transaction substitution, and fragment loss; preserve gates. | 05, 06, 15 |
| BW-T08 | Try stale consent and target substitution; inspect records for secret disclosure. | 07, 08, 12, 14 |
| BW-T09 | Compare safe and duress handling under retries, capacity pressure, and failures; preserve acknowledgment observations. | 05, 08, 13 |
| BW-T10 | Exceed input, retry, storage, and counter bounds; preserve authority and required evidence, and measure costs. | 04, 13 |
| BW-T11 | Withhold receipt or acknowledgment and attempt cancellation; preserve authority and conditional completion. | 04, 09, 16 |
| BW-T12 | Replenish and conceal an old signed spend during migration; enforce eligibility and old-setup exposure. | 11, 14, 15 |

An unmet requirement blocks adoption or requires an explicit revision of the
security claim.

## Decisions and integration

- Choose enforceable source exclusion for loss or compromise, with its trust and
  availability limits.
- Specify protected lifecycle state, rollback resistance, receipt recovery,
  current-state evidence, and trusted device review.
- Decide consent handling, backup replenishment, and whether replacement keeps
  the descriptor or creates a new setup.
- Specify active-withdrawal recovery with protocol recovery and the checkpoint
  proposal before permitting handover during withdrawal.

Adoption requires updates to SPEC Sections 7, 9–10, 12–13, 15–18, and 21–22;
wire schemas and vectors; setup and withdrawal contracts and source diagrams;
DESIGN and GLOSSARY; DG-04/DG-39 entries; and an ADR for authority and
compatibility decisions. No exclusion protocol or executable lifecycle result
is yet claimed.
