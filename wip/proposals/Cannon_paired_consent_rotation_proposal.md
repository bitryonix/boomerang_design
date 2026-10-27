# Paired consent rotation proposal

[Complete sequence](Cannon_paired_consent_rotation_sequence.puml).
[Integrated sequence](../../diagrams/ancillary/consent_rotation_diagram.puml).
Decisions are recorded in [ADR 0009](../../adr/0009-niso-relayed-consent-rotation.md) and
[ADR 0010](../../adr/0010-consent-counter-display.md).

Adopted in [SPEC Section 16.7](../../spec/SPEC.md#167-consent-rotation). Encoding, signatures,
directional channels, and CBC-CMAC follow SPEC Sections 8–9.

## Behavior

Boomlet and its designated inactive Boomletwo replace their unordered
five-country `duress_consent_set`. The descriptor, logical peer, ST input method,
and WT and SAR registrations remain intact. New attempts begin only in `IDLE`
or `DIGGING`; a stalled withdrawal uses its retained phase. DIGGING includes
waiting for other peers after reaching the local mystery threshold.

The user enters the current set on ST to approve rotation, chooses the new set,
and confirms that selection. Boomlet sends Boomletwo one authenticated message
containing ST approval, the chosen set, and the duress flag. Boomletwo
saves the set and flag before confirming receipt. Boomlet requires that receipt
before resuming consent or reporting completion.

Every successful replacement MUST include the current-set check through trusted
ST. For a valid wrong answer, Boomlet MUST set `under_duress` to true before any
reply and continue through the same selection, confirmation, and delivery flow
as for a correct answer:

```text
under_duress := under_duress OR answer_is_wrong
user_is_in_duress := under_duress
```

Correct answers, cancellation, confirmation, retries, restart, and further
rotation cannot clear the flag. Wrong ordinary withdrawal answers can also set
it. Subsequent commitments and pings carry the key-bearing SAR placeholder while
it is true. Idle duress carries into the next withdrawal. A valid mistaken answer
can cause false duress.

ST immediately repeats confirmation until it matches the selected set. Install
the confirmed set as a new consent state even when its countries equal the
current set. Cancellation after Boomlet records ST approval sends the current set
and stored flag and requires the same receipt. After receipt both devices
agree and consent may resume, subject to the withdrawal's SAR resumption gate.

Boomlet installs the chosen set when saving its outgoing message; Boomletwo
installs it before replying. Erase the old set at installation. Keep no past sets.

## State and pairing

| State | Meaning |
| --- | --- |
| `duress_consent_set` | Current set; ST holds the candidate during input. |
| `under_duress` | Durable Boolean, initially false. |
| `consent_counter` | Shared monotonic `u64` that rejects replayed approvals and resets. |
| `rotation_state` | One attempt: ST approval, current stage and query, latest answer digest and successors, chosen outcome, exact device message, and ST completion reply once created. |

Boomlet stages are `new_set_input`, `waiting_receipt`, and `complete`.
Boomletwo retains its last accepted message and receipt.
The initial query and incomplete attempt pause Boomlet's consent operations and
withdrawal.
Boomlet keeps a completed attempt until it records another ST approval or
completes paired reset.

Follow SPEC Section 18.3 for crash recovery and retries. Reserve bounded space
for one attempt, its recovery cache, and a paired reset. Keep reset receipts
separately for retries.
Increment the counter once per recorded rotation outcome, including identical
countries and cancellation after Boomlet records ST approval, and once per paired
reset. Confirmation retries and exact message retries never increment it.
Existing signing, replay, and rescue evidence remains in the withdrawal record.

Setup Step 26 initializes `consent_counter := 0` and
`under_duress := false`. Copy their current values to Boomletwo in setup's
authenticated state transfer. On restart, load both from protected storage.
Preserve the counter's monotonic value across restart, restore, setup changes,
and re-pairing while identity keys are reused; missing or regressed state blocks
that device's operations. Reinitialization requires fresh pair and ST identities.
Reject overflow. `BEGIN_ROTATION()` requires
`consent_counter < 2^64 - 2`, reserving increments for both the outcome and reset.

Use the pair established by setup's normal-key-authorized backup request,
authenticated state transfer, and signed `BackupDone`. Boomlet retains the
backup public key; Boomletwo retains the setup, source, and ST identities.
Rotation cannot change those identities.

Boomletwo signs replies with its original identity key. Its inactive applet
confines the copied source key to SAR verification; it cannot originate source,
ST, or withdrawal messages or expose arbitrary signing or encryption with that
key. Roles and restrictions survive restart. The inactive backup only OR-merges
Boomlet's flag, which Boomlet retains until reset.

## Current-set query and approval

Boomlet generates `rotation_id := random_bytes(32)`, then creates
`consent_rotation_request`, a `MessageWithNonce<u64>` containing
the current `consent_counter` with `nonce: rotation_id`, and one fresh permutation
of 1–193. It sends both in the initial encrypted query:

```text
ConsentRotationChallenge = (
  request: MessageWithNonce<u64>,
  duress_check_space: DuressCheckSpace
)
ConsentRotationAnswer = (
  approval: SignedMessage<MessageWithNonce<u64>>,
  duress_signal_index: DuressSignalIndex
)
```

ST authenticates Boomlet and displays “Change consent set. Enter current set to
approve.” This prompt omits the counter. Submitting the five choices approves
rotation. ST signs the request under `Boomerang/consent_rotation/v1/st_review`
and encrypts approval and indices together. The approval uses `rotation_id` as
its nonce to bind that response; retain the approved request and exact response
before output. Boomletwo receives only approval, chosen set, and flag. No mnemonic,
passphrase, or normal private key is needed.

| Local selector | Command and input | Output |
| --- | --- | --- |
| 41 | BEGIN_ROTATION() | Encrypted `ConsentRotationChallenge`. |
| 45 | RESUME() | Retained next message; Boomletwo returns its current receipt. |

Selectors 42–44 are reserved; local commands supply no authority.
BEGIN_ROTATION requires an active source in IDLE or DIGGING, an available ST
session, counter capacity for outcome and reset, and no pending attempt.
In DIGGING, finish and send any ping already under construction and service the
request at the start of the next iteration before its duress check and next ping
construction. If digging ends first, leave the request unperformed. IDLE requests start immediately.

Rotation starts when Boomlet issues the initial ST challenge; generate it only
at entry. Pause consent operations and the current withdrawal, preserving the
iteration and SAR duties. Retain one volatile query and exact envelope; repeats
return it. Cancellation or query loss before Boomlet records ST approval
releases this pause; query loss requires a fresh query and nonce.

On receiving the first ST answer, verify the ST signature, current counter,
exact outstanding request, and five valid distinct indices. Resolve them using
the retained permutation and compare all five countries. Save approval,
updated `under_duress`, consumed response digest, exact new-set query, `new_set_input`,
and continued withdrawal pause before replying. Correct and valid wrong answers both authorize
rotation. Invalid responses leave state unchanged.
The counter remains unchanged during input; the fresh nonce distinguishes queries
at the same counter value.

A retry matching the latest accepted digest returns the exact successor without
reclassification. Earlier responses are stale; conflicting responses cannot
replace an attempt. Before recording ST approval, Boomlet requires any preceding
attempt to be complete and no pending reset. Pairing or setup changes retire
queries.

## New set and confirmation

Boomlet sends one `consent_set_query`, a
`MessageWithNonce<DuressCheckSpace>` with `rotation_id` as nonce and one fresh
permutation. ST authenticates the query and approved nonce, displays five
independently shuffled columns, and holds the user's five distinct selected
countries locally. It independently reshuffles the columns for confirmation and
immediately repeats confirmation on mismatch, preserving that confirmation map.

Only after both entries match does ST send one `consent_set_answer`, a
`MessageWithNonce<DuressSignalIndex>` with `rotation_id` as nonce. Its indices
refer to the original query's permutation. Boomlet checks the outstanding
`new_set_input` stage, nonce, and five valid distinct indices, resolves the final
set, and installs it with the outgoing device message. It holds no candidate or
confirmation stage during input. Selection and confirmation never classify
duress; local mismatches leave Boomlet's counter, flag, and withdrawal pause intact.

Matching the two entries relies on ST's trusted display and input implementation
(SPEC Section 5.3). Boomlet validates the final authenticated answer but has no
earlier entry against which to detect an otherwise valid ST encoding fault.

After authenticating the new-set query for the approved `rotation_id`, ST derives
`new_consent_counter := consent_rotation_request.content + 1` immediately before
displaying the new-set selection screen. Boomlet releases that query only after
recording ST approval and updated `under_duress`. Show the counter as
pending during selection and confirmation, and as applied with the authenticated
final result after Boomletwo's receipt. This counter applies to identical
countries and cancellation too. Status and recovery follow the same restrictions
and require an authenticated matching phase. Correct and valid wrong answers
reach the same counter displays.

Consume the final answer with the installed set and outgoing device message.
Retain only the latest answer digest and successors. Duplicate queries preserve
ST's current local entry and maps or return its exact final answer; conflicting
content fails. ST records its final answer before output and freezes it thereafter.
Before final emission, lost local input may restart selection and confirmation
against authenticated pending status and the same query. If emission cannot be
ruled out, recover the exact answer or cancel and finish the recorded outcome
before starting a fresh rotation. Status alone cannot exclude an answer in transit.

Cancellation uses `MessageWithNonce<u8>{content: 0, nonce: rotation_id}` under
`cancel`, authenticated by ST's channel. Verify zero content and a matching
outstanding query or input stage; serialize against the current-set answer,
confirmed new-set answer, and final message creation. Before Boomlet records ST
approval, retire only the volatile query. Afterward, send the unchanged set and
stored flag.
A saved outgoing device message is irrevocable; invalid messages preserve pending input.

Lost query state before Boomlet records ST approval requires a fresh first query;
ST state loss afterward requires authenticated status before input or completion.

## Set delivery and receipt

```text
ConsentSetChange = (
  approval: SignedMessage<MessageWithNonce<u64>>,
  duress_consent_set: Collection<u16>,
  under_duress: bool
)
ConsentReset = (
  consent_counter: u64,
  approved_withdrawal_id: bytes32,
  sar_receipt: CbcCmacEnvelope
)
```

Consent sets are unordered; comparisons ignore country order. Encode
`ConsentSetChange.duress_consent_set` as exactly five ascending, distinct integers
in 1–193. The approval is signed by ST; its content is the current counter and
its nonce identifies the attempt. The source signature attests to accepted
input and the chosen outcome.

| Selector | Operation | Exact signed content |
| --- | --- | --- |
| 5 | `consent_set` | `ConsentSetChange`, including ST-signed approval. |
| 6 | `consent_set_received` | SHA-256 of the exact consent-set envelope's canonical encoding. |
| 12 | `reset` | `ConsentReset`, including the current counter. |
| 13 | `reset_ack` | SHA-256 of the exact RESET envelope's canonical encoding. |

Requests flow Boomlet to Boomletwo; receipts flow back. Identity signatures bind
the stated operation and protocol version. Both endpoints and their roles are
bound by the paired directional channel; a receipt acknowledges the whole request,
including its approval, set, and flag.

Boomlet installs the chosen set, increments `consent_counter` by one,
and saves the exact encrypted message, fixed outcome (`rotated` or `cancelled`),
flag, and `waiting_receipt`. Erase the query and predecessor
set; retain the latest answer digest for retries. Once Boomlet records ST approval,
completion and cancellation use this sequence.

Boomletwo verifies both signatures, the fixed pair, inactive role, valid set,
available storage, and no pending reset. Require the approval's counter to equal
its current `consent_counter`, with capacity for outcome and reset.
Install the set, OR-merge the flag, increment `consent_counter` by one,
and save the exact request and receipt before output. Erase the predecessor set
and remain inactive.

Boomlet accepts a receipt only while `waiting_receipt`, from its paired backup,
with the exact recorded envelope digest. Immediately before sending the ST
acknowledgement, encrypt the existing `rotation_id: bytes32` under `completed`
for a confirmed rotation or `cancelled` for cancellation.
Save the receipt, `complete`, any paused withdrawal's SAR resumption gate, and
the exact encrypted ST reply before output. Exact retries recover that reply.
ST authenticates the context and requires the ID to match its approved rotation
before displaying the outcome and applied counter. For completion, cancellation,
or terminal recovery status, derive that counter from the retained
approved `consent_rotation_request.content + 1`, even if selection was never
displayed.

## Withdrawal and reset

During DIGGING preserve the withdrawal's identifiers, approvals, phase, mystery,
progress, signing history, replay state, current iteration, and SAR duties.
While paused, produce no new progress, withdrawal ST input, signing output, or
fragment export. Continue handling emitted messages and their SAR receipts.

After Boomletwo's receipt, resume the same iteration with fresh country input
using the installed set. Update `under_duress` using that input before constructing the
next placeholder. Apply ordinary pong processing and send the next-sequence
ping carrying the resulting `under_duress`, retaining TxCommit. Persist a
resumption gate bound to `consent_counter`, that exact packet, and expected
placeholder before output. Its exact verified SAR receipt permits later iteration
progress, signing, and export; an earlier ping's receipt cannot clear the gate.
Preserve older duties and ordinary freshness and chain-view checks.

Once rotation has started, keep local DIGGING until this receipt, even if WT
finishes meanwhile. Retain the reached collection for signing revalidation.
Rotation supplies no progress credit or deadline extension. Both flag values
follow this path.

WT forwards each valid ping's placeholder to the setup-bound SAR and relays
`(approved_withdrawal_id,
duress_placeholder_signed_by_sar_encrypted_by_sar_for_boomlet_0)` through Niso,
including after digging. Verify the approved ID, SAR signer, context, and exact
placeholder. After digging, WT replaces the source's reached evidence with its
latest true ping and redistributes it for signing revalidation. Receipts create
no pong, progress, or reopening of digging.

At withdrawal end freeze duress input and send a fresh final ping for either
flag value. Persist the packet and expected placeholder; require its exact SAR
receipt before clearing the flag. WT retains delivery through cleanup.
Block new withdrawals and rotations until delivery and reset finish.
Abandonment before an approved withdrawal ID carries duress into the next withdrawal.

If an attempt record exists, require `complete` and perform paired RESET.
Boomlet records withdrawal end, verified final SAR receipt, and exact RESET,
retaining its flag and counter. RESET contains its current counter and is signed
directly as `ConsentReset`. Boomletwo verifies the source signature, matching
current counter with room to increment, inactive role, and a saved consent-set
receipt. Using the restricted copied-source keys, verify the receipt's approved ID, SAR signer,
context, and placeholder. Its plaintext must be zeros or `doxing_key_for_sar`;
a true local flag requires the key-bearing value. The source signature attests
to this exact final receipt for the ended withdrawal.

Boomletwo clears its flag and accepted consent-set record, increments
its counter by one, and records exact RESET_ACK before output. Boomlet verifies
the exact pending RESET digest and that its counter still matches its recorded
RESET, then clears its flag and attempt, increments its counter by
one, and finishes cleanup. Keep the reset receipt for read-only retries and
preserve signing, replay, and rescue evidence.
With no attempt record, clear locally after final SAR delivery.
Both flag values follow the same reset path. Rotation alone never clears duress.

## Recovery and activation

RESUME returns retained bytes. An exact consent-set message
returns its cached receipt without reinstalling or merging flags. An exact
RESET returns cached RESET_ACK.
Old approvals, answers, device messages, and receipts cannot change another
attempt or a flag after reset.

| Interruption | Recovery |
| --- | --- |
| Before DIGGING rotation starts | Finish and send any ping already under construction; enter rotation at the start of the next iteration. If digging ends before entry, leave the request unperformed. |
| Before Boomlet records ST approval | A lost initial query requires a fresh nonce and current-set answer. |
| After Boomlet records ST approval, during input | Current set and accepted duress remain; recover the final ST answer, resume local entry if none was emitted, or cancel through ST. |
| After source installs, before backup receives | Source has the chosen set, backup its predecessor; source stays paused. Retry the exact consent-set message. |
| After backup saves, before source receives confirmation | Both have the chosen set; source stays paused. Retry the message to recover its cached receipt. |
| After backup receipt, before the next ping's SAR receipt | Resume the saved iteration or retry its exact ping; signing and export remain blocked, including after WT finishes. |
| During reset | Retry exact RESET; cached RESET_ACK cannot clear a later flag. |

Niso stores no authoritative state. Restart recovers each device's atomic records
and stored duress. Timeout, invalid messages, or unavailable storage preserve its
recorded stage. Missing or regressed protected state blocks that device.

Boomletwo remains inactive during rotation. On activation it uses its own stored
`duress_consent_set`, `consent_counter`, and `under_duress`. Its first consent query
asks for the previous set through a fresh authenticated ST challenge that includes
its counter. ST displays it through retries and recovery until that answer is
accepted. Boomlet's current-set rotation query, ordinary withdrawal
queries, and later backup consent queries omit the counter from display.
Boomletwo checks the answer against its stored set and sets `under_duress` to
true for a valid wrong answer before continuing. A correct answer preserves its stored flag.

If the combined set-and-flag message was saved, use those values even if its ACK
was lost; otherwise use the previous stored values. Activation can proceed with
Boomlet permanently unavailable. Single-active-device protection and role
transition remain separate activation requirements (SPEC Section 18.5).

For ST recovery, ST sends `MessageWithNonce<bytes32>` containing its approved
`rotation_id` and a fresh ST status nonce under `ui_status`. Boomlet verifies the
ID and returns `MessageWithNonce<RotationStatus>` under `ui_status_response`,
echoing the status nonce:

```text
RotationStatus = (status: u8, space: DuressCheckSpace)
```

Status is 0 WAITING, 2 NEW_SET_INPUT, 4 ROTATED, or 5 CANCELLED;
1 and 3 are reserved. NEW_SET_INPUT carries the current permutation; others use
the identity permutation as padding, which ST never displays. Input always uses
the approved `rotation_id` as its nonce. Status requires a matching `rotation_state`
and follows the recorded stage and verified receipt; it never replaces
a prompt or resumes consent. No matching attempt requires a fresh BEGIN_ROTATION,
which still rejects any unresolved attempt. Silence cannot establish whether
Boomlet recorded ST approval.

Keep one ST status nonce, its exact encrypted reply, and a bounded request count.
The same nonce returns its snapshot. Another nonce snapshots current state and
replaces the cache before output. ST retains one outstanding request,
retries its envelope, and accepts only its matching nonce. State loss requires
a fresh request. Older replayed requests can spend the budget but cannot change
input, flags, or decisions.

## Wire and security rules

Device signatures use `Boomerang/consent_rotation/v1/` plus the operation;
encryption uses `canonical_encode("consent_rotation_" + operation, setup_instance_id)`.
ST encryption uses `canonical_encode("consent_rotation_" + suffix)`.
The approval signature uses its stated domain. The combined initial query and
answer use dedicated `current_set_challenge` and `current_set_answer` contexts;
ordinary withdrawal answers cannot authorize rotation. Other ST messages use
channel authentication. ST selectors are 23 `current_set_challenge`,
24 `current_set_answer`,
26 `new_set_query`, 27 `new_set_answer`,
34 `cancel`, 35 `completed`, 36 `ui_status`, 37 `ui_status_response`, and
38 `cancelled`. The `completed` and `cancelled` payloads are exactly `bytes32`.
ST selectors 21–22, 25, and 28–33 are reserved. Device selectors 1–4 and 7–11
are reserved.

| Bound | Requirement |
| --- | --- |
| Input | One current-set answer and one confirmed new-set answer; selection and confirmation retries stay on ST. |
| Automatic retries | Eight per object; preserve state on exhaustion. |
| New ST status nonces | Eight per attempt; retain current reply on exhaustion. |
| Envelopes | At most 2,048 bytes for devices, 1,536 for ST; nesting depth at most 12. |
| Frames | Fixed 4,096 bytes between devices and 2,048 for ST. |

Retry and status counts survive restart; RESUME preserves them. Framing is
`operation: u8 || envelope_length: uint16_be || envelope_bytes || zero_padding`,
excluded from signatures and digests. Enforce exact types, canonical encoding,
lengths, depth, and zero padding before allocation or semantic use. Reject unknown
versions, selectors, and enum values. Fresh messages use fresh IVs; exact retries
reuse stored bytes. Reusing the approved nonce across phases is permitted only
with their distinct fixed contexts and one immutable prompt per phase.

ST approval permits rotation within the fixed pair. Control of the trusted
devices and ST can permit unauthorized maintenance or forced duress. Knowledge
of the correct set can avoid that check's signal. Replacement, rebinding, signing,
and activation retain separate requirements. Rotation cannot revoke released
signing output.

Correct and wrong previous answers and both flag values MUST produce identical
prompts, message counts, lengths, routing, retries, comparison work, durable
writes, queues, trusted fixed deadlines, and permitted failures. Keep sets,
flags, signatures, and receipts encrypted; their canonical shapes must conceal
the flag even if relay padding is stripped. Fix each operation's release time on
arrival before parsing; missed deadlines give the same failure without a late
reply. Keep classification out of status, logs, metrics, and diagnostics,
including recovery and reset, and preserve SPEC Sections 16.3–16.6.
