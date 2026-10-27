# Withdrawal development contracts

This file supplies the compact contracts referenced by the initiator and
non-initiator protocol-development diagrams. It is a development aid.
Normative requirements remain in [`SPEC.md`](../../spec/SPEC.md), especially
Sections 14 through 18.

## Notation

| Form | Meaning |
| --- | --- |
| `Sig_X(M)` | Domain-separated signature by actor `X` over canonical `M`, including the active protocol version. |
| `Enc_X(M)` | Authenticated encryption for `X` under the exact directional channel and context required by the specification. |
| `H(fields)` | The specification-defined tagged hash over canonical typed fields. |
| `via Niso` | Niso transports the object and performs every visible prevalidation available to it. Endpoint signatures and encryption remain authoritative. |
| `x4` or `x5` | One object from each indicated active role, with membership, uniqueness, and canonical ordering enforced. |
| `G-NAME` | The complete acceptance guard defined below. |
| `W-DURESS-CHECK` | The reusable withdrawal duress fragment defined below. |

Unless a guard says otherwise, it includes canonical schema and size bounds,
protocol version, expected phase, sender identity, ceremony identifiers,
signature or authenticated-envelope verification, and replay checks. Niso
prevalidation protects the constrained device. Boomlet independently
revalidates every condition needed to change trusted state or release its
signing share.

## State contract

| Transition | Required Boomlet event |
| --- | --- |
| `IDLE -> REVIEWING_TX` | Setup is complete, no withdrawal or unresolved consent rotation is active, the primary branch is eligible, the proposed PSBT passes local authorization checks, and a fresh ST review is created. |
| `REVIEWING_TX -> APPROVED` | Boomlet verifies ST's signature over its exact outstanding nonce-bound `tx_id`, then signs its `TxApproval`. |
| `APPROVED -> COMMITTED` | Initiator signs its own `TxCommit`; a non-initiator first verifies the WT-signed initiator commit and then signs its own commit. |
| `COMMITTED -> DIGGING` | Boomlet verifies the exact complete five-commit collection and its own SAR acknowledgment. It draws `mystery` exactly once on this transition. |
| `DIGGING -> READY_TO_SIGN` | Boomlet verifies the latest reached ping from every peer and a hydrated PSBT preserving the committed transaction; rotation and its exact SAR resumption receipt gate are complete. |
| `READY_TO_SIGN -> SIGNING` | Boomlet begins a fresh MuSig2 session for the verified PSBT and isolated normal-key participant. |
| `SIGNING -> SIGNATURE_EXPORTED` | Both partial signatures verify, the peer fragment is complete, export finishes, and session nonce material is erased. |
| `SIGNATURE_EXPORTED -> IDLE` | Signed-fragment export, exact final SAR receipt and any paired reset complete under SPEC Section 16.7; clear active state and mystery. |

## Ceremony bindings and semantic objects

| Object | Required semantic content and binding | Protection and use |
| --- | --- | --- |
| `TxReview` | Exact `tx_id` and fresh Boomlet nonce. | Encrypted Boomlet to ST; ST signs the exact object after user confirmation and encrypts it back to Boomlet. |
| `withdrawal_id` | `H(setup_instance_id, tx_id, initiator Boomlet identity, initiator approval nonce)`. | Scopes proposal distribution and all `TxApproval` values. |
| `TxApproval_i` | `withdrawal_id`, fresh peer approval nonce, and event block height. | Signed by Boomlet `i`; encrypted to WT during collection. |
| `WtTxApproval` | `withdrawal_id` and WT event block height. | Signed by WT after accepting the initiator approval. |
| `approved_withdrawal_id` | `H(withdrawal_id, exact ordered five signed TxApprovals)`. | Scopes commitments, placeholders, pings, pongs, reached evidence, signing, export, and replay state. |
| Approval-set attestation | Fingerprint over the exact ordered five approvals and `WtTxApproval`. | Signed by each non-initiator Boomlet. It proves receipt, verification, and agreement on the approved withdrawal. |
| `TxCommit_i` | `approved_withdrawal_id` and event block height. | Signed by Boomlet `i`; paired with its Placeholder in one signed outer object encrypted to WT. |
| `Placeholder_i` | Fresh encryption of either the safe value or peer `i`'s SAR-specific duress key, bound to `approved_withdrawal_id`. | Routed by WT only to peer `i`'s setup-bound SAR. Traffic shape does not reveal classification. |
| SAR acknowledgment | SAR signature over the exact encrypted Placeholder it durably processed. | Encrypted for the originating Boomlet. It must match that Boomlet's exact sent Placeholder. |
| `Ping_i` | `approved_withdrawal_id`, `last_seen_block`, strictly increasing `ping_seq_num`, and monotonic `reached_mystery_flag`. | Signed by Boomlet `i`; paired with a freshly encrypted Placeholder in one signed outer object encrypted to WT. |
| `Pong_i` | `approved_withdrawal_id`, WT event block height, and one current signed ping from every peer except recipient `i`, in setup order. | Signed by WT and encrypted for Boomlet `i`. |
| Reached collection | One current signed ping with reached flag set from every peer. | Signed and distributed by WT; Niso prevalidates and Boomlet independently revalidates. |
| Hydrated PSBT | Original approved transaction plus permitted signing support data. | Niso constructs it; Boomlet requires unchanged transaction identity, semantics, ordering, sequences, sighash policy, and descriptor authorization. |
| Signed peer fragment | Completed peer MuSig2 contribution for the approved PSBT. | Exported after signing; WT aggregates five fragments and verifies the exact committed `tx_id`. |

## Reusable duress fragment

`W-DURESS-CHECK` has the following contract.

1. Boomlet creates a fresh permutation of `1..193`, a fresh nonce, and an
   outstanding record bound to the active withdrawal phase.
2. Boomlet encrypts the challenge for its paired ST. Niso only transports it.
3. ST authenticates the challenge, maps it through the fixed vocabulary,
   creates five independent display shuffles, and retains the reverse maps.
4. User selects one displayed entry from each column.
5. ST returns exactly five original indices and the same nonce in an
   authenticated envelope for Boomlet.
6. Boomlet requires one outstanding challenge in the current phase, exact
   nonce equality, exactly five distinct in-range indices,
   and successful authenticated decryption.
7. Boomlet consumes the outstanding challenge and updates
   `under_duress := under_duress | (resolved_set != duress_consent_set)`.
   `Evaluate(under_duress, doxing_key_for_sar)` returns the key when the flag is
   true, or 32 zero bytes otherwise. Correct later answers preserve the flag.
8. Every later Placeholder uses a fresh IV and is bound to the active
   `approved_withdrawal_id`, including rounds where the plaintext did not
   change.

Finish and send any ping already under construction, then enter rotation at the
start of the next DIGGING iteration before input and next ping construction. Apply
[SPEC Section 16.7](../../spec/SPEC.md#167-consent-rotation) for recovery
and SAR receipt gates.

## Acceptance guards

### Review and approval

| Guard | Owner | Acceptance conditions |
| --- | --- | --- |
| `G-WITHDRAWAL-PRECONDITIONS` | Niso and Boomlet | Final local setup checkpoint exists; no active withdrawal or unresolved consent rotation; current local block height reaches `milestone_block_0`; selected inputs are controlled by the Boomerang descriptor and satisfiable through the primary branch. Boomlet revalidates independently. |
| `G-PSBT-PREVALIDATION` | Niso | Canonical PSBT, bounds, inputs, outputs, amounts, fees, sighash policy, descriptor membership, milestone eligibility, and hydratability are acceptable before invoking Boomlet. |
| `G-PSBT-AUTHORIZATION` | Boomlet | Transaction parsing, descriptor membership, primary-path eligibility, allowed sighash, transaction identity, input authorization, and local state all pass on Boomlet. |
| `G-TX-REVIEW-APPROVAL` | Boomlet | ST signature and identity are valid; returned review equals the exact outstanding `{tx_id, nonce}`; nonce is fresh and current; Boomlet consumes the outstanding review in `REVIEWING_TX`. |
| `G-INITIATOR-APPROVAL-AT-WT` | WT | Initiator is a peer in the current setup; outer context and signature are valid; outer and signed `withdrawal_id` agree; block height is within the profile window; encrypted PSBT collection contains the intended recipient entries. |
| `G-NONINITIATOR-OFFER` | Niso and non-initiator Boomlet | WT and initiator signatures verify; initiator membership is valid; approval IDs and block heights agree; milestone is eligible. Boomlet decrypts its PSBT, recomputes `withdrawal_id` from the active setup and transaction, and requires equality. Niso validates visible conditions before forwarding; Boomlet revalidates independently. |
| `G-TX-APPROVAL-COLLECTION` | WT | Exactly one valid current `TxApproval` from every peer; every approval carries the same `withdrawal_id`; no missing or duplicate signer; block heights satisfy the profile; result is ordered by setup peer order. |
| `G-APPROVAL-SET-AT-PEER` | Niso and Boomlet | Expected WT approval and exact peer approval membership; valid signatures; one approval per peer; canonical setup order; common `withdrawal_id`; profile freshness relations. Each computes the same `approved_withdrawal_id`; Boomlet revalidates independently. |

### Commitment and SAR gating

| Guard | Owner | Acceptance conditions |
| --- | --- | --- |
| `G-APPROVAL-ATTESTATIONS` | WT | Exactly four signatures from all and only non-initiators; no duplicate; every content equals WT's recomputed fingerprint over the exact ordered approval set and `WtTxApproval`. |
| `G-INITIATOR-COMMIT-AT-NONINITIATOR` | Niso and non-initiator Boomlet | WT and initiator signatures verify; commit binds the expected `approved_withdrawal_id`; block height is current under the recipient's local view; phase permits non-initiator commitment. Boomlet revalidates independently. |
| `G-TX-COMMIT-AT-WT` | WT | Attestation gate is open; outer envelope and signature verify; inner Boomlet signature verifies; commit binds the expected approved ID and fresh block height; Placeholder belongs to the same authenticated outer object; exact SAR acknowledgment has returned before WT signs the commit. |
| `G-SAR-PLACEHOLDER` | SAR | Record the fixed acknowledgment deadline on complete request receipt. Authenticated decryption under the originating Boomlet channel succeeds; context binds the active approved ID; plaintext is exactly safe or derives an existing SAR record. Safe and duress values follow the same bounded processing, fixed-shape durable write, acknowledgment construction, release deadline, retry behavior, and visible failure behavior. A fresh tuple of approved ID, Boomlet identity, and IV is durably recorded before acknowledgment; a valid duress value commits rescue activation. A missed deadline releases no late acknowledgment. Malformed values receive no acknowledgment. |
| `G-COMMIT-COLLECTION` | Niso and Boomlet | Exactly one WT-signed, Boomlet-signed `TxCommit` per peer; expected membership and uniqueness; common approved ID; profile freshness relations; recipient's SAR acknowledgment decrypts, verifies, and covers the exact Placeholder it sent. Boomlet revalidates independently before entering `DIGGING`. |

### Digging and signing

| Guard | Owner | Acceptance conditions |
| --- | --- | --- |
| `G-PING-AT-WT` | WT | Outer envelope, Boomlet signatures, signer membership, approved ID, block-height range, strict sequence increase, and reached-flag monotonicity pass. First ping cannot claim reached. The paired Placeholder receives its exact SAR acknowledgment before the ping contributes to a pong. |
| `G-PONG-AT-BOOMLET` | Niso and Boomlet | WT signature and recipient encryption verify; approved ID and WT block height are fresh; pong contains exactly one current signed ping from every other peer, in setup order, with no recipient, missing peer, or duplicate; each ping has expected signer, approved ID, monotonic sequence, permitted block height, and monotonic reached state. Exact local SAR acknowledgment also verifies. Pending rotation or a SAR resumption gate blocks later progress; replayed pongs do not change state. Boomlet revalidates independently. |
| `G-COUNTER-ADVANCE` | Boomlet | Evaluate before modifying `last_seen_block`. Local Niso block height advanced beyond the prior local `last_seen_block`, and every included other-peer ping has `last_seen_block` within the configured window around that local block height. A valid pong that fails this guard performs bounded catch-up without increment. |
| `G-REACHED-COLLECTION` | Niso and Boomlet | WT signature verifies; exactly one signed ping from every peer; expected signer membership, setup order, common approved ID, current sequence, fresh block height, reached flag set, and no missing or duplicate peer. Boomlet revalidates independently after any resumption gate. |
| `G-HYDRATED-PSBT` | Boomlet | Hydration changes only permitted support data; exact `tx_id`, inputs, outputs, amounts, fees, ordering, sequences, sighash policy, and descriptor authorization remain unchanged; reached collection and active state still match. |
| `G-MUSIG2-SIGNING` | Iso and Boomlet | Signing package is internally consistent and belongs to the approved PSBT and descriptor; normal key and Boomlet share aggregate to the expected peer key; no rotation or SAR resumption gate is pending; latest reached evidence revalidates; nonces are fresh and session-bound; each partial verifies before finalization; nonce material is erased on completion or failure. |
| `G-FINAL-FRAGMENTS` | WT | Exactly one valid signed fragment from every peer; every fragment belongs to the approved withdrawal and exact committed `tx_id`; aggregation produces a valid final transaction with that same identity. |

## Withdrawal invariants retained by the compact diagrams

1. The trusted ST confirmation covers the exact transaction identity derived by Boomlet.
2. `withdrawal_id` binds setup, transaction, initiator identity, and a fresh initiator nonce.
3. `approved_withdrawal_id` binds the exact unanimous approval set and scopes every later security object.
4. Every non-initiator proves receipt and verification of the exact approval set before WT releases the initiator commitment path.
5. WT routes every Placeholder to the originating peer's setup-bound SAR and waits for the exact acknowledgment required by that step.
6. A non-initiator signs its commitment only after independently verifying the WT-signed initiator commitment.
7. Boomlet enters `DIGGING` and draws one fresh mystery only after the complete commit set and its own initial SAR acknowledgment verify.
8. Counter advancement depends on local chain progress and current other-peer progress. Catch-up alone cannot increment it.
9. Reached flags and observed peer reach state are monotonic.
10. A reached Boomlet continues supplying current pings and Placeholders until all five are reached.
11. Niso prevalidates and Boomlet revalidates reached evidence and the hydrated transaction.
12. Signing and final aggregation preserve the exact approved transaction identity.

## Failure and retry contract

- A failed guard stalls the active ceremony while retaining its setup,
  withdrawal identifiers, accepted transcript, and replay state.
- An identical authenticated object may be retransmitted where SPEC permits
  it. Nonces, IVs, ping sequences, approval values, and MuSig2 session material
  cannot be repurposed.
- WT or SAR unavailability stalls the ceremony at the corresponding gate.
  Progress cannot skip a missing SAR acknowledgment.
- Block-height decrease or material local and WT chain-view disagreement stalls
  with the chain-safety failure behavior.
- Explicit abandonment follows SPEC Sections 16.7 and 18.2.
- Any signing-session failure follows BIP327 nonce-safety rules and erases
  session nonce material before retry.

## Detailed coverage

| Development fragment | Detailed initiator steps | Detailed non-initiator steps | Normative source |
| --- | --- | --- | --- |
| `WI-REVIEW`, `WI-APPROVE` | 1 through 27 | `WN-OFFER`, `WN-REVIEW`, and `WN-APPROVE`, original steps 10 through 27 | SPEC Sections 15.1 through 15.4 |
| `WI-COMMIT` | 28 through 45 | `WN-COMMIT`, original steps 28 through 44 | SPEC Sections 15.5, 15.6, and 16 |
| `WI-DIG` | 46 through 59 | Common peer-local sequence after original step 44 | SPEC Sections 15.7 through 15.10 and 16 |
| `WI-FINAL` | 60 and 61 | Common peer-local sequence | SPEC Sections 15.11 and 15.12 |
| `WI-SIGN`, `WI-RELAY` | 62 through 73 | Common peer-local sequence | SPEC Sections 15.13 and 15.14 |

Pure forwarding is collapsed into `via Niso`. Exact message schemas, signed
domains, encryption contexts, freshness inequalities, block-height tolerances,
canonical field order, retry rules, and failure codes remain in SPEC Sections
6, 8 through 10, and 14 through 18 and in the annotated withdrawal diagrams.
