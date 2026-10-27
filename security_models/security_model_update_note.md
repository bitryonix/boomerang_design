# Security model update note

Tracks adopted design changes awaiting independent security reassessment.
No risk, threat or gap is closed here; scores, statuses and control assessments
need review. Distinguish protocol mitigations, implementation evidence and
operational risks. Adoption, catalogue checks and bounded conformance checks
provide design evidence; hardware durability, rollback resistance, side-channel
equivalence, interoperability and independent assurance remain unproven.

## Dynamic rescue-data lifecycle (2026-09-22)

Sources: [SPEC Sections 10 and 13.1](../spec/SPEC.md) and the
[adopted proposal](../wip/proposals/Cannon_dynamic_rescue_data_lifecycle_proposal.md).
The security model still describes selecting one current dynamic update.

### Changed controls and remaining exposure

- Protected first registration bundles payment receipts, identifier, static
  envelope, `dynamic_update_auth_key` and first upload in `SetupPhoneSarMessage2`;
  `SetupSarPhoneMessage2` returns the signed upload receipt. Atomic commit covers
  the envelope, `PROTOCOL_VERSION`, key, first upload and receipt. Reassess
  transport authentication, partial failures and complete-request retries.
- Authenticated append retains every accepted upload; Phone cannot erase it.
  Retries are idempotent; conflicting bytes under one upload identity are rejected.
  Exact signed receipts prove individual encrypted-upload acceptance, neither
  complete retention nor truthful payloads.
- Per-device sequences expose uploader-assigned first-submission order despite
  delivery delays, not send time, cross-device order or completeness. Random
  upload IDs supply unique identity and prevent preemption solely by sequence
  prediction. Reassess gaps, counter reuse, duplicate-number ambiguity and
  out-of-order acceptance.
- Any `dynamic_update_auth_key` holder, including an old Phone, can append under
  any device ID. If SAR state leaks, the stored key adds an offline password-guess
  verifier. Compromised Phones can falsify or stop uploads; SAR can hide or lose
  history. Authentication establishes neither relevance nor freshness; rescue
  applications must interpret both.
- Enrollment, crash recovery and post-duress history access need deployment
  evidence. SAR is assumed to preserve accepted history through supported storage
  failures. Reassess incomplete restoration, shared-resource contention and
  protocol-visible acknowledgment timing/failure leakage. Storage is modeled as
  unbounded; exhaustion and finite-capacity behavior remain open.

### Review targets

| File | Entries to reassess |
| --- | --- |
| [Risk register](README.md) | Assets, SAR registration, H-03, R-06, R-07, R-22, R-26, DG-11, DG-40 and validation action. |
| [Architecture](architecture.md) | Phone/SAR boundaries, registration/upload flows, SAR storage, classification and `DynamicDoxingData.captured_at`. |
| [Assumptions](assumption_register.md) | AR-45, AR-59, AR-67, AR-81, FM-32: key establishment, concurrent append, lost replies, replacement Phones, crashes, restoration and payload interpretation. |
| [Attack trees](attack_trees.md) | P5 and Phone/SAR compromise: false/stopped uploads, hidden history, incomplete restoration. Tree 4 D, D2, G5: reassess account-record/account-metadata labels against Section 13.1 registration/payment metadata and minimization. |
| [Audit mappings](audit_mappings.md) | T-DATA-02 replacement branch; T-CRYPTO-03, T-PHONE-01, T-INFO-01, T-INFO-08, T-DURESS-04 and capacity threats/controls. Review upload-key rotation advice: Section 13.1 defines no `dynamic_update_auth_key` rotation for an existing `doxing_data_identifier`. |

## Paired consent rotation (2026-10-17)

Sources: [SPEC Section 16.7](../spec/SPEC.md#167-consent-rotation),
[adopted proposal](../wip/proposals/Cannon_paired_consent_rotation_proposal.md),
ADRs [0009](../adr/0009-niso-relayed-consent-rotation.md) and
[0010](../adr/0010-consent-counter-display.md). The security model still claims
no rotation or consent-recovery procedure exists.

### Changed controls and remaining exposure

- Signed, nonce-bound ST approval authorizes rotation through Niso within the
  setup-bound source/inactive-backup pair. Both verify approval and the protected
  counter. Inactive copied-source-key restrictions need implementation evidence.
  Selection and matching confirmation rely on trusted ST local input/encoding.
  Approval supplies less independent ownership authentication than the normal
  key and cannot establish freedom from coercion or privacy. Control of the
  trusted pair and ST can authorize unwanted maintenance or force duress.
- Valid wrong current-set or withdrawal answers set `under_duress`; correct
  later answers preserve it. Set delivery carries the flag; the backup saves both
  before acknowledging the exact request digest. IDLE rotation can defer SAR
  delivery until a later approved withdrawal.
- Monotonic counters, exact-message recovery, bounded persistent retry/status
  budgets and atomic transitions constrain replay and interruption. Reassess
  protected-storage rollback, lost replies, cancellation races, cache retention,
  restoration and exhaustion.
- DIGGING rotation preserves the iteration and prior SAR duties. Fresh input
  precedes the resumption ping; its exact SAR receipt gates later progress,
  signing and export. Final SAR delivery and any paired reset precede clearing.
  Review concurrent WT
  completion, receipt substitution, pong replay, latest reached evidence and
  signing-fragment retention.
- Observing the current answer can suppress that check's signal; continued
  observation can reveal replacements, and identical sets are allowed. The
  counter becomes visible after Boomlet records ST approval. Review mistakes,
  false duress, coerced entry and counter inference.
- Niso censorship or unavailable ST, backup, WT or SAR can stall completion,
  resumption or reset. Backups may retain older consent after an unaccepted
  transfer; activation authority, source exclusion, revocation and
  single-active-device protection remain open.
- Safe/duress equivalence must cover prompts, counts, canonical/framed lengths,
  routing, comparison work, durable writes, fixed deadlines, retries, failures and
  telemetry through cancellation, restart, status recovery and reset, including
  leakage after padding removal.

### Review targets

| File | Entries to reassess |
| --- | --- |
| [Risk register](README.md) | Consent recovery/physical-observation claims; DG-38, DG-39, DG-04, R-12, R-21, R-18, R-13, R-14, R-22, R-25; setup-backup/export/reset phases and validation actions. |
| [Architecture](architecture.md) | No-rotation protocol-binding claim; Niso relay, ST approval/local confirmation, source/inactive-backup keys, persistent consent, post-digging SAR receipt relay, signing gates and cleanup boundaries. |
| [Assumptions](assumption_register.md) | AR-79, AR-31, AR-33, AR-35, AR-36, AR-80, FM-07, FM-30: rollback resistance, atomic state/reply retention, finite storage, counter display, delayed rescue and source-unavailable backup recovery. |
| [Attack trees](attack_trees.md) | Tree 3 observed-answer reuse/trusted-device compromise; interrupted delivery, stale-backup activation, forced duress, classification leakage, continued observation and single-active-device concerns. |
| [Audit mappings](audit_mappings.md) | T-HUM-06 and scenario, T-INFO-10, T-CRYPTO-01, T-CRYPTO-05, T-DURESS-05, T-PROTO-06; recovery, availability and secure-element controls. Approval is not proof of ownership or freedom from coercion. |
