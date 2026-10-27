# WT failover

A setup authorizes an ordered roster of one to five Watchtowers. All five peers
can replace the active Watchtower with another roster member through a reviewed,
signed decision. The replacement prepares its service before activation, and
each Boomlet preserves its withdrawal security state throughout the change.

`WT_FAILOVER_V1` requires a fresh setup and the complete
[wire](wire_contract.md) and [resource](resource_limits.md) profiles. The base
single-WT format cannot enable it. Adoption also requires full protocol
verification, interoperable cryptographic vectors, and hardware measurements.

This document is a normative delta over [SPEC.md](../../../spec/SPEC.md). All base
requirements apply unless a rule below explicitly replaces or extends them.

## High-level flow

1. Bind the ordered WT roster, initial WT and setup checkpoints during setup.
2. When service fails, have all five users review one candidate and freeze local state.
3. Let the candidate validate recovery evidence and discharge outstanding SAR duties.
4. Collect five PREPARE votes and five COMMIT votes before installing the new WT head.
5. Recover an interrupted decision from retained certificates and higher-ballot reports.
6. Preserve or close the active ceremony according to its phase, then resume setup,
   DIGGING, signing or broadcast under the certified head.

## 1. Security boundary

The participants and their base trust roles are defined in SPEC Sections 4 and
5. This profile adds one to five ordered WT identities and requires all five
Boomlets for every switch decision. A one-entry roster provides no failover.

Safety requires one uncloned, non-rollbackable logical peer across WT switches
and device rollover. Completion requires all five peers; candidate preparation
may also require payment, user interaction, safe chain observations and each
fixed SAR. A prepared decision can finish from retained peer evidence without
the candidate or SAR.

Colluding Niso and WT observations still cannot prove real block progress. The
delay guarantee remains blocked without authenticated chain observation; see
[security verification](security_verification.md).

## 2. Base protocol dependency

Setup, withdrawal, duress, replay, failure and signing behavior comes from SPEC
Sections 12 through 18. This profile changes only WT roster binding, authority,
switch decisions, frozen-state recovery, continuation and fragment retention.

## 3. Changes to setup and WT authority

### 3.1 Immutable roster

Setup review includes the exact preference order of one to five `WtId` values.
Each identity contains a WT public key and Tor address. Keys and canonical
identities must be unique. ST displays complete addresses, roster positions,
and fixed public-key fingerprints without ambiguous truncation.

Index zero selects the initial WT. Only that service needs registration and
payment for setup to proceed. Optional dormant-service registration can reduce
later delay, at the cost of payment and metadata disclosure. A switch selects a
different index from the same roster. Changing an entry, its position, its key,
or an address without a setup-bound continuity mechanism requires a new setup
and fund rollover. SAR identities remain fixed.

### 3.2 Active head

Boomlet stores an active WT index and a 32-byte activation head. Define the hash
notation used below as:

```text
H(tag, fields...) = tagged_sha256(
  tag, canonical_encode(PROTOCOL_VERSION, fields...)
)

active_wt_index = 0
active_wt_head = H("Boomerang/wt/genesis", setup_instance_id)
```

A later head is the activation digest certified by five final decision votes.
It binds the exact predecessor, switch attempt, and candidate readiness content.
Returning to a previously used WT creates a different head.

Every state-advancing WT signature covers a tuple containing the active head
and its protocol content. This includes transaction approval, commit
acknowledgment, pong, reached collection, and fragment or broadcast statements.
The expected signer comes from the stored active index. Immutable registration
receipts remain setup facts and are checked separately from current authority.

Active WT channel authentication uses:

```text
canonical_encode(message_type, ceremony_scope_id, active_wt_head)
```

The ceremony scope is the setup ID, withdrawal ID, or approved withdrawal ID
appropriate to that message. Authentication binds message type, traffic
direction, sender, and recipient. The head belongs in authenticated context;
the encrypted envelope need not gain another serialized field. Signed WT
content includes the head explicitly. Protocol version separates these signed
shapes from other profiles.

A receiver rejects a WT message with a different head, context, or signer.
Peer-authored approvals, commits, pings, and signed fragments remain WT-neutral.
A current submission requires the originating Boomlet's authenticated channel,
or exact membership in that Boomlet's signed switch snapshot during reconstruction.
A historical signature alone cannot authorize a current submission.

### 3.3 Setup checkpoints and backup authority

WT switches do not replace the SPEC setup checkpoint chain. Completed checkpoints
remain valid; incomplete WT-dependent prerequisites are repeated through the
candidate before activation preparation.

The designated Boomletwo and one-time backup remain governed by SPEC Section
13.10. The target stays inactive, and public archives cannot authorize it.

The [Boomletwo proposal](boomlet_rollover.md) evaluates self-contained DIGGING
checkpoints encrypted to that same target and authenticated by existing ping
signatures. It has no continuously online ST or backup requirement. The proposed
mutable-state export cannot change private setup policy, select a different
target or include secret MuSig2 nonces. Exact formats remain proposal work.

Backup activation must preserve exclusive authority and state after the supplied
checkpoint. Source exclusion and later-state recovery remain unresolved; a
checkpoint or peer report alone cannot enable signing or WT voting.

### 3.4 Peer WT discovery and self-inclusive pong

Boomletwo can authenticate existing `WtReady` content against the WT head in a
source-signed checkpoint and derive that WT from the immutable roster. One
matching record suffices without new peer votes. It identifies the checkpoint's
WT; current authority still requires a justified decision history.

A candidate pong format carries all five signed pings, including the recipient's
own, while retaining its exact SAR acknowledgment separately. Normal Boomlet
compares its own returned ping with its retained bytes; recovery verifies the
source signature. The other-four freshness predicate is unchanged. WT can sign
one shared pong body, and detached recovery ciphertext avoids duplicating large
private snapshots inside every collection. This candidate requires a complete
version-selected wire definition before adoption.

## 4. Trigger, review, and frozen snapshot

### 4.1 Trigger

After sending a complete valid WT request, Niso tracks the block height of the last
accepted state-specific response. A profile-defined silence threshold permits
proposing a switch. Generic heartbeats, duplicates, malformed traffic, or
responses missing required peer or SAR evidence do not refresh that progress
timer. Explicit user decisions or authenticated terminal errors may also trigger
a proposal. A timeout is advisory and changes no trusted authority.

For a request with a SAR acknowledgment, the host's monotonic elapsed timer must
exceed `SAR_ACK_DELAY + WT_SAR_PROCESSING_BUDGET + ROUND_TRIP_ROUTING_BUDGET +
CHAIN_OBSERVATION_BUDGET` before a silence prompt. Block-height silence alone cannot
establish that elapsed interval; rapid blocks cannot shorten a SAR deadline.
The [resource profile](resource_limits.md) fixes timer admission and retry rules.
Untrusted host time can suggest review but cannot advance the frozen journal.

Block height decreases, contradictory RPC observations, or material disagreement with
the WT produce `CHAIN_VIEW_UNSAFE`. Otherwise the block-height policy supplies the
freshness and milestone checks used throughout the ceremony.

### 4.2 Snapshot and ST review

Resolve any live MuSig2 nonce under SPEC Section 15.13 before freezing; retain a
produced fragment. Also finish and bind any required duress answer to its pending
placeholder. If the corresponding commit or ping cannot be formed, stall. Switch
preparation draws no new challenge and cannot erase an existing duty.

Boomlet constructs a manifest containing:

- setup ID, predecessor head, current WT identity and index, candidate identity
  and index;
- exact local phase, relevant ceremony identifiers, and the conditional recovery
  policy in Section 7;
- durable `digging_initialized` state;
- for DIGGING and later phases, accepted-pong status and block height, inherited
  predecessor block height, current ping digest and sequence, and public reached state;
- a digest of the local resume package;
- an explicit `NO_PLACEHOLDER` record or one exact outstanding placeholder
  commitment, including its approved ID, IV, padded-object digest, and gate
  evidence digest; unresolved state has no encodable absence value;
- retained completed-withdrawal IDs, so an idle peer cannot conceal signing or
  export state relevant to the proposed scope.

`digging_initialized` remains true through reached, signing, and retained export
state. An idle device after fragment export cannot report that the withdrawal
never entered DIGGING. A missing or contradictory state report is unresolved.

The resume package contains the public evidence needed for its phase:

| Local phase | Retained evidence |
| --- | --- |
| Setup | Parameter agreement, completed checkpoint certificate, and pending local setup receipts |
| No active withdrawal | Completed setup evidence and explicit absence of an active ceremony |
| Pre-DIGGING | Reviewed transaction commitment, local authorization state, and exact committed artifacts or their authenticated retention references |
| DIGGING | Accepted approval-set binding, local padded commit, current signed padded ping, exact outstanding placeholder obligations, and accepted public progress summaries |
| Reached or signing | DIGGING evidence, reached collection, and signing-session disposition |
| Fragment collection | Immutable fragment, transaction binding, and retained public progress evidence needed by peers still completing the withdrawal |

Each phase has a fixed canonical layout, exact counts, and a maximum size.
Private withdrawal state stays on Boomlet. Host-provided evidence must match its
retained digest; reconstruction uses the committed bytes.

```text
resume_digest = H("Boomerang/wt/local_resume",
                  phase, scope_kind, scope_id, local_resume_package)
review_commitment = H("Boomerang/wt/switch_review", manifest)
review_message = MessageWithNonce(review_commitment, fresh_32_byte_nonce)
```

`scope_kind` selects setup, withdrawal, or approved withdrawal; `scope_id` is
the corresponding identifier. `MessageWithNonce` carries the commitment and
its fresh review nonce as one signed value.

ST recomputes the manifest commitment and displays the setup, ceremony, both WT
identities and recovery consequence. For a pre-DIGGING peer, the review covers
closure if nobody initialized DIGGING or preservation otherwise. The SPEC ST
authentication and nonce rules apply. Any manifest change requires a fresh review.

Before exporting its signed switch intent, Boomlet stores the exact
intent, manifest, retention commitments, and frozen state. The intent signs the
same nonce-bound review message under a distinct signature domain. Queued WT
messages, ST answers, signing, and ordinary state advancement wait while frozen.
The sole withdrawal-side exception is verification and durable discharge of the
exact frozen placeholder under Section 5.2; it changes neither the signed
snapshot nor the ordinary phase, counters, block heights, or sequences.
Crashes reproduce the same intent.

### 4.3 Common attempt

Peers exchange intents over authenticated paths independent of the old WT.
Every Boomlet verifies all five signatures in setup order, its own exact stored
intent, common setup and predecessor, the same reviewed candidate, compatible
scopes, and package commitments. Local transaction and phase checks remain
mandatory; another peer's advanced state cannot substitute for them.

```text
wt_switch_id = H("Boomerang/wt/switch_id", ordered_five_switch_intents)
```

The five fresh review nonces distinguish the attempt. Only the reviewed candidate
receives its setup and reconstruction evidence. Host-relayed artifacts are usable
only when their exact bytes occur in their originator's committed package.

## 5. Candidate preparation

### 5.1 Registration and service evidence

The candidate receives all five signed setup records, matching parameter
fingerprints, its roster identity, intent material, and required resume packages.
It verifies signatures, identifiers, membership, ordering, package hashes, and
phase prerequisites before retaining the reconstruction input durably.

Incomplete SPEC WT registration or SAR finalization is repeated through the
candidate for the same setup and fixed SAR. Each Boomlet retains and verifies its
own receipts before PREPARE; no active-head endorsement is needed. Valid immutable
registration evidence may be reused. A new head alone creates no new charge.

Candidate channel authentication uses:

```text
canonical_encode("wt_candidate_registration", message_type,
                 setup_instance_id, wt_switch_id, candidate_wt_index)
```

SPEC envelope rules apply. Retransmitted signed padded objects and inner
placeholder envelopes remain byte-identical.

### 5.2 Discharge the frozen placeholder

The SPEC placeholder and replay rules apply. Each peer retains at most one exact
outstanding obligation; a later placeholder cannot replace it.

Before ballot-zero ACTIVATE PREPARE in **every** continuation mode, each Boomlet
must either verify its retained exact SAR acknowledgment or establish from its
frozen journal that no placeholder is outstanding. The latter signs explicit
`NO_PLACEHOLDER` in its intent. Missing bytes, unknown acknowledgment state, a
pending challenge, or conflicting evidence stalls preparation.

The candidate reconstructs the gate-valid padded object from its originator's
committed package and transports the exact placeholder to the setup-bound SAR.
For an initiator commit the original complete approval set, exact WT approval,
and all four matching approval-set attestations are required before routing.
For a non-initiator commit, also retain its verified initiator acknowledgment.
Retained ping evidence binds the initialized withdrawal and current sequence.
The host cannot replace this gate evidence with a claimed phase. Missing gates
stall preparation, including withdrawal closure.

The SPEC SAR checks apply to the response. A candidate envelope may transport
the WT-neutral acknowledgment, and Niso may extract the same response from a
late old-WT packet. Old WT authentication authorizes nothing else. Discharge is
journaled and exact duplicates cause no second write.

The frozen five-record vector determines:

```text
obligation_digest = H("Boomerang/wt/obligations",
  ordered_five_frozen_obligation_records)
```

An ACTIVATE PREPARE attests that the signer has discharged its own listed duty
and checked the common vector against all five intents. A complete prepared
certificate supplies the five closure attestations. No additional Boomlet
signature round or acknowledgment-signature bytes enter the activation digest.
An honest peer's unresolved rescue signal therefore prevents preparation even
with four dishonest cosigners. Health checks and route probes cannot replace
the exact evidence or promise future SAR availability.

### 5.3 Readiness

After registration, payment, routing preparation, and durable reconstruction,
the candidate signs deterministic readiness content:

```text
ready_record = WtReady(scope, candidate_wt_index, recovery_mode,
                       closed_withdrawal_id, closed_approved_id,
                       obligation_digest)
ready_commitment = H("Boomerang/wt/candidate_ready", ready_record)

activation_digest = H("Boomerang/wt/activation",
  setup_instance_id, previous_active_wt_head, wt_switch_id, ready_commitment)
```

The candidate commits to the frozen obligation vector; each Boomlet enforces
its own discharge before PREPARE. Candidate readiness may arrive while an exact
SAR response is in flight, so no extra Boomlet readiness-report round is needed.

The activation certificate carries the valid candidate signature. The head hashes
the content commitment rather than signature bytes. A free timestamp or nonce
cannot vary the activation value within an attempt.

Before its initial activation PREPARE, each Boomlet verifies its own registration
and payment evidence, own discharged obligation, candidate signature, and phase
prerequisites. It independently derives `recovery_mode`, `closure_scope`, and
`obligation_digest` from the verified frozen intents. `closure_scope` names the
exact abandoned withdrawal ID and optional approved ID for closure; otherwise
it is the explicit absent variant.
Those checks are retained for decision recovery. Service expiry can stop an
initial PREPARE; it cannot revoke a completed certificate or add another live
service dependency to an already prepared decision.

## 6. Recoverable activation and cancellation

### 6.1 Scope, ballots, and values

```text
decision_scope = (PROTOCOL_VERSION, setup_instance_id,
                  previous_active_wt_head, wt_switch_id)
cancellation_digest = H("Boomerang/wt/switch_cancel",
  setup_instance_id, previous_active_wt_head, wt_switch_id)
```

The instance has exactly two permitted values, represented by a kind byte and
digest: `ACTIVATE(activation_digest)` and `ABORT(cancellation_digest)`.
`decision_ballot` is a durable u64 within the pending instance, initially zero.
It leaves the candidate, reviewed snapshot, and service head unchanged. Ballots
never wrap. The [resource limits](resource_limits.md) bound successor admission,
retries, total admitted ballots, and durable transactions.

PREPARE is provisional. Only five matching COMMIT signatures decide a value.
Decision COMMIT has a separate domain and meaning from withdrawal `TxCommit`.
It cannot substitute for transaction approval, a commit set, SAR acknowledgment,
or pong.

### 6.2 Certificates

| Certificate | Required evidence |
| --- | --- |
| PreparedCertificate | Five PREPARE signatures over identical scope, ballot, and value |
| RecoveryCertificate | Five RECOVER reports for the same target ballot, each carrying its stated prepared certificate or explicit NONE |
| DecisionCertificate | Five COMMIT signatures over identical scope, ballot, and value, with committed intent material and candidate readiness evidence for ACTIVATE |

Every collection contains exactly one signature from each setup peer in setup
order. Recipients verify domains, versions, scope, counts, identities, signatures,
their own stored intent or recovery report, and the authorized decision value.

RECOVER signs the scope, target ballot, and its reported prepared vote content. A bounded sidecar supplies the five PREPARE signatures.
The reported certificate's ballot is strictly below the target. NONE has an
explicit absent tag. Certificate commitments cover scope, ballot, and value,
independently of signature encodings.

A final certificate need not repeat PREPARE signatures: every honest COMMIT
signer first verifies and durably retains them. A recovery collection carries at
most five prepared certificates. It does not recursively embed ballot history.
Each five-signature prepared certificate includes an honest witness that checked
its ballot justification before signing.

### 6.3 Successful activation

1. Collect the five reviewed intents and complete candidate preparation,
   including each signer's own exact placeholder discharge. For closure, every
   manifest must establish that DIGGING was never initialized and explicitly
   authorize abandonment of the same identifiers.
2. In ballot zero, each Boomlet persists and exports PREPARE for ACTIVATE.
   It signs at most one PREPARE value per ballot.
3. After receiving five matching PREPARE votes for its current ballot and its
   own prepared value, it verifies and durably retains that certificate as its
   highest locally accepted prepared certificate.
4. It then persists and exports COMMIT for the same scope, ballot, and value.
   COMMIT requires the current ballot, its matching PREPARE, and the complete
   prepared certificate. Another live candidate or SAR response is unnecessary.
5. Any peer assembles and distributes the final DecisionCertificate. Each Boomlet
   installs the new head and roster index before accepting candidate
   traffic or applying the continuation in Sections 7 through 9.

Votes and certificates are durably disseminated among peers. The candidate cannot
be their sole holder. Partial final-certificate delivery leaves remaining peers
frozen until they verify the same final decision. Retransmission is idempotent.

### 6.4 Recovery

A trusted recovery request admits the next ballot within the same frozen
instance, subject to Section 10 and the resource profile. A host timeout alone
cannot persist a promise. The original ST review authorizes finishing that
reviewed candidate or canceling the attempt. A bounded automatic recovery allowance needs no second
ST approval; extending that allowance requires trusted local admission.

1. Before exporting RECOVER, Boomlet persists a higher promised ballot
   and its exact report containing the highest prepared certificate locally
   accepted under Section 6.3, or NONE. That write forbids creating lower-ballot
   PREPARE or COMMIT votes.
2. Reports are immutable for their target ballot. Late lower-ballot certificates
   cannot rewrite them or authorize old COMMIT votes. Older certificates in
   another peer's report can justify a recovery value without reopening their
   original ballot. Retries reproduce the stored report.
3. Any peer or relay collects all five reports. Every Boomlet verifies the full
   collection, its own exact report, and target equal to its promised ballot.
4. If any valid prepared certificate is reported, select the value of the one
   with the highest ballot. Equal-highest certificates must agree on value.
   If every report is NONE, select ABORT. Each Boomlet computes the result itself.
5. Run PREPARE and COMMIT for the selected value in the target ballot. If delivery
   splits or a higher ballot intervenes, repeat recovery with retained evidence.

A signed claim that a peer prepared is insufficient. The report must contain the
actual five-signature certificate. Individual PREPARE votes do not force the
recovery result. No discretionary coordinator choice or coordinator signature
is needed to select it.

When recovery selects ACTIVATE, its certificate proves that all five peers
already checked their local prerequisites, or carried forward a certificate
establishing those checks. Recovery PREPARE and COMMIT use retained evidence even
if the candidate or SAR is unavailable. After installing a failed candidate,
peers can review another roster WT and switch from that new head. Ordinary
withdrawal progress still requires its service and chain checks.

When recovery selects ABORT, candidate cooperation is unnecessary. Boomlet
persists the final cancellation tombstone before unlocking the matching local
operation under the unchanged predecessor. It preserves DIGGING, pending duress
checks, outstanding placeholders, inherited spacing floors, and replay memory.
Verified obligation discharges survive ABORT; rollback of the freeze cannot
restore an acknowledged obligation or undo SAR activation.
A new attempt needs fresh ST reviews and nonces.

A valid final DecisionCertificate takes priority over recovery for the matching
unresolved instance, even when its ballot is below the local promise. Promises
restrict new votes; they do not prevent learning a completed decision. A decided
peer returns its final certificate instead of another RECOVER report. An old
certificate cannot roll back a later installed head.

### 6.5 Pending intents and durable state

An incomplete local intent can be withdrawn through fresh nonce-bound ST
cancellation approval or an equivalent trusted local user action, only before
its first PREPARE **and before its first RECOVER report**. Once it has reported,
remain frozen until a final decision. Archive its exact identity and withdrawn
tombstone. Host replay cannot recreate it as pending.

If peers later supply a complete set containing that withdrawn intent, its
originator may help close it through RECOVER and ABORT votes only. It reports
NONE because it never prepared that instance. A valid activation prepared
certificate cannot contain that honest peer's nonexistent PREPARE. Closing a
stale instance changes neither a different current operation nor its snapshot.

After PREPARE export, remain in the same instance until a final decision. Each
ballot permits at most one value per voting stage. A higher promise excludes
lower-ballot votes, and a new switch ID at the predecessor requires final ABORT or a locally
withdrawn intent that emitted neither PREPARE nor RECOVER. Admit at most one
live instance. Do not sign multiple five-intent combinations containing the same
local intent; persist the first accepted switch ID before any RECOVER or vote.
A withdrawn intent may close adversarial combinations only with ABORT. Process
one such closure at a time after any live instance finishes; each consumes a
trusted decision-instance reservation. These closures cannot unlock or modify
another operation. Local tombstone matching precedes cryptographic processing.

Persist the promised ballot, highest complete prepared certificate, exact current
report and own votes, prerequisite commitments, and final decision before their
corresponding outputs. Keep the highest certificate's five signatures durably
available on Boomlet; a host-held digest alone cannot support recovery. Replacing
that certificate with a higher valid one is atomic.

A crash before export can retain a vote or promise nobody received. Exact replay
recovers it. A crash after export cannot forget it. Retain final certificates or
equivalent authenticated recoverable evidence for peer catch-up. Storage failure,
unresolved writes, missing evidence, or ballot exhaustion stalls safely.

### 6.6 Safety and completion

A prepared value controls recovery only through its complete certificate. A
COMMIT signer retains that certificate, and every recovery collection includes
the honest signer's report or learns its final decision. Higher promises prevent
old votes, so authentic signatures, collision-resistant commitments, correct
validation, and one durable honest identity exclude conflicting final values.
The [verification report](security_verification.md) gives the full argument.

Completion still requires all five peers, retained evidence, finite resources,
and eventual agreement on one ballot. Different intent locks at honest peers can
prevent either combination from collecting five reports. Preserve those locks
and use fallback; unilateral rebinding would permit conflicting heads.

## 7. Continuation during setup and before DIGGING

### 7.1 Interrupted setup and an idle setup

| Failure stage | Continuation after ACTIVATE |
| --- | --- |
| Before parameter agreement | Finish a consistent reviewed setup agreement before deriving its roster-bound genesis and using failover |
| `PARAMETERS_AGREED` | Verify candidate registration and local receipt, then complete the matching WT-ready checkpoint exchange |
| `WT_READY` | Preserve completed checkpoints, verify retained candidate-path SAR finalization evidence, and complete the SAR-ready checkpoint exchange |
| `SAR_READY` | Preserve completed checkpoints; finish designated encrypted backup, ordinary Iso checks and `BackupDone` |
| `BACKUP_READY` or partial final delivery | Verify and relay matching completed checkpoint evidence; finish each local completion guard |
| Setup complete, no withdrawal active | Install the head and use it for the next withdrawal |

Partial checkpoint delivery is resolved from exact signed evidence. A peer's
reported phase alone cannot complete another peer's checkpoint. Pending
WT-specific receipts are checked against the candidate whenever their local
prerequisite must be repeated. A final ABORT resumes the matching frozen setup
operation under its predecessor head.

### 7.2 Recovery mode

The five frozen manifests select a deterministic policy:

```text
all five establish no active withdrawal:
    change service only
all five establish compatible withdrawal state with DIGGING never initialized:
    CLOSE_PRE_DIGGING_WITHDRAWAL
any initialized DIGGING and all scopes can be reconciled:
    PRESERVE_WITHDRAWAL
otherwise:
    stall
```

Incomplete authorization reports bind the withdrawal under discussion and each
peer's actual participation. A completed archive matching another peer's active
scope is reconciled by supplying the exact full signed transaction before freezing. That peer verifies
its completion and local SAR duties; a phase claim cannot substitute for the
transaction. An unmatched closed scope stalls review. A missing approved ID has
a phase-specific meaning; it is not permission to erase unexplained later state. A false initialized report
can stall a restart. False negative reports cannot reset an honest peer's delay.

### 7.3 Close a pre-DIGGING withdrawal

Before initial PREPARE, every signed intent explicitly authorizes the exact
closure scope and all five snapshots establish `digging_initialized == false`.
Section 5.2 discharges each frozen obligation, including a padded commit that
the predecessor WT accepted but never forwarded. The readiness commitment and
every activation vote cover the common closure scope and obligation vector.
An unresolved or missing scope, prepared placeholder, or signing archive stalls
closure. If any peer has initialized DIGGING, preserve that withdrawal.

After final ACTIVATE, tombstone the abandoned withdrawal ID and any
approved ID, invalidate its pending transaction and duress-review nonces, and
retire its local active state before unlocking. Any replacement follows a fresh
SPEC withdrawal; switch approval does not approve its transaction. Historical
approval-set continuation is outside this profile.

A final ABORT keeps the original operation and its still-valid local reviews.
Already completed rescue activation is irreversible. Each honest peer has
verified all its committed obligations before any activation certificate can be
prepared. Delayed duplicate responses cannot resurrect a closed withdrawal.

Do not evict a tombstone while its ID could authorize an action. Exhaustion
stalls new attempts.

### 7.4 Mixed DIGGING entry

If any peer has initialized DIGGING, all initialized peers retain their approved
transaction, both withdrawal IDs, mystery, counter, block heights, sequences, reached
state, and duress obligations. A lagging peer enters DIGGING exactly once after:

1. Its own durable state establishes acceptance of the same approval set and
   creation of its exact commit through the SPEC initiator-first gates.
2. The candidate reconstructs the five exact signed padded commits from the
   originating peers' committed packages, verifies signatures and approved ID,
   and routes their exact placeholders to the fixed SARs.
3. The peer verifies the complete `COMMIT_SET` handoff, its own exact SAR
   acknowledgment, final activation certificate, and a new-head WT commit response.
4. Local descriptor, milestone, chain-safety, and phase conditions hold.

`COMMIT_SET` is the only historical phase-advancing package in this profile.
Its historical timestamps describe the committed snapshot. The exception is
limited to elapsed block age of (1) the five exact `TxCommit.event_block_height`
values and (2) the retained predecessor initiator commit acknowledgment and WT
approval used only to verify their already-satisfied gate. These objects must
be byte-identical to the committed packages and locally recorded authorization.

No approval, duress answer, or new commit may be created through this exception.
All SPEC checks remain mandatory except the two listed age checks. In particular,
future block-height, ping, pong, ST, signing-session, service, chain-safety and exact
SAR-envelope checks receive no exception. Missing evidence stalls.

## 8. DIGGING continuation

### 8.1 Outstanding obligations and recovery ping

Initial activation preparation discharges the frozen placeholder under Section
5.2. Installation retains that discharge, including after candidate loss and
decision recovery. An already acknowledged placeholder requires no replay.
Recovery pings create the next ordinary SAR obligation under the installed head.

A required duress challenge from an accepted ordinary pong must finish before
creating another ping. Preserve these challenges and outstanding placeholder
records through cancellation and repeated switches. A newer safe placeholder
cannot replace an unacknowledged duress placeholder.

Each initialized Boomlet then creates at most one recovery ping per new head
without receiving another pong first. It uses the next unused sequence, retained
block height and reached flag, and fresh placeholder encryption over current local
plaintext. Persist the signed padded bytes and sequence update before export;
a crash retransmits that same ping. Waiting for prior acknowledgments prevents
repeated switches from accumulating outstanding recovery placeholders.

Recovery-ping creation changes neither mystery, counter, nor local block height and
draws no new duress challenge. A newly initialized lagging peer sends its normal
initial ping. Every other peer's recovery ping is newer than anything it emitted
before its snapshot froze, including after partial old-pong delivery.

### 8.2 pong spacing and validation

The five manifests report accepted-pong status, last accepted event block height, and
any inherited predecessor block height. Each Boomlet checks its own report against
durable state and derives the maximum reported predecessor block height `F`.

```text
first_new_pong.event_block_height >= F + MIN_PONG_SPACING_BLOCKS
```

`F` is a predecessor block height, not a next-eligible block height; applying recovery
repeatedly must not compound the same wait. Preserve it across a switch with no
intervening accepted pong and across cancellation. The first-pong exemption
applies only when every report says no accepted pong, no inherited floor exists,
and local state agrees. Unknown history or unsafe block heights stalls. A malicious
report cannot lower an honest peer's stored floor; inflated block heights can cause
denial of service and remain subject to chain checks.

After enforcing this floor, validate and apply pongs under SPEC Sections
15.8 through 15.10, with the installed head and exact outstanding SAR
acknowledgment. Activation and recovery-ping creation never increment counters.
A valid increment accepted before the freeze remains; peers that missed it
invent none. Old-head pongs cannot advance state.

### 8.3 Reached evidence

The candidate obtains five fresh post-activation reached pings, discharges their
placeholders, and distributes a current-head SPEC reached collection with each
recipient's exact acknowledgment. This also applies when all peers had reached
before the switch.

## 9. Signing, fragments, and broadcast

SPEC Sections 15.12 and 15.13 govern hydration and signing. Before freezing,
finish the exact live MuSig2 session or erase its unused nonce and record the
abort. Retain any completed partial signature as an immutable fragment.

After activation, peers still needing signatures verify current reached evidence
and the PSBT before signing. Peers with completed fragments retain them and the
public progress state needed by unfinished peers. Retain pending duress state
and placeholder obligations until those duties finish. Export alone cannot
turn a signing withdrawal into a restartable pre-DIGGING attempt.

Boomlet retains its exact fragment before export, together with the
approved IDs, transaction commitment, initialized flag, signing disposition,
current ping and replay state, and pending security duties. Niso stores a copy
and forwards a Boomlet-authenticated current-head submission. A crash after
export retransmits the same fragment; it never authorizes another signing pass.

The active WT durably stores the exact fragment before signing a head-bound
`FragmentReceipt` naming setup, approved ID, origin peer, and fragment digest.
The originating Boomlet checks the receipt against its retained bytes. Receipt
acceptance records dissemination only. Because Niso and WT may both be malicious,
neither their receipt nor a broadcast claim permits Boomlet to erase its copy.

Completion requires the SPEC transaction checks and the peer's latest SAR duty.
The head-bound `CompletionReceipt` commits to the exact transaction and ordered
fragment digests but records dissemination only. The verified transaction and
local discharge authorize `COMPLETED_RETAINED`; a pending challenge or placeholder
blocks it. Exact receipt replay is idempotent.

At `COMPLETED_RETAINED`, erase consumed signing secrets and private mystery and
counter, invalidate reviews, and retain a durable closed-withdrawal record plus
the exact local fragment. The archived `digging_initialized` flag stays true.
An initialized peer that lacks the final transaction may use an authenticated
holder's exact complete transaction to finish its ordinary validation, even if
its installed head differs; transaction verification is WT-neutral. A closed
peer supplies its immutable fragment and complete transaction evidence instead
of creating a new ping. If any honest peer still needs to sign, no valid complete
transaction can exist, so every honest exported peer still retains the public
state needed for fresh reached pings.

Four archive slots are reserved before admitting withdrawals. Retained local
fragments survive later withdrawals and WT changes until permanent retirement
of the setup identity, or stay indefinitely when retirement cannot be established.
Public copies at Niso, WT, and another peer improve availability; the honest
originator's own retained bytes are the recovery anchor. Full archive slots
stall new withdrawals while leaving replay and pending recovery usable. Setup
retirement disables every protocol operation for that identity
before erasing archives, and requires completed local rescue duties and trusted
approval of permanent setup abandonment with the verified rollover or fallback
spend. Retirement forbids reuse of that identity and descriptor for new funding. It cannot be inferred from a WT broadcast claim.

A fully signed retained transaction can be verified and rebroadcast with the
same `tx_id` by its holder. A duplicate broadcast does not authorize different
outputs or another signing attempt. WT service-authority changes still follow
the reviewed decision procedure.

## 10. Domains, storage, and workload

### 10.1 Signature domains

The [wire contract](wire_contract.md) defines the profile's domains, contexts,
records and bounds. SPEC decoding and cross-profile rejection rules apply.

### 10.2 Retention and validation

Boomlet keeps the roster, active head and index, frozen local manifest, exact
exported control statements, promised ballot, highest prepared certificate,
final decision, recovery-ping replay state, required ceremony secrets, and
bounded tombstones. Persistent writes precede corresponding outputs.

Niso retains public packages, vote collections, setup evidence, and fragments.
Candidate WT retains reconstruction inputs, route records, certificates, and
replay state. Their data remain untrusted and require verification. Stream
bounded artifacts against stored digests rather than loading five full transcripts
into Boomlet. Digests protect integrity; durable holders supply availability.

Niso filters invalid input, but Boomlet independently checks every transition.
Protected caches apply only to identical artifacts in an unchanged frozen context.

### 10.3 Workload

| Operation | Added control work |
| --- | --- |
| Successful switch | Three Boomlet signatures per peer, fifteen total: intent, PREPARE, COMMIT |
| ST review | One signature and one Boomlet verification per peer |
| Candidate readiness | One signed deterministic commitment |
| Frozen obligation discharge | At most one exact SAR acknowledgment verification per Boomlet; reuse an already verified acknowledgment |
| Recovery ballot | Up to one RECOVER, PREPARE, and COMMIT signature per peer, plus recovery-certificate verification |
| DIGGING continuation | Up to five fresh padded pings and SAR exchanges, plus unresolved placeholder replay |
| Pre-DIGGING closure | Existing intent and PREPARE carry closure authorization and discharge attestation |
| New withdrawal after closure | Fresh transaction reviews, approvals, and commits |
| Completion | Stream final transaction once and retain own immutable fragment; a WT receipt records dissemination |

Each padded ping has two Boomlet signatures. Certificate verification, hashing,
encryption, channel operations, interface traffic, and durable writes add further
cost. Bound and measure complete workloads, maximum package sizes, journal bytes,
memory, latency, and write endurance on the intended hardware. Concrete admission
limits, fixed storage reservations, and recovery allowances are specified in
[resource_limits.md](resource_limits.md). Duplicate control messages use protected
cached results. Prepared sidecars with identical content are verified once per
frozen instance, with no recursive history or duplicate stored certificates.

## 11. Companion sequence diagrams

Arrows show logical authenticated messages. Niso relays are abbreviated where
they do not affect the ordering being illustrated. A grouped peer lifeline denotes
independent operations by the named peers, with five distinct signatures and
local prerequisite checks wherever a five-peer collection is required. Diagram
references abbreviate the common exchange defined in Sections 4 through 6.

| Diagram | Coverage |
| --- | --- |
| [01_setup_changes.puml](01_setup_changes.puml) | Roster review, genesis binding, initial registration, SAR finalization, and backup additions |
| [02_common_switch.puml](02_common_switch.puml) | Review, freeze, candidate preparation, obligation discharge, PREPARE, COMMIT, and head installation |
| [03_setup_failover.puml](03_setup_failover.puml) | Interrupted setup checkpoints and an idle completed setup |
| [04_predigging_failover.puml](04_predigging_failover.puml) | Closure before any mystery initialization and selection of preservation for mixed states |
| [05_mixed_digging_failover.puml](05_mixed_digging_failover.puml) | Exact commit handoff to lagging peers while initialized peers retain their delay |
| [06_digging_failover.puml](06_digging_failover.puml) | Partial pong delivery, outstanding placeholders, recovery pings, spacing, and reached evidence |
| [07_decision_recovery.puml](07_decision_recovery.puml) | Cancellation, higher promises, certificate selection, and candidate failure during a decision |
| [08_signing_and_broadcast_failover.puml](08_signing_and_broadcast_failover.puml) | Ready-to-sign, live nonce handling, retained fragments, aggregation, and broadcast |

## 12. Validation and adoption

The [verification report](security_verification.md) records model bounds,
results, negative controls, and the open real-chain observation blocker. The
[issue register](security_issues.md) maps findings to controls, and
[security requirements](security_requirements.md) define adoption integration.

Deployment still requires complete wire and cryptographic vectors, integrated
state-machine verification, independently reviewed firmware, power-loss and
anti-rollback evidence, and hardware measurements. Bounded model success is not
production approval.

The underlying prepare, commit, and recovery pattern is informed by
[Castro and Liskov's authenticated agreement construction](https://static.usenix.org/publications/library/proceedings/osdi99/full_papers/castro/castro_html/node4.html).
The five-of-five certificate rule and its availability assumptions are specific
to this proposal. MuSig2 nonce and session requirements follow
[BIP 327](https://bips.dev/327/).
