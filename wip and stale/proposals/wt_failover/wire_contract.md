# WT failover wire contract

`WT_FAILOVER_V1` is a proposed complete profile with `PROTOCOL_VERSION` encoded
as the ASCII text `boomerang-wt-1`. The base `wip/wire_catalog.json` describes
the single-WT profile. Its generic signature and envelope wrappers alone are
insufficient to enable this profile. Version selection precedes key derivation
and setup hashing and persists in the protected setup state. There is no
in-place negotiation or fallback to a different version after setup.

Canonical tags, field IDs, fixed-byte types, ordered lists, record headers,
BIP340 signatures, CBC-CMAC envelopes, and directional key derivation follow
SPEC Sections 8 and 9. Fields below have ascending IDs starting at 1, are all
required, and use record version 1. A contextual `value` always resolves to the
one listed type. Reject unknown fields, domains, enum values, version changes,
wrong ordering, counts, duplicate identities, noncanonical absent values, and
trailing bytes. Apply [resource_limits.md](resource_limits.md) before allocating
or verifying. Domain and profile version remain inputs to `sign_message`.

## Control records

The supplemental [wire catalog](wire_catalog.json) records the following exact
field order and types as an extension to the base structural catalog.
[Canonical control examples](canonical_vectors.json) record representative
encodings and signature preimages.

| ID | Record | Fields in order |
| --- | --- | --- |
| 30 | `WtHeadContent` | `active_wt_head: bytes32`, `content: value` |
| 31 | `WtScope` | `setup_instance_id: bytes32`, `previous_head: bytes32`, `switch_id: bytes32` |
| 32 | `WtOptionalId` | `present: bool`, `id: bytes32` |
| 33 | `WtObligation` | `present: bool`, `approved_id: bytes32`, `iv: bytes16`, `envelope_digest: bytes32`, `padded_digest: bytes32`, `gate_digest: bytes32` |
| 34 | `WtManifest` | `setup_id: bytes32`, `previous_head: bytes32`, `previous_index: u8`, `candidate_index: u8`, `phase: u8`, `scope_kind: u8`, `scope_id: bytes32`, `withdrawal_id: WtOptionalId`, `approved_id: WtOptionalId`, `initialized: bool`, `accepted_pong: bool`, `pong_height: u32`, `inherited_floor: WtOptionalHeight`, `ping_digest: WtOptionalId`, `ping_sequence: u64`, `reached: bool`, `resume_digest: bytes32`, `obligation: WtObligation`, `closed_ids: list<bytes32>` |
| 35 | `WtOptionalHeight` | `present: bool`, `height: u32` |
| 36 | `WtReady` | `scope: WtScope`, `candidate_index: u8`, `mode: u8`, `closed_withdrawal: WtOptionalId`, `closed_approved: WtOptionalId`, `obligation_digest: bytes32` |
| 37 | `WtVote` | `scope: WtScope`, `ballot: u64`, `value_kind: u8`, `value_digest: bytes32` |
| 38 | `WtPrepared` | `vote: WtVote`, `signatures: list<bytes64>` |
| 39 | `WtRecover` | `scope: WtScope`, `target_ballot: u64`, `has_prepared: bool`, `prepared_ballot: u64`, `prepared_kind: u8`, `prepared_digest: bytes32` |
| 40 | `WtFragmentReceipt` | `setup_id: bytes32`, `approved_id: bytes32`, `origin_peer: u8`, `fragment_digest: bytes32` |
| 41 | `WtCompletionReceipt` | `setup_id: bytes32`, `approved_id: bytes32`, `tx_id: bytes32`, `transaction_digest: bytes32`, `fragment_digests: list<bytes32>` |
| 42 | `WtPublicArchive` | `setup_id: bytes32`, `params: BoomerangParams`, `checkpoint: bytes32`, `public_evidence: bytes` |
| 43 | `WtResumeEntry` | `artifact_kind: u8`, `payload: bytes` |

All absent variants have `present=false` and every remaining byte or scalar
zero. A present ID must be nonzero. `WtObligation.present=false` means the
signer's frozen journal establishes `NO_PLACEHOLDER`; unknown state is rejected
before encoding. An acknowledged prior envelope is included as evidence in the
resume package where its phase needs it. `closed_ids` has at most four distinct
withdrawal IDs, ordered by bytes. `fragment_digests` has exactly five entries in
setup peer order. `WtPrepared.signatures` has exactly five BIP340 signatures in
setup peer order. All identity indices are zero-based and checked against the
stored setup. An initial phase uses zero block heights and sequence, absent ping and
inherited floor, and false reached and accepted-pong flags. The phase determines
whether initialized, identifiers, and retained evidence are mandatory.

| Enum | Values |
| --- | --- |
| Phase | 0 setup, 1 idle, 2 reviewing, 3 approved, 4 committed, 5 digging, 6 reached, 7 signing, 8 fragment retained, 9 completed retained |
| Scope kind | 0 setup, 1 withdrawal, 2 approved withdrawal |
| Recovery mode | 0 service only, 1 close pre-DIGGING withdrawal, 2 preserve withdrawal |
| Decision value kind | 0 ACTIVATE, 1 ABORT |

`WtManifest` uses roster indices to refer to exact setup-bound identities. ST
receives the roster and reconstructs both identities before display. Its review
signs `MessageWithNonce<bytes32>` containing the hash of the canonical manifest.
The Boomlet intent signs the identical nonce-bound content under its own domain.
A five-intent collection consists of exactly five such signed messages and five
manifests in peer order. `wt_switch_id` hashes the five signed intents. Each
manifest's public package must match its resume commitment.

`WtReady` is the canonical readiness content in README Section 5.3: candidate
identity is obtained from its index and `closure_scope` is the pair of optional
closed IDs. The ready commitment hashes this record. Activation hashes the
scope and ready commitment. Cancellation hashes the scope. Every Boomlet derives
the single permitted activation and cancellation values locally. A candidate
signature is mandatory evidence but its signature bytes cannot vary these values.

`WtPrepared` stores one common vote and five raw signatures, each verified over
that same canonical `WtVote` under the PREPARE domain with the indexed identity.
This avoids five repetitions of identical signed content. Its commitment hashes
the common vote; it never hashes signature encodings. A final certificate uses
the same common vote and exactly five signatures under COMMIT, with five intents,
manifest commitments, and candidate-signed readiness attached for ACTIVATE.
Stored local commitments can satisfy already verified attachments. ABORT has
no candidate-readiness dependency. A recovery collection has exactly five
`SignedMessage<WtRecover>` objects and up to five distinct prepared sidecars,
matched by common vote content. NONE zeros its prepared ballot, kind and digest.
A present prepared sidecar has ballot strictly below target and the same scope.
`prepared_digest` equals the sidecar vote's `value_digest`; scope, prepared
ballot and prepared kind reconstruct the complete reported vote. There is no
recursive certificate or history field.

The exact collection types are:

- intent collection: `tuple<list<SignedMessage<MessageWithNonce<bytes32>>>, list<WtManifest>>`, both lists exactly five in peer order;
- recovery collection: `tuple<list<SignedMessage<WtRecover>>, list<WtPrepared>>`, five reports and at most five distinct matched sidecars;
- final ABORT: `WtPrepared` verified under the COMMIT domain, with its locally locked intent collection;
- final ACTIVATE: `tuple<WtPrepared, list<SignedMessage<MessageWithNonce<bytes32>>>, list<WtManifest>, WtReady, SignedMessage<bytes32>>`, with exactly five intents and manifests, COMMIT signatures, and the candidate's signed ready commitment.

Cached local evidence can avoid repeat validation; transmitted objects still
use these exact types. A prepared object in a final-decision context must have
valid COMMIT signatures. PREPARE signatures cannot satisfy that context.

Signature domains are `Boomerang/wt/switch_review`, `switch_intent`,
`candidate_ready`, `decision_prepare`, `decision_commit`, `decision_recover`,
`pending_intent_cancel_review`, `fragment_receipt`, and `completion_receipt`,
with the `Boomerang/wt/` prefix applied to each suffix. Content types respectively
are `MessageWithNonce<bytes32>`, `MessageWithNonce<bytes32>`, `bytes32` (the ready
commitment), `WtVote`, `WtVote`, `WtRecover`, `MessageWithNonce<bytes32>` (the
locally stored intent digest), `WtHeadContent<WtFragmentReceipt>`, and
`WtHeadContent<WtCompletionReceipt>`. There is no SAR probe signature domain.

## WT authority and transport

Every WT-signed state transition uses `SignedMessage<WtHeadContent<T>>`. The
receiver first selects the exact domain and `T` below, then verifies the stored
active index, head, scope, signature, and local state. A bare `T`, an old head
from the same returning WT key, or a candidate signature in this path is invalid.
There is no generic "unwrap and accept" handler.

| WT operation | Exact `T` inside `WtHeadContent` |
| --- | --- |
| SAR finalization forwarding statement | `WtSarSetupResponse` |
| Transaction approval | `WtTxApproval` |
| Acknowledgment of initiator or non-initiator commit | `SignedMessage<TxCommit>` |
| pong | `Pong` |
| Reached collection | Exactly five `SignedMessage<Ping>` objects in setup order |
| Fragment receipt | `WtFragmentReceipt` |
| Completion or broadcast receipt | `WtCompletionReceipt` |

The setup response uses `Boomerang/setup/wt_sar_response`. Withdrawal objects
use their corresponding ordinary withdrawal domains; reached
collections use `Boomerang/withdrawal/reached_collection`. WT fragment and
completion domains are listed above. A pong or reached delivery pairs the signed
WT object with the recipient's SAR acknowledgment inside the recipient's
head-authenticated transport. The signed ping or commit and the exact inner SAR
envelope together determine which obligation may be discharged. Bare broadcast
status has no state-changing meaning.

The following are the only outer context variants; each also includes the base
profile's direction, sender, recipient and protocol version through its channel
key and envelope authentication. `message_type` is an exact ASCII registered
operation from the dispatch table, never a caller-provided arbitrary label.

| Path | Canonical context tuple |
| --- | --- |
| Active WT | `(message_type, ceremony_scope_id, active_wt_head)` |
| Candidate | `("wt_candidate_registration", message_type, setup_id, switch_id, candidate_index)` |
| Independent peer switch control | `("wt_peer_control", message_type, setup_id, previous_head, local_or_common_attempt_id)` |
| ST switch review | `("wt_switch_review", setup_id, previous_head, review_nonce)` |
| Inner SAR placeholder and acknowledgment | Existing approved-withdrawal context, bound to the same Boomlet and fixed SAR |

Registered active types are `wt_sar_response`, `wt_tx_approval`, `wt_approval_collection`,
`wt_tx_commit_ack`, `wt_commit_collection`, `wt_pong`, `wt_reached_collection`,
`wt_fragment_submission`, `wt_fragment_receipt`, and `wt_completion_receipt`.
Candidate types are `registration_request`, `registration_response`,
`payment_request`, `payment_response`, `sar_setup_request`, `sar_setup_response`,
`resume_package`, `placeholder_request`,
`placeholder_response`, and `candidate_ready`. Peer types are `intent`,
`prepare`, `commit`, `recover`, `prepared_certificate`, `decision_certificate`,
and `public_evidence`. Before a common attempt exists, `intent` uses the signed
local intent digest as its attempt ID; subsequent operations use `switch_id`.
Candidate registration cannot deliver a pong, live approval, or commit response.
Candidate `sar_setup_request` transports the existing end-to-end SAR setup
request. Candidate `sar_setup_response` transports the existing SAR-encrypted
`SignedMessage<SarSetupResponse>` directly. Boomlet verifies the inner SAR
signature, setup and rescue-data binding and stores the receipt as preparation
evidence. It needs no WT head-bound forwarding statement before activation.
After activation that same receipt satisfies the SAR-ready local prerequisite.

Ordinary registration and payment receipts are immutable setup facts bound to
their service and invoice. They do not gain state-advancing authority by being
presented over candidate transport. Setup checkpoint signatures remain peer
statements over completed setup evidence. Their WT-specific local prerequisites
use the current selected service. Peer-authored approvals, commits, pings, and
fragments remain WT-neutral; current use requires their origin's authenticated
submission or exact membership in a frozen committed package.

A historical predecessor WT approval or initiator acknowledgment may be parsed
only by the `COMMIT_SET` evidence handler. It verifies the head recorded when
that gate was accepted, exact committed bytes and the limited age exceptions
in README Section 7.4. It cannot dispatch the embedded object to an ordinary
live-message handler. Setup facts and complete signed Bitcoin transactions are
also evidence, with their own validation paths, never sources of WT authority.

Base schemas 21, 22 and 23 retain their setup backup meanings.
`BoomletBackupState` is admitted only by the authorized one-time setup import
into the designated inactive Boomletwo. `BackupDone` retains its base payload
and signature context. The setup ID and authenticated parameters bind the WT
roster; exact encoding of this parameter extension requires conformance vectors.
WT service changes never invoke a backup import or reactivate that handler.

`WtPublicArchive` is optional public evidence storage. It is not a private
backup, a replacement for `BackupDone`, or an authority-bearing receipt.
Its public bytes retain the 16,384-byte bound.

## Resume packages and conformance

Each package is a canonical `list<WtResumeEntry>` with at most 32 entries. Kinds are 0 setup agreement, 1 checkpoint,
2 service receipt, 3 reviewed transaction, 4 approval set, 5 predecessor WT
approval, 6 approval-set attestations, 7 padded commit, 8 initiator acknowledgment,
9 padded ping, 10 SAR acknowledgment, 11 reached collection, 12 signed fragment,
13 complete transaction, 14 completion receipt, and 15 signing disposition.
Sort by kind and then origin peer; reject duplicates for the same kind and origin.
Origin is derived from the validated artifact, never an untrusted extra signer.
A collection kind contains exactly its protocol-defined five or four signers;
a local kind has one origin. Phase requirements in README Sections 4, 5, 7 and 9
are mandatory. A kind can never contain another package. Transaction and fragment
byte payloads are parsed against the ordinary descriptor and signing profile.

Exploratory checks covered schema composition, fixed counts and domains,
resource bounds, representative canonical hash inputs, and dispatch guards
using authenticated abstractions. Independent
implementations must additionally agree on complete signed and encrypted wire
vectors, malformed-field rejection, every setup and withdrawal phase, exact
receipt replay, and power-loss boundaries. The proposed profile is unavailable
for deployment until those conformance and device requirements are satisfied.

## Self-contained ping candidate

The [Boomletwo candidate](boomlet_rollover.md) requires target-encrypted checkpoint
fields, a detached ciphertext, a five-ping pong schema and an activation handler.
The current catalog defines none of them. Adoption requires fixed fields, padding,
bounds, incarnation rules and version-selected dispatch; one profile cannot
accept both four-ping and five-ping shapes. Counter freshness still uses the
other four peers. Checkpoint evidence alone grants no progress or authority.
