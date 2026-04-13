# Boomerang Protocol Specification - WIP

Status: Research draft, not production ready.

Where the existing repository prose uses multiple names for the same concept, this document prefers the terminology already used in the Rust PoC repository at `bitryonix/boomerang`, especially names such as `boomerang_params`, `most_work_bitcoin_block_height`, `duress_placeholder`, and `reached_mystery_flag`.

## 1. Status Of This Document

Boomerang is a research-stage Bitcoin cold-storage protocol. This document specifies the intended protocol behavior, but it does not claim that all security assumptions have been validated or that all implementation details are final.

Several parts of the design remain intentionally open:

- exact parameter selection for mystery ranges, freshness windows, and duress-check cadence;
- reorg handling and chain-view disagreement policy;
- the final cryptographic profile for authenticated encryption, KDF usage, and serialization;
- Boomletwo activation and anti-clone semantics;
- timeout, blame, failover, and device-replacement ancillaries.

Open items are called out inline as `OPEN ISSUE`.

## 2. Conventions

The key words `MUST`, `MUST NOT`, `REQUIRED`, `SHOULD`, `SHOULD NOT`, and `MAY` in this document are to be interpreted as described in RFC 2119 and RFC 8174.

This document uses:

- `Peer`: one joint custodian in the Boomerang set.
- `User`: the human operator for a given peer.
- `Iso`: the peer's isolated, offline environment.
- `Niso`: the peer's online environment.
- `Boomlet`: the peer's active secure element.
- `Boomletwo`: the peer's backup secure element.
- `ST`: the Secure Terminal.
- `WT`: the Watchtower.
- `SAR`: the Search And Rescue service.
- `Phone`: the device used for SAR registration and dynamic doxing updates.

The current design profile assumes 5 peers and a 5-of-5 Boomerang regime. Generalization to other thresholds is possible in principle, but is not defined here.

## 3. Goals And Non-Goals

### 3.1 Goals

Boomerang is designed to:

- enforce a bounded but unpredictable withdrawal delay in the primary spending path;
- embed plausibly deniable duress signaling into the normal withdrawal flow;
- preserve eventual recoverability through deterministic fallback scripts;
- avoid changes to Bitcoin consensus rules.

### 3.2 Non-Goals

Boomerang does not attempt to:

- eliminate coercion risk;
- replace ordinary cold storage for typical users;
- provide production-ready hardware or operational guidance today;
- fully specify SAR real-world rescue procedures.

## 4. System Model

Each peer controls the following local components:

- a mnemonic-backed normal key used by `Iso`;
- a non-exportable Boomlet key share used by `Boomlet`;
- an `ST` used as the trusted UI for verification and duress answers;
- a `Niso` instance used for networking, Bitcoin RPC access, and coordination;
- a `Phone` used to register encrypted doxing data with SAR;
- an optional inactive `Boomletwo` backup.

The external environment includes:

- the Bitcoin network;
- a Bitcoin RPC endpoint visible to each `Niso`;
- the Tor network for peer and WT communication;
- one active WT;
- one SAR per peer.

The protocol trust model assumes:

- at least one honest peer;
- Boomlet private material is non-exportable;
- ST preserves display and input integrity;
- cryptographic primitives hold;
- WT and SAR are available for the duration of relevant ceremonies.

## 5. Security And Timing Model

Boomerang treats time predictability as an attack surface.

The protocol therefore has two on-chain spending regimes:

- a primary Boomerang regime that is only spendable after `milestone_block_0` and only once every Boomlet has reached its own secret threshold; and
- a deterministic fallback regime that becomes available later through milestone timelocks and gradually reduced normal-key thresholds.

The Boomerang regime provides bounded unpredictability. The fallback regime provides eventual liveness and recovery.

`OPEN ISSUE`: the exact timing constants, tolerated delays, and reorg policy are not yet final.

## 6. Actors And State

### 6.1 Per-Peer Long-Lived State

Each peer maintains, at minimum:

- `mnemonic`, `passphrase`, and derived `normal_privkey` in `Iso`;
- `boom_musig2_privkey_share` in `Boomlet`;
- a Boomlet identity keypair;
- an ST identity keypair;
- a Tor secret key used by `Niso`;
- `duress_consent_set` in `Boomlet`;
- `mystery`, `counter`, `last_seen_block`, and `reached_mystery_flag` in `Boomlet`;
- the peer's selected SAR identities and registration data;
- `boomerang_params`, including the descriptor-relevant shared parameters.

### 6.2 Shared State

All peers are expected to converge on the same:

- ordered peer identity set;
- active WT set;
- milestone schedule;
- `boomerang_descriptor`;
- transaction `tx_id` for a given withdrawal ceremony.

Any peer that detects divergence in these shared values MUST fail closed for the relevant ceremony.

## 7. Cryptographic Objects

### 7.1 Key Classes

Boomerang uses the following key classes:

- `normal key`: recoverable, mnemonic-backed, held by `Iso`;
- `boom key share`: generated inside Boomlet and not backed up in exportable form;
- `boom_pubkey`: the MuSig2 aggregate of the peer's normal key and Boomlet key share;
- Boomlet identity keypair: used for message signing and confidential peer-targeted payloads;
- ST identity keypair: used for Boomlet-to-ST authenticated exchanges;
- WT and SAR long-term identity keys.

### 7.2 Doxing Material

Each peer defines:

- `doxing_password`;
- `doxing_key = SHA256(doxing_password)`;
- `doxing_data_identifier = SHA256(doxing_key)`.

SAR stores encrypted static and dynamic doxing data indexed by `doxing_data_identifier`.

### 7.3 Encryption And Serialization

All encrypted protocol messages MUST provide confidentiality and integrity. All signed protocol messages MUST have a canonical serialization and MUST bind the fields that define their meaning.

`OPEN ISSUE`: the exact AEAD mode, KDF transcript structure, and canonical wire encoding remain to be fixed.

## 8. Descriptor

### 8.1 Overview

Funds are locked in a Taproot output with an unspendable key path and a script tree containing:

- one Boomerang branch using `boom_pubkey_i`; and
- a deterministic waterfall of normal-key branches activated by later milestones.

In the current design profile, the intended structure is:

```text
Unspendable key path
|
|--> and(thresh(5, pk(boom_pubkey_1)...pk(boom_pubkey_5)), after(milestone_block_0))
|
|--> and(thresh(5, pk(normal_key_1)...pk(normal_key_5)), after(milestone_block_1))
|
|--> and(thresh(4, pk(normal_key_1)...pk(normal_key_5)), after(milestone_block_2))
|
|--> and(thresh(3, pk(normal_key_1)...pk(normal_key_5)), after(milestone_block_3))
|
|--> and(thresh(2, pk(normal_key_1)...pk(normal_key_5)), after(milestone_block_4))
|
|--> and(thresh(1, pk(normal_key_1)...pk(normal_key_5)), after(milestone_block_5))
```

### 8.2 Descriptor Requirements

The descriptor MUST satisfy the following:

- `milestone_block_0 < milestone_block_1 < ... < milestone_block_5`;
- the first spendable branch MUST be the Boomerang branch;
- the Boomerang branch MUST require every `boom_pubkey_i` in the current 5-peer profile;
- the fallback branches MUST use only normal keys;
- the fallback threshold MUST weaken over time to preserve eventual recoverability.

Any implementation that constructs a different descriptor shape MUST treat that as a different protocol profile, not a silent variant of this one.

## 9. Setup Protocol

Setup establishes keys, shared parameters, duress state, SAR registration, and the Boomlet backup.

### 9.1 Setup Preconditions

Before setup completes:

- each peer MUST choose a network and milestone schedule;
- each peer MUST identify the WT set and SAR;
- each peer MUST prepare a secure out-of-band channel for peer identity exchange.

### 9.2 SAR Registration

For each peer:

1. `Phone` receives `doxing_password`, SAR identities, and static doxing data from `User`.
2. `Phone` computes `doxing_key` and `doxing_data_identifier`.
3. `Phone` registers `doxing_data_identifier` with SAR, receives payment instructions, and returns them to `User`.
4. After payment, `Phone` sends payment proof plus encrypted static and dynamic doxing data to SAR.
5. SAR stores the encrypted data associating it with `doxing_data_identifier` and acknowledges synchronization.

Dynamic doxing updates MAY continue after setup and SHOULD remain encrypted at the source by `doxing_key`.

### 9.3 Normal-Key And Boomlet Initialization

For each peer:

1. `Iso` derives the peer's normal key material from user-provided entropy and passphrase.
2. `Iso` installs the Boomlet applet onto an empty smart card.
3. During installation, `Iso` provides Boomlet with at least:
   - `normal_pubkey`;
   - `doxing_key`;
   - the selected SAR identities;
   - network information.
4. Boomlet generates:
   - its identity keypair;
   - its MuSig2 private share;
   - the aggregate `boom_pubkey`;
   - its Tor secret key or equivalent identity material needed by `Niso`;
   - an initial secret `mystery`.

The Boomlet private share and identity private key MUST remain non-exportable.

### 9.4 ST Pairing And Duress-Consent Enrollment

`Iso` mediates an authenticated pairing between Boomlet and ST.

The pairing flow MUST ensure that:

- Boomlet learns `st_identity_pubkey`;
- ST learns `boomlet_identity_pubkey`;
- later duress challenges are protected against replay with fresh nonces.

After pairing:

1. Boomlet generates a randomized `duress_check_space`.
2. ST maps challenge values to a stable human vocabulary. In the current design, the vocabulary is the ordered list of countries and the mapping is based on alphabetical rank.
3. `User` chooses a 5-element consent pattern.
4. ST returns the selected indices, bound to the Boomlet nonce.
5. Boomlet resolves those indices into a stored `duress_consent_set`.
6. A confirmation round repeats the process and MUST match the original consent set before setup continues.

The consent set is unordered and contains no repetition.

### 9.5 Peer Identity Exchange And Shared-Parameter Agreement

Each peer MUST exchange the following out of band:

- peer identity information;
- the Boomlet-signed Tor address or equivalent peer-reachability binding.

After this exchange:

- `Niso` and `Boomlet` MUST validate the signatures over all advertised peer addresses;
- each peer MUST verify inclusion of its own identity record in the peer set;
- all peers MUST converge on a common `boomerang_params_seed` containing, at minimum:
  - peer identities;
  - WT identities;
  - milestone schedule.

Each peer then derives or verifies the same `boomerang_descriptor` from the agreed parameters.

### 9.6 WT-Mediated Setup Finalization

After peer-parameter agreement:

- each peer's Boomlet MUST deliver the SAR-facing registration handle for that peer through WT;
- SAR MUST return a signed acknowledgement proving that the registered `doxing_data_identifier` exists and that the stored static data fingerprint matches;
- Boomlet MUST retain enough SAR acknowledgement data for later offline verification by `Iso`.

Peers MUST then exchange signed fingerprints proving that all peers reached the same setup-finalization state before the protocol is considered ready for backup.

### 9.7 Boomletwo Backup

The backup process MUST:

- install Boomletwo on a distinct empty smart card;
- require authorization by the peer's normal key;
- transfer Boomlet state encrypted for Boomletwo;
- exclude the active Boomlet `mystery` from the exported backup state;
- return a signed Boomletwo confirmation that the backup was imported.

After successful backup, the active Boomlet MUST mark itself as backed up and MUST reject repeated backup attempts for the same active state.

`OPEN ISSUE`: this document does not yet define the activation/deactivation protocol that guarantees only one of Boomlet and Boomletwo can be active.

### 9.8 Setup Success Conditions

Setup completes only when all of the following hold:

- the peer set and descriptor are agreed;
- SAR registration has been acknowledged;
- the Boomlet-ST pairing is complete;
- the consent set has been enrolled and confirmed;
- Boomlet backup completion has been acknowledged across all peers.

## 10. Withdrawal Protocol

Withdrawal is a coordinated, multi-phase ceremony.

### 10.1 Preconditions

A Boomerang-regime withdrawal may begin only if:

- the selected inputs belong to the Boomerang descriptor;
- `most_work_bitcoin_block_height >= milestone_block_0`;
- the transaction is represented as a PSBT whose `tx_id` can be derived and committed to;
- the PSBT remains satisfiable under the Boomerang branch.

If any peer determines that the transaction is attempting to use the fallback regime instead, that MUST be surfaced explicitly as a different path with different security properties.

### 10.2 Initiator Approval

The initiating peer performs the following:

1. `User` provides a PSBT to `Niso`.
2. `Niso` checks milestone eligibility and basic satisfiability, then forwards the PSBT plus the observed block height to Boomlet.
3. Boomlet derives `tx_id`, commits to it for the rest of the ceremony, and sends `tx_id + nonce` to ST through Niso.
4. ST shows `tx_id` to `User`.
5. `User` confirms that the displayed `tx_id` matches the intended withdrawal transaction.
6. ST signs the nonce-bound approval and returns it to Boomlet.
7. Boomlet verifies the ST approval and creates a signed initiator approval for WT.
8. Boomlet also encrypts the PSBT separately for each other peer's Boomlet.

The initiator approval MUST bind at least:

- the fact that the transaction is approved;
- the committed `tx_id`;
- the initiator's event block height.

### 10.3 Non-Initiator Review And Approval

For each non-initiator peer:

1. WT forwards:
   - the WT approval;
   - the initiator's signed approval;
   - the PSBT encrypted for that peer.
2. `Niso` and Boomlet validate:
   - WT signature;
   - initiator identity;
   - freshness against the local chain view;
   - `tx_id` consistency across approvals and PSBT.
3. Boomlet decrypts the PSBT and returns it to `Niso`.
4. `User` reviews the PSBT through `Niso`.
5. After `User` agrees, Boomlet repeats the ST-mediated `tx_id` confirmation flow.
6. Boomlet emits a peer approval for WT.

Every non-initiator approval MUST bind the same `tx_id`.

### 10.4 Commitment Phase

WT MUST NOT start the digging game until it has a valid approval from every peer for the same `tx_id`.

Once unanimous approval exists:

- every peer is committed to the same transaction;
- duress-capable messaging becomes mandatory for the rest of the ceremony;
- WT becomes the coordinator for the digging game.

### 10.5 Digging Game

The digging game is the non-deterministic phase of withdrawal.

Each Boomlet maintains:

- a secret `mystery`;
- a `counter`;
- a `last_seen_block`;
- a monotonic `ping_seq_num`;
- a monotonic `reached_mystery_flag`.

Each iteration proceeds as follows:

1. Boomlet evaluates whether to trigger a duress check.
2. Boomlet generates a `duress_placeholder`.
3. Boomlet emits a signed `ping` for WT that includes at least:
   - `"ping"` magic;
   - `tx_id`;
   - peer-local event block height;
   - `ping_seq_num`;
   - `reached_mystery_flag`.
4. WT validates freshness, sequence progression, and signature validity.
5. WT forwards the `duress_placeholder` to SAR and obtains a signed acknowledgement.
6. WT emits a signed `pong` to each peer that includes the current `most_work_bitcoin_block_height` and the latest validated pings from the other peers.
7. Each Boomlet validates the WT `pong`, validates the included peer pings, updates local reach-state, and conditionally increments its counter.

WT SHOULD ensure at least one new block is observed between successive effective `pong` updates so that counter advancement remains block-coupled rather than purely message-coupled.

### 10.6 Counter Advancement

A Boomlet MUST increment its counter only if all required freshness constraints hold. In the current design, those checks depend on:

- the Boomlet's last seen block;
- the local `Niso` event block height;
- the most recent peer pings carried by WT.

A Boomlet that has once set `reached_mystery_flag = 1` MUST NOT revert it to `0`.

The loop terminates when WT has collected a valid reached-state ping from every peer for the committed `tx_id`.

`OPEN ISSUE`: the exact freshness formula, jump limits, and reorg handling thresholds are not yet fixed in a final normative profile.

### 10.7 Hydration And Signing Readiness

After WT observes that all peers reached their mystery:

1. WT sends the terminal `reached_pings_collection` to each peer.
2. `Niso` verifies the collection and hydrates the PSBT for final signing.
3. Boomlet verifies that the hydrated PSBT still derives the same committed `tx_id`.

If the hydrated PSBT does not preserve the committed `tx_id`, the peer MUST abort the signing phase.

### 10.8 Final Signing

For each peer:

1. `User` physically moves Boomlet from `Niso` to `Iso`.
2. `Iso` reconstructs the normal-key signing context from `mnemonic`, `passphrase`, and network parameters.
3. Boomlet provides:
   - the hydrated PSBT;
   - the `boomerang_descriptor`;
   - its MuSig2 public share;
   - its public nonce.
4. `Iso` produces its own public nonce and normal-key partial signature.
5. Boomlet verifies the signing context, produces its partial signature, and combines state as required by the MuSig2 flow.
6. The resulting signed PSBT fragment is preserved both by `Iso` and Boomlet.
7. `User` moves Boomlet back to `Niso`.
8. Boomlet exports the signed PSBT fragment to `Niso`.

After exporting the signed PSBT fragment, Boomlet MUST:

- clear withdrawal-specific state for the completed ceremony;
- regenerate a fresh `mystery` for future withdrawals.

### 10.9 WT Aggregation And Broadcast

WT collects the signed PSBT fragments from every peer, aggregates them into a fully signed transaction, and broadcasts the resulting transaction to the Bitcoin network.

The broadcast transaction MUST correspond to the originally committed `tx_id`.

## 11. Duress Signaling

### 11.1 Scope

The duress mechanism is intended to cover:

- being forced to initiate a withdrawal;
- being forced to continue a withdrawal after commitment.

It does not currently claim to cover setup.

### 11.2 Consent Pattern

At setup, the user chooses a 5-element consent pattern from the displayed challenge vocabulary. In the current design:

- the vocabulary is the list of countries;
- the choice is represented internally as a 5-element set of numbers in `[1, 195]`;
- order does not matter;
- repetition is not allowed.

### 11.3 Duress Challenges

At withdrawal commitment and at random intervals during the digging game:

1. Boomlet generates a fresh challenge space and nonce.
2. ST renders the challenge to the user.
3. If the user is safe, the user reproduces the consent set.
4. If the user is under duress, the user selects any different set.
5. ST returns the selected indices bound to the Boomlet nonce.
6. Boomlet resolves the indices and compares the resulting set to `duress_consent_set`.

If the resulting set equals `duress_consent_set`, the answer is interpreted as safe. Otherwise it is interpreted as duress.

### 11.4 Duress Placeholder Rules

Every post-commitment message path that carries a duress-bearing payload MUST preserve the same observable control flow regardless of whether the user is safe or in duress.

Specifically:

- safe case: Boomlet encrypts zero-like padding or equivalent inert content for SAR;
- duress case: Boomlet encrypts `doxing_key` or equivalent unlock material for SAR.

WT MUST forward the placeholder to SAR in both cases. SAR MUST respond in both cases. The protocol flow visible to peers and WT MUST NOT diverge based on the duress outcome.

### 11.5 SAR Handling

On receiving a forwarded `duress_placeholder`, SAR:

1. decrypts the placeholder;
2. treats all-zero or inert content as no-duress;
3. treats any non-zero valid unlock material as a duress signal;
4. if duress is detected, derives `doxing_data_identifier` from the unlock material, retrieves the user's encrypted doxing data, and begins SAR operations;
5. returns a signed acknowledgement to WT for delivery back to the originating Boomlet.

Boomlet SHOULD require the SAR acknowledgement before considering the placeholder delivered.

## 12. Freshness, Replay Resistance, And Binding

The following invariants apply across the protocol:

- every ST-mediated approval MUST be nonce-bound;
- every peer approval MUST bind the committed `tx_id`;
- every ping and pong MUST bind the committed `tx_id`;
- `ping_seq_num` MUST increase monotonically per withdrawal ceremony;
- `reached_mystery_flag` MUST be monotonic;
- peer identities and Tor addresses MUST be signed and verified before use;
- the same withdrawal ceremony MUST NOT silently migrate to a different PSBT.

`OPEN ISSUE`: setup-instance uniqueness and transcript-wide binding are not fully closed yet. Future revisions should add explicit setup IDs and stronger transcript hashes.

## 13. Failure Behavior

Unless a more specific recovery path is defined, peers MUST fail closed in the Boomerang regime when they observe:

- descriptor mismatch;
- peer identity mismatch;
- WT or SAR signature failure;
- `tx_id` mismatch;
- stale or replayed ST responses;
- stale or replayed pings;
- invalid reached-state transitions;
- hydrated-PSBT mismatch at signing time.

Fail-closed behavior in this context means refusing to advance the current ceremony. It does not imply automatic movement to deterministic fallback.

`OPEN ISSUE`: timeout, blame, and service-failover procedures remain ancillary work.

## 14. Security Considerations

Boomerang's security depends on several strong assumptions:

- Boomlet actually protects its private share, internal state, RNG, and lifecycle state.
- ST actually preserves display and input integrity and is sufficiently tamper-evident.
- WT and SAR are not custody actors, but they are still security-critical coordination actors.
- Users must roll funds into a fresh Boomerang setup before deterministic fallback makes coercion timing predictable.
- The protocol is metadata-heavy relative to ordinary cold storage and can increase targeting risk if deployed poorly.

Known design risks include:

- forced determinism if rollover or peer cooperation fails;
- incomplete reorg policy;
- incomplete cryptographic profile;
- unspecified Boomletwo activation semantics;
- hardware trust assumptions that exceed what a prototype device automatically guarantees.

## 15. Canonicality And Companion Documents

This document is intended to be the repository's canonical protocol narrative.

The following companion documents remain important:

- `README.md` for project overview;
- `DEEPDIVE.md` for motivation, economics, and design rationale;
- `setup/README.md` and `withdrawal/README.md` for stepwise message detail;
- `duress_protection/README.md` for the duress design rationale;
- `security_models/` for threat analysis and design gaps;
- the Rust PoC repository for implementation-oriented naming and execution structure.

If a companion document conflicts with this specification on intended protocol behavior, this specification SHOULD win and the companion document SHOULD be updated.

## 16. TODO For A Complete RFC-Style Specification

The following work remains before this document can be considered a complete RFC-style protocol specification:

- resolve the current repo-level discrepancies and then update all companion documents to match the reconciled text;
- reconcile the duress-challenge model across `setup/README.md`, `withdrawal/README.md`, `duress_protection/README.md`, and this specification;
- reconcile canonical-source wording across the repository so older files no longer claim authority over this specification;
- reconcile fallback-regime wording so all documents agree whether the deterministic regime is normal-key-only;
- reconcile WT terminology so all documents consistently describe the relationship between the configured WT set and the active WT used for a ceremony;
- reconcile mystery-generation wording so overview documents, setup documents, and this specification all describe the same timing and parameterization;
- define one final canonical wire format for every message, including field names, encoding rules, versioning, and size limits;
- define the exact cryptographic profile, including AEAD mode, KDF/HKDF contexts, nonce construction, transcript binding, and signature domains;
- define one final duress-challenge presentation model and align setup, withdrawal, and duress companion docs to it;
- define the exact policy for selecting `MIN_TRIES_FOR_DIGGING_GAME_IN_BLOCKS`, `MAX_TRIES_FOR_DIGGING_GAME_IN_BLOCKS`, `DURESS_CHECK_INTERVAL_IN_BLOCKS`, and all freshness tolerances;
- define reorg handling, divergent chain-view handling, and the trust model for Bitcoin RPC sources;
- define the complete state machine for setup, withdrawal, commit, digging game, signing, abort, and reset transitions;
- define timeout, retry, blame, and recovery behavior for failed or stalled ceremonies;
- define WT redundancy, active-WT selection, failover, and quorum behavior if multiple WTs are supported;
- define SAR redundancy, acknowledgement semantics, deduplication rules, and failure handling;
- define Boomletwo activation, deactivation, anti-clone, and revocation semantics so only one active signing authority exists at a time;
- define setup-instance uniqueness and transcript identifiers to close replay and cross-ceremony confusion gaps;
- define a normative PSBT hydration policy, including what may change after commitment and what MUST remain frozen;
- define canonical error codes or failure classes so independent implementations fail in interoperable ways;
- define security and privacy considerations in a more RFC-complete manner, including metadata leakage, side channels, and service trust assumptions;
- add implementation guidance appendices with worked examples, message transcripts, and descriptor examples;
- add conformance material such as test vectors, interop fixtures, and a checklist for compliant implementations;
- decide whether the final document needs explicit registries, extension points, or an "IANA considerations" section, even if the answer is "none";
- align every companion document in this repository so no older file contradicts the final normative text.
