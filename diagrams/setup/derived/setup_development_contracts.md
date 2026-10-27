# Setup development contracts

This file supplies the compact contracts referenced by
[`setup_protocol_development.puml`](setup_protocol_development.puml).
It is a protocol-development aid. Normative requirements remain in
[`SPEC.md`](../../spec/SPEC.md), especially Sections 9, 12, 13, 17, and 18.

## Notation

| Form | Meaning |
| --- | --- |
| `Sig_X(M)` | Domain-separated signature by actor `X` over the canonical encoding of `M`, including the active protocol version. |
| `Enc_X(M)` | Authenticated encryption for `X` under the directional channel and exact context required by the specification. |
| `H(fields)` | The specification-defined tagged hash over canonical typed fields. |
| `checkpoint(sid, phase, previous)` | The setup checkpoint update bound to `sid`, the exact phase label, and the predecessor checkpoint. |
| `via Niso` or `via Iso` | The named host transports the object. Transport does not grant authority or replace endpoint verification. |
| `x4` or `x5` | One object from each indicated active setup role, with expected membership, uniqueness, and canonical ordering enforced. |
| `G-NAME` | The complete acceptance guard defined below. |

All guards include canonical schema, size, protocol-version, expected phase,
expected sender, cryptographic verification, and replay checks when those checks
apply. An invalid guard result stalls the setup attempt under SPEC Section 18.

## Setup state contract

| Transition | Required local proof |
| --- | --- |
| `EMPTY -> INSTALLED` | Successful key generation, aggregate-key validation, and applet initialization on an empty Boomlet. |
| `INSTALLED -> ST_ENROLLED` | Two fresh nonce-bound consent rounds resolve to the same five-element set. |
| `ST_ENROLLED -> PARAMS_REVIEWED` | Boomlet verifies ST's signature over the exact outstanding nonce-bound setup commitment. |
| `PARAMS_REVIEWED -> PARAMETERS_AGREED` | Boomlet verifies one identical signed `boomerang_params_fingerprint` from every peer. |
| `PARAMETERS_AGREED -> WT_READY` | Boomlet verifies its local `WtSetupReceipt` and the five matching `wt_ready` checkpoint signatures. |
| `WT_READY -> SAR_READY` | Boomlet verifies the WT wrapper, its own SAR response, and the five matching `sar_ready` checkpoint signatures. |
| `SAR_READY -> BACKUP_READY` | Boomlet verifies `BackupDone`, the five matching `backup_ready` checkpoint signatures, and durably stores the final checkpoint. |

## Semantic objects

| Object | Required semantic content and binding | Protection and consumer |
| --- | --- | --- |
| SAR stored data | Static envelope plus append-only `DynamicRescueUpload` histories, `PROTOCOL_VERSION`, and signed receipts. | Static rescue data uses the SAR-scoped root; dynamic rescue data uses device-derived keys. `dynamic_update_auth_key` authenticates uploads before rescue-key release. |
| `PeerSetupRecord_i` | `PeerId_i`, fresh `peer_setup_nonce_i`, and peer Tor address. | Signed by Boomlet `i`; Niso, ST, peers, Boomlets, and WT verify it for their own operations. |
| `setup_instance_id` | `H(PROTOCOL_VERSION, canonical BoomerangParamsSeed)`. The seed includes the five ordered signed peer records, WT preference order, and milestones. | Recomputed by Boomlet and ST. It binds later setup objects and withdrawals. |
| Setup review commitment | The nonce-bound setup commitment required by SPEC Section 13.6. | Encrypted Boomlet to ST, signed by ST after user review, then encrypted ST to Boomlet. |
| `boomerang_params_fingerprint` | Hash of the exact `BoomerangParams`, including setup ID, peer order, WT order, milestones, and descriptor. | Every Boomlet signs the identical fingerprint. |
| `WtSetupReceipt` | Local `setup_instance_id` and common `boomerang_params_fingerprint`. | Signed by the selected WT for the specific peer registration. |
| `SarSetupResponse` | Setup ID, doxing identifier, and the static rescue-data ciphertext commitment required by SPEC Section 13.9. | Signed by the setup-bound SAR, encrypted for the peer Boomlet, and carried in a WT-signed wrapper. |
| `BoomletBackupRequest` | Target Boomletwo public key and reconstructed normal public key. | Signed by the normal private key and accepted once by the active Boomlet. |
| `BoomletBackupState` | Authorized long-lived state, including the active `setup_instance_id`; it excludes any withdrawal mystery. | Authenticated and encrypted from active Boomlet to the named Boomletwo target. |
| `BackupDone` | Setup ID, target Boomletwo key, and source Boomlet identity. | Signed by Boomletwo after successful authenticated import. |
| Setup checkpoint | `checkpoint(setup_instance_id, phase, previous_checkpoint)`. | Signed by every Boomlet for `wt_ready`, `sar_ready`, and `backup_ready`; every receiver requires the same value and expected signers. |

## Acceptance guards

### Registration and enrollment

| Guard | Owner | Acceptance conditions |
| --- | --- | --- |
| `G-SAR-REGISTRATION` | Phone and SAR | Confidential exchange authenticates the selected `SarId`. Payment verifies against the SAR invoice associated with `doxing_data_identifier` and `PROTOCOL_VERSION`. Require `first_dynamic_doxing_data_upload.doxing_data_identifier == doxing_data_identifier`, a valid upload CMAC, and sequence number zero. An existing `dynamic_update_auth_key` must match in constant time. SAR commits the static envelope, `PROTOCOL_VERSION`, `dynamic_update_auth_key`, `first_dynamic_doxing_data_upload`, and `dynamic_doxing_data_upload_receipt_signed_by_sar` before `SetupSarPhoneMessage2`. Phone verifies the receipt under SPEC Section 10 before success. |
| `G-SETUP-CONSENT-RESPONSE` | ST and Boomlet | ST accepts one displayed value from each of five columns. Boomlet requires an outstanding challenge for the current enrollment round, exact nonce equality, exactly five distinct indices in range, and one response only. The confirmation round must resolve to the first stored set. |
| `G-LOCAL-PEER-RECORD` | Niso and ST | Boomlet signature and identity are valid; the Tor address derived by Niso equals the address inside the record; ST sees the Boomlet identity established during pairing. |

### Parameter review and agreement

| Guard | Owner | Acceptance conditions |
| --- | --- | --- |
| `G-SETUP-SEED` | Niso and Boomlet | Exactly five valid signed peer records; exact local record occurs once; identities are unique and strictly ordered by canonical Boomlet identity bytes; WT preferences and milestones are canonical and valid; Boomlet recomputes the seed and setup ID. |
| `G-PARAMS-REVIEW` | ST | Authenticated outer and decrypted setup IDs agree; nonce-bound commitment is valid; seed fields are canonical; recomputation matches the commitment; displayed peer order, WT preference order, milestones, and protocol version are the committed values. |
| `G-PARAMS-REVIEW-APPROVAL` | Boomlet | ST signature is valid and covers the exact outstanding nonce-bound setup commitment under the expected setup ID and phase. |
| `G-SETUP-AGREEMENT` | Niso and Boomlet | Exactly one signature from every active Boomlet; all signers are expected and unique; every signed fingerprint equals the locally computed fingerprint; peer order and descriptor are the committed values. |

### Service activation and backup

| Guard | Owner | Acceptance conditions |
| --- | --- | --- |
| `G-WT-REGISTRATION` | WT | Exactly five setup registrations; each signed peer record corresponds to its signed params fingerprint; setup IDs agree; every signature and signer membership check passes. |
| `G-WT-SETUP-RECEIPT` | Niso and Boomlet | WT signature and identity are valid; receipt contains the active setup ID and exact common params fingerprint; payment and registration phase are complete. Boomlet revalidates independently. |
| `G-SETUP-CHECKPOINT` | Niso and Boomlet | Phase label and predecessor produce the locally expected checkpoint; exactly one valid signature from every active Boomlet; no missing or duplicate signer; all signed contents are identical. Boomlet revalidates independently before transition. |
| `G-SAR-BINDING` | WT | Boomlet signature is valid; binding contains the active setup ID and the peer's selected `SarId`; the encrypted identifier payload is forwarded only to that identity. |
| `G-SAR-SETUP-RESPONSE` | Niso and Boomlet | WT wrapper signature, setup ID, and status are exact; SAR response decrypts under the selected SAR channel; SAR signature, setup ID, doxing identifier, and local expected identifier all match. Boomlet revalidates independently. |
| `G-BACKUP-REQUEST` | Boomlet | Normal-key signature verifies against stored `normal_pubkey`; request names the intended target key and same normal public key; no backup was completed for this active state; source state is eligible for backup. |
| `G-BACKUP-PACKAGE` | Iso | Supplied milestones equal `boomerang_params`; reconstructed descriptor and output key match; active Boomlet identity occurs exactly once; SAR signature and setup ID verify; derived doxing identifier and reconstructed static ciphertext commitment match the SAR response. |
| `G-BACKUP-DONE` | Boomlet | Boomletwo signature is valid; setup ID equals the active setup; target key equals the authorized target; source key equals the active Boomlet; imported state was acknowledged once. |

## Setup invariants retained by the compact diagram

1. Private Boomlet identity and MuSig2 material never becomes plaintext host data.
2. Consent enrollment requires two independent fresh challenges resolving to one equal set.
3. Every peer derives the same setup ID from the same canonical seed.
4. User review covers the peer order, WT preference order, milestones, and protocol version committed by Boomlet.
5. Parameter agreement requires all five Boomlet signatures over one exact fingerprint.
6. Each checkpoint advances from its exact predecessor and contains only the common phase value. Peer-local service receipts remain local evidence.
7. WT routes SAR setup only to the peer's setup-bound SAR.
8. Backup is authorized by the normal key, bound to one target Boomletwo, includes the active setup ID, and excludes withdrawal mystery state.
9. Setup success follows durable storage of the final checkpoint.

## Failure and retry contract

- A failed guard stalls the active setup attempt while retaining its setup
  binding and replay state.
- A service failure cannot be bypassed by advancing the checkpoint chain.
- A retry may retransmit an identical authenticated object where SPEC permits
  it. A nonce, setup record, receipt, or checkpoint cannot be repurposed for a
  different phase or setup instance.
- Restart from installation requires explicit abandonment and a fresh
  `peer_setup_nonce`. Completed setup IDs remain in replay memory.
- Private keys, imported backup state, and checkpoint storage follow the
  fail-closed behavior in SPEC Section 18.

## Detailed coverage

| Development fragment | Detailed setup steps | Normative source |
| --- | --- | --- |
| `S-SAR` | 1 through 8 | SPEC Section 13.1 |
| `S-INSTALL`, `S-PAIR`, `S-CONSENT` | 9 through 27 | SPEC Sections 13.2, 13.3, and 16.1 |
| `S-PARAMS`, `S-REVIEW`, `S-AGREE` | 28 through 45 | SPEC Sections 13.4 through 13.7 |
| `S-WT` | 48 through 60 | SPEC Section 13.8 |
| `S-SAR-ACT` | 61 through 71 | SPEC Section 13.9 |
| `S-BACKUP`, `S-DONE` | 72 through 94 | SPEC Sections 13.10 and 13.11 |

Pure host forwarding inside a named authenticated endpoint channel is collapsed
into `via Niso` or `via Iso`. Message construction formulas, domain strings,
KDF inputs, envelope contexts, canonical field order, and exact failure codes
remain in SPEC Sections 8 through 10, 17, and 18 and in the annotated setup
diagram.
