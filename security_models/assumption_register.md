# Boomerang threat-model assumptions and design gaps

> **Last change — 2026-11-16:** updated assumptions for rescue-data history, consent rotation, optional backup and recovery; separated specified controls and retired entries.

Boomerang's security claims depend on protocol, hardware and operational
assumptions. `SPEC.md` is authoritative for protocol behavior.

## Security posture

The protocol does not enforce Boomlet behavior, ST integrity, peer cooperation,
WT and SAR availability, or timely rollover. Setup and withdrawal IDs, chained
checkpoints, directional channel contexts, approval-set attestations, and exact
SAR acknowledgements bind messages to the intended setup and ceremony.

Freshness checks bind accepted messages to the active ceremony, sequence and
chain view. Rules for resuming after a chain reorg and retaining withdrawal
transcripts are unspecified.
Boomletwo activation, Niso/ST replacement and joint parameter constraints remain
unspecified. Physical rescue and timely rollover remain operational dependencies.

## Assumption register

### A. Baseline security scope and trust boundaries

| ID | Assumption | Sources | If False | Status |
| --- | --- | --- | --- | --- |
| AR-03 | Security claims apply to the exactly five-peer, 5-of-5 Boomerang profile. | `SPEC`, `R`, `D`, `SM` | Claims about one honest peer preserving Boomerang guarantees would not generalize. | Explicit |
| AR-04 | Implementations preserve the protocol's trust boundaries and message semantics. | `R`, `D` | An implementation may satisfy a different security model while claiming protocol conformance. | Implicit |

### B. Bitcoin, descriptor, and timing model

| ID | Assumption | Sources | If False | Status |
| --- | --- | --- | --- | --- |
| AR-06 | Bitcoin consensus, timelocks, and Taproot spending conditions behave as modeled by the descriptor design. | `R`, `D`, `SM` | Both non-deterministic and fallback security claims may be wrong. | Explicit |
| AR-07 | Milestones are strictly increasing and provide enough remaining runway at every deposit. | `SPEC`, `SM` | Late deposits inherit old absolute heights; late rollover can expose fallback early. | Explicit structure; open policy |
| AR-08 | Withdrawal starts only after `milestone_block_0`, and every component evaluates that gate against an acceptable chain view. | `SPEC` | Protocol state can diverge or enter unintended pre-milestone behavior. | Explicit |
| AR-09 | Users will roll funds into a new Boomerang setup before deterministic fallback makes coercion timing predictable. | `FD`, `R`, `D` | Boomerang degrades into a known-timeline withdrawal system. | Explicit |
| AR-10 | Stalling on local or WT height decrease and material RPC/WT disagreement is sufficient until a reorg-recovery policy exists. | `SPEC`, `SM` | Freshness checks, milestone gating, and counter progression can be evaluated on stale or reversible chain state, or remain stalled indefinitely. | Open |
| AR-11 | Niso and WT obtain accurate enough block height and chain context from their configured Bitcoin node(s). | `R`, `SU`, `WD`, `SM` | A bad node can skew timing, approvals, and liveness decisions. | Implicit |
| AR-12 | Fee estimation and PSBT hydration remain viable even after a possibly long withdrawal ceremony. | `WD`, `D` | A valid intent may become economically or practically unsignable by the time signing starts. | Implicit |
| AR-13 | The ping-pong "digging game" usually terminates before deterministic fallback if users start in time. | `R`, `D`, `FD`, `BD` | The main coercion-resistance promise collapses into forced determinism. | Open |
| AR-14 | Freshness tolerances, catch-up jumps, WT spacing and service timeouts jointly permit honest progress. | `SPEC`, `SM` | Honest lag may never close, or WT spacing violations may credit already elapsed blocks in a burst. | Open |

### C. Cryptography, serialization, and binding

| ID | Assumption | Sources | If False | Status |
| --- | --- | --- | --- | --- |
| AR-15 | Schnorr, MuSig2, ECDH, tagged hashing, SP 800-108 CMAC KDF, AES-256-CBC/PKCS#7, and AES-CMAC encrypt-then-MAC are secure as specified and implemented. | `SPEC`, `R`, `D`, `SU`, `WD`, `ADR` | Core funds, identity, and duress guarantees fail. | Explicit |
| AR-16 | Every component serializes and interprets the same canonical message content identically. | `SPEC`, `SU`, `WD`, `SP`, `WI`, `WN` | Signatures, CMACs, and identifiers can validate over mismatched semantics, or peers can disagree about what was approved. | Implicit |
| AR-17 | RNG quality is sufficient across Boomlet, ST, WT, SAR, and supporting hardware. | `R`, `D`, `ST`, `HW-H7` | Mystery, nonces, keys, and duress spaces can be biased or predictable. | Explicit |
| AR-18 | Nonces, setup and withdrawal scope IDs, chained setup checkpoints, sequence numbers, strict state transitions, freshness checks, and SAR replay tuples are sufficient replay protection for the message set. | `SPEC`, `ADR` | Old approvals, checks, placeholders, or commitments may be replayed into new contexts. | Explicit |
| AR-19 | `tx_id`, `withdrawal_id`, `approved_withdrawal_id`, approval-set attestations, signing-package verification, and final transaction revalidation are strong enough to anchor transaction-authorization correctness throughout the ceremony. | `SPEC`, `WD`, `WI`, `WN`, `D` | A participant may believe they are approving one spend while another spend is actually signed. | Explicit |
| AR-20 | Fresh CBC IVs, context-bound placeholders, status-free acknowledgements, a fixed release deadline, the same bounded durable-write path, and uniform retry/failure behavior make valid safe and duress traffic indistinguishable within the specified protocol surface. | `SPEC` | WT or an observer can infer duress from ciphertext, timing, storage, queue, log, metric, or failure behavior. | Explicit |
| AR-21 | `doxing_key_for_sar -> doxing_data_identifier` is unique enough operationally to locate the right setup-bound SAR record without ambiguity. | `SPEC`, `SU`, `SM` | SAR may fail to rescue the right user or may treat malformed inputs incorrectly. | Implicit |

### D. Boomlet and Boomletwo

| ID | Assumption | Sources | If False | Status |
| --- | --- | --- | --- | --- |
| AR-22 | Boomlet private key shares and identity keys remain host-inaccessible and are never exportable in plaintext; the only permitted export is an authenticated envelope bound to the authorized Boomletwo target. | `R`, `SM`, `D` | The Boomerang regime can be bypassed or cloned. | Explicit |
| AR-23 | Boomlet blocks direct host access to `mystery` and protects active-withdrawal state against rollback, cloning and updates outside the specified transitions. | `SPEC`, `SM`, `FD` | Current-ceremony timing becomes knowable or controllable by an attacker. | Explicit boundary |
| AR-24 | Boomlet faithfully enforces one-way state such as "reached mystery" and does not regress after a flag is set. | `WD`, `WI`, `WN` | Peers can be desynchronized or coerced into incorrect liveness behavior. | Implicit |
| AR-25 | Boomlet preserves the active ceremony through stalls and retries, never regenerates its mystery, and requires the SAR receipt for the final ping's placeholder and any required Boomletwo reset before clearing an approved withdrawal under Section 16.7. | `SPEC`, `ADR` | The ceremony can fail midstream, reuse a disclosed threshold, or leak state into a later withdrawal. | Explicit |
| AR-26 | Java Card-class hardware can perform the required cryptography and state updates within practical time and endurance limits. | `D`, `FD` | The protocol becomes too slow, too fragile, or impossible to execute as designed. | Hardware assumption |
| AR-28 | Only one of Boomlet and Boomletwo will ever be active, although activation/deactivation is unspecified. | `R`, `SM`, `D` | Duplicate active devices or confused ownership can undermine both safety and liveness. | Open |
| AR-29 | Device-loss risk is acceptable for each peer's backup choice; Boomletwo activation remains undefined. | `SPEC`, `FD`, `D` | Without a backup, one lost Boomlet can block the primary path; normal-key recovery cannot restore its share. Backup import alone provides no recovery guarantee. | Operational assumption |
| AR-30 | Secure-element supply chain, lifecycle, and side-channel properties are acceptable for a Bitcoin custody system. | `R`, `D`, `SM` | A hidden hardware weakness can destroy the central Boomerang guarantee. | Explicit |

### E. Secure Terminal and human interface

| ID | Assumption | Sources | If False | Status |
| --- | --- | --- | --- | --- |
| AR-31 | ST is the trusted UI for verification and duress input; Niso is not. | `R`, `ST`, `SM` | Duress signaling and user verification reduce to whatever the online host shows. | Explicit |
| AR-32 | ST securely generates, stores, and uses its long-term keys. | `ST`, `SP` | An attacker can read or alter the ST-Boomlet channel. | Explicit |
| AR-33 | ST displays exact Boomlet-authored content, records user intent, and performs rotation selection and matching confirmation locally. | `SPEC`, `ST`, `WI`, `WN` | Users can be tricked into false safe or false duress outcomes. | Explicit |
| AR-34 | ST remains air-gapped except for the intended QR exchange path. | `R`, `ST` | Malware-capable connectivity can directly exfiltrate consent-set or duress data. | Explicit |
| AR-35 | An attacker cannot practically observe both the ST display and the user's selection behavior well enough to learn the consent set under coercion. | `ST` | Plausible deniability degrades, and attackers can demand the true safe response. | Explicit |
| AR-36 | Users can memorize the consent set and reliably reproduce it during legitimate safe use, including under stress and fatigue. | `SPEC` | Valid but mistaken selections trigger false duress and unnecessary rescue escalation. | Implicit |
| AR-37 | One ST per peer is enough for ongoing operation, although ST replacement remains ancillary. | `D` | ST loss or compromise becomes an operational dead end. | Open; ST replacement remains undefined (DG-10). |
| AR-38 | ST hardware and software provide the required primitives, key protection and trusted input and display. | `ST`, `HW-BB`, `HW-H7` | Missing primitives prevent protocol execution; exposed keys or untrusted input defeat the ST boundary. | Hardware assumption |
| AR-39 | ST boot, debug and peripheral interfaces cannot bypass key isolation or run untrusted code. | `HW-BB`, `HW-H7`, `ST` | Untrusted code or peripherals can read keys or falsify ST input and display. | Hardware assumption |

### F. Iso, Niso, and phone environments

| ID | Assumption | Sources | If False | Status |
| --- | --- | --- | --- | --- |
| AR-40 | Iso remains trusted and isolated throughout installation, setup, Boomletwo backup, and final signing; compromise or substitution during setup is excluded. | `SPEC`, `SM` | An attacker can install a chosen `normal_pubkey`, `doxing_key`, or SAR, steal mnemonic material, or authorize a malicious backup target. | Explicit |
| AR-41 | User entropy, BIP39/BIP32 derivation, mnemonic handling, and passphrase handling on Iso are correct and secret. | `SPEC` | Normal-key security collapses independently of Boomlet. | Implicit |
| AR-42 | Niso may be malicious without authorizing a spend because Boomlet and ST verify critical state; Niso can still censor, misroute, corrupt presentation, or supply unsafe chain data. | `SPEC`, `SM` | The online host can gain more authorization power than the protocol assigns it. | Explicit boundary |
| AR-43 | Niso's RPC endpoint is honest, reachable, and configured correctly. | `R`, `SU`, `WD`, `SM` | Block height, satisfiability, and freshness decisions can be manipulated. | Implicit |
| AR-44 | Phone OS/app security is good enough for dynamic doxing collection and SAR registration. | `R`, `D`, `SU` | Rescue data can be stale, forged, leaked, or suppressed. | Explicit |
| AR-45 | Phone authenticates the selected SAR onion destination and uses confidential registration; SAR binds payment, atomically commits registration and returns a verifiable signed upload receipt. | `SPEC` | The user can think they are covered while SAR never received valid rescue data. | Explicit |
| AR-45A | `doxing_password` is user-chosen; Boomlet receives `doxing_key` and does not enforce password policy. | `SPEC`, `ADR`, `SM` | Setup or backup may depend on a second mnemonic-length secret, or components may diverge on password-policy enforcement. | Explicit |
| AR-46 | Users can safely shuttle Boomlet between Iso and Niso without plugging the wrong device into the wrong host or mixing peer hardware. | `SU`, `WD`, `D` | Cross-device confusion can undermine both safety and liveness. | Implicit |
| AR-47 | Phone, Niso and ST replacement preserves setup-bound state and identities. | `SPEC`, `D` | Device loss or replacement can make the system unsafe or unusable. | Phone replacement specified in bf74a71241c3c3e5f39ec5cdd5108f031ebc4639 (SPEC Section 13.1); Niso and ST replacement remain open (DG-10). |

### G. Peers, users, and human operations

| ID | Assumption | Sources | If False | Status |
| --- | --- | --- | --- | --- |
| AR-48 | At least one honest peer exists in the N-of-N Boomerang regime. | `R`, `SM`, `D` | The "one honest peer preserves the promise" argument disappears. | Explicit |
| AR-49 | Peers honestly exchange and verify peer IDs and Tor addresses out of band. | `R`, `SU`, `SP`, `SM` | Impersonation or routing attacks can be injected before Tor is even used. | Explicit |
| AR-50 | Users and peers meaningfully verify ordered peer setup records, WT order, milestone blocks, `setup_instance_id`, transaction identifiers, and peer data when prompted. | `SPEC`, `SU`, `WD`, `SP`, `WI`, `WN` | Human checks become ceremonial only, leaving host-mediated substitution attacks alive. | Implicit |
| AR-51 | Peers remain available across randomized delays, recurring duress checks, and long withdrawal timelines. | `D`, `FD`, `BD` | Honest liveness fails under normal operational conditions. | Implicit |
| AR-52 | Non-cooperation is rare enough that N-of-N remains acceptable despite known liveness cost. | `R`, `FD`, `D` | The system regularly falls back to deterministic recovery or deadlock. | Explicit |
| AR-53 | Fallback security requires attackers to hold fewer normal keys than the active branch threshold. | `SPEC`, `FD`, `D` | Once its milestone opens, the fallback branch permits a spend by anyone meeting its threshold. | Explicit |
| AR-54 | Users protect normal-key recovery material and any optional Boomletwo, track actual backup presence, heed rollover timing and follow operational guidance. | `SPEC`, `R`, `FD`, `D` | A completed setup checkpoint does not establish a Boomletwo copy; normal-key backups cannot restore Boomlet's share. Security depends on operator discipline. | Explicit |

### H. WT and SAR service assumptions

| ID | Assumption | Sources | If False | Status |
| --- | --- | --- | --- | --- |
| AR-55 | The one active WT stays available and responsive for the entire setup or withdrawal ceremony. | `SPEC`, `SM` | Coordination and counter progression stop even when peers are honest. | Explicit |
| AR-56 | WT signs only correct protocol statements and forwards messages without selective censorship or bias. | `WD`, `WI`, `WN`, `SM` | A non-custodial service can force desynchronization or liveness failure. | Implicit |
| AR-57 | WT's block-height view is trustworthy enough to act as the protocol heartbeat. | `R`, `D`, `SM`, `WD` | Freshness windows and progress rules rest on an attacker-controlled clock. | Explicit |
| AR-58 | WT switching and failover require procedures that preserve setup-bound identities and ceremony state. | `SPEC`, `D` | An outage stalls the primary path, or a switch breaks setup binding. | Open; WT switching and failover remain undefined (DG-12). |
| AR-59 | SAR protects rescue envelopes, metadata and the upload key and preserves accepted history and original receipts through supported storage failures. | `SPEC`, `R`, `SM`, `SU`, `ADR` | Leaked artifacts enable password guessing; incomplete restoration or finite capacity can lose rescue evidence. | Explicit |
| AR-60 | Each setup-bound SAR classifies valid placeholders, commits the same-shaped durable record by the fixed deadline, handles repeats idempotently, and signs the exact encrypted placeholder. | `SPEC`, `SM` | Positive duress may be missed, observable processing may reveal it, or Boomlet may continue under false delivery assumptions. | Explicit |
| AR-61 | Effective, lawful, correctly directed and non-escalatory SAR action requires correct payload interpretation and incident handling; protocol acknowledgement guarantees neither. | `R`, `D`, `SM` | Signaling can succeed while rescue is misdirected, delayed, duplicated or harmful. | Explicit |
| AR-62 | Reputation and SAR operator selection are an acceptable control for WT and SAR social trust. | `D`, `SM` | Operational trust remains an assumption rather than a designed control. | Explicit |
| AR-63 | SAR will not become a later attacker after identity is revealed during a rescue event. | `D`, `SM` | Duress rescue itself may create a future targeting or coercion channel. | Implicit |
| AR-77 | SAR has a reliable monotonic clock, a fixed positive `sar_placeholder_ack_delay` longer than bounded worst-case pre-acknowledgement work, and a matching WT timeout. | `SPEC` | Deadline misses stall the ceremony, while a conditional or variable release path can reveal duress. | Explicit |

### I. Network, metadata, and privacy

| ID | Assumption | Sources | If False | Status |
| --- | --- | --- | --- | --- |
| AR-64 | Tor provides enough reachability and anti-correlation benefit for peer, WT, and SAR communication. | `R`, `SM`, `SU` | Operators can be deanonymized or the protocol can be DoS'd more easily than assumed. | Explicit |
| AR-65 | Signed onion addresses are sufficient identity binding for peer communications. | `SU`, `SP`, `SM` | Attackers can redirect peer traffic despite signed address exchange. | Implicit |
| AR-66 | Metadata leakage to WT, SAR, Phone infrastructure, RPC providers, and network observers is acceptable for the target threat model. | `SPEC`, `SM` | The system may reduce coercion timing risk while increasing targeting risk. | Explicit |
| AR-67 | Static and dynamic doxing data remain confidential until a real duress event, subject to the entropy and uniqueness of the user-chosen `doxing_password`. | `SPEC`, `ADR`, `SM` | Weak or reused passwords expose rescue data after ciphertext, identifier or stored upload-key disclosure. | Explicit |
| AR-69 | Direct Bitcoin node RPC access outside Tor does not create unacceptable integrity or targeting risk. | `R`, `SM`, `SU` | Node traffic becomes another deanonymization or chain-view manipulation vector. | Implicit |

### J. Recovery, uniqueness, and design evolution

| ID | Assumption | Sources | If False | Status |
| --- | --- | --- | --- | --- |
| AR-75 | ST boot, debug and peripheral access cannot replace trusted firmware without authorization. | `ST`, `HW-BB`, `HW-H7` | Attacker-selected firmware can falsify ST interactions or disclose secrets. | Hardware assumption |
| AR-76 | Build, update, deployment, and hardening controls prevent compromise of Iso, ST, Boomlet, WT, and SAR. | `SM`, `R`, `D` | Malicious updates, compromised build pipelines, or service flaws can bypass protocol guarantees. | Operational assumption |

### K. Cross-cutting protocol boundaries

| ID | Assumption | Sources | If False | Status |
| --- | --- | --- | --- | --- |
| AR-78 | Peer separation requires independent secrets and failure domains as well as distinct identities; shared mnemonics, provisioning, services or legal exposure can correlate failures. | `SPEC`, `SM` | One shared compromise can defeat checks that assume independent peers or services. | Operational assumption |
| AR-80 | Backup import, SAR acknowledgment release and signed PSBT export recover from interruption without losing accepted state, duplicating activation or clearing the only usable fragment. | `SPEC`, `SM` | A crash or lost response can strand setup, lose withdrawal progress, expose a duress distinction or create ambiguous authority. | Partly resolved in 40e09c3b4c9162d6d9870092e49f3d6241e82e48 for SAR activation and replay; backup import and signed PSBT recovery remain open (DG-39). |

## Specified protocol controls

| ID | Control | Sources | If violated | Resolution |
| --- | --- | --- | --- | --- |
| AR-27 | Setup backup contains only long-lived setup authority and replay state; neither Boomlet nor Boomletwo creates or imports a `mystery` during setup. | `SPEC`, `ADR` | Backup can clone or disclose a future timing threshold. | Resolved in f74a8f0ce2563d0f2b68c50ff876dfa14f62f398 by the per-withdrawal mystery lifecycle and ADR 0006. |
| AR-70 | Setup-instance uniqueness is provided by `peer_setup_nonce`, canonical `setup_instance_id`, ST recomputation and review, strict record ordering, and chained setup checkpoints. | `SPEC`, `ADR` | Setup attempts can be confused or replayed across instances. | Resolved in 96734855a35db6af7c6d8be06d72d452433bbc6b by setup-instance binding and ADR 0001. |
| AR-79 | ST-approved rotation replaces an observed consent set, completing locally without backup or synchronizing any existing Boomletwo. | `SPEC`, `SM` | An observed safe response remains reusable for the lifetime of the setup. | Resolved in bf74a71241c3c3e5f39ec5cdd5108f031ebc4639 (DG-38); extended to operation without backup in f47c26a5f477cfd00e186b49bedc6dd5df9fe335. |
| AR-81 | SAR preserves authenticated upload history with defined identity, sequencing, conflict and retry rules. | `SPEC`, `SM` | An old or conflicting upload can displace an accepted rescue record. | Resolved in bf74a71241c3c3e5f39ec5cdd5108f031ebc4639 by authenticated append and conflict rejection (DG-40; SPEC Section 13.1). |

<a id="formal-analysis-obligations"></a>
<a id="retired-review-entries"></a>
## Retired entries

Identifiers are preserved without reuse. Review checks are in the
[verification guidance](audit_mappings.md#protocol-verification-guidance).
Original descriptions, sources and statuses below are retained for historical
reference; retirement reasons identify their current treatment.

### Retired assumptions

| ID | Original assumption | Original sources | Original consequence if false | Original status | Retirement reason |
| --- | --- | --- | --- | --- | --- |
| AR-05 | Ancillary procedures, implementation choices, and operator discipline are sufficient for properties that the protocol does not enforce. | `D`, `SM` | Real-world safety and integrity may depend on controls that are absent or ineffective. | Open | A blanket claim cannot establish the sufficiency of unspecified controls. Concrete protocol gaps and operating dependencies are recorded separately in Appendix D. |
| AR-68 | The data classification covers the major secrets, safety data, identifiers, transcripts, service records, logs, and timing metadata exposed by the current protocol. | `SPEC`, `SM` | Passive privacy exposure is larger than the risk analysis assumes. | Open | Asset and exposure coverage is a review task for the architecture's data classification and privacy analysis, supported by FM-03 and FM-08. |
| AR-71 | Ancillary procedures can be deferred: WT switch, Boomletwo activation, Phone change, Niso change, ST change, response to setup-bound SAR unavailability, timeout handling, blame handling. | `D`, `SM` | Realistic operations break long before the core withdrawal logic is reached. | Open | SPEC defines Phone replacement and outage stalls. Unspecified recovery procedures receive no security credit; their concrete gaps remain in DG-04, DG-10, DG-12 and DG-24. |
| AR-72 | The protocol has no unmodeled interaction that creates a failure beyond the listed gaps. | `R`, `D`, `SM` | Emergent behavior can invalidate safety or liveness arguments based on individual transitions. | Explicit | Completeness is an objective of threat analysis and verification. Identified interaction failures retain their own threat and gap entries. |
| AR-73 | Timing and liveness claims require simulation, formal analysis, conformance tests, and implementation evidence. | `SPEC`, `SM` | The timing and liveness claims may not be operationally defensible. | Open | Validation remains work in the protocol verification guidance. Missing evidence alone does not leave a specified protocol control unresolved. |
| AR-74 | Open parameter choices for `mystery`, intervals, and tolerances can be resolved without changing the core security properties. | `SM`, `D`, `FD`, `BD` | The promised deterrence/liveness balance may depend on values that do not actually exist. | Open | This presumes a successful future parameter choice. Joint parameter constraints and feasibility remain open in DG-02 and DG-25 and require timing and liveness verification. |

### Retired design gaps

| ID | Original issue | Original security concern | Original priority and status | Derived from | Retirement reason |
| --- | --- | --- | --- | --- | --- |
| DG-19 | The protocol lacks formal or simulation-based validation of timing behavior | No formal model or simulation evidence establishes timing and liveness under adversarial delay, reorgs, crashes, or equivocation. | High; Open | `AR-72`, `AR-73`, `AR-74` | Formal analysis and simulation are verification tasks, retained in the audit roadmap and protocol verification guidance. Concrete timing and liveness gaps remain in DG-02, DG-24 and DG-25. |
| DG-20 | Security-critical properties depend on future ancillaries or operator discipline | The design explains the cooperative path, but many real-world safety properties live outside the protocol core. | High; Open | `AR-05`, `AR-71`, `AR-72` | The blanket entry combines missing protocol procedures with external operating duties. Concrete recovery gaps remain in DG-04, DG-10, DG-12, DG-24 and DG-30; operating duties remain explicit assumptions. |

## Design-gap provenance

Canonical gap descriptions, priorities, statuses and resolution evidence:
[README.md Appendix D](README.md#appendix-d-detailed-design-gaps). Each active or resolved `DG-*` entry maps
to its source assumptions and the [verification guidance](audit_mappings.md#protocol-verification-guidance);
direct threat-catalog mappings appear in the same row.

| Gap ID | Derived from |
| --- | --- |
| DG-01 | `AR-10`, `AR-11`, `AR-14`, `AR-57`, `AR-69` |
| DG-02 | `AR-07`, `AR-09`, `AR-13`, `AR-14` |
| DG-03 | `AR-09`, `AR-13`, `AR-29`, `AR-52`, `AR-53`, `AR-54` |
| DG-04 | `AR-27`, `AR-28`, `AR-29`, `AR-54`, `AR-80`, `FM-13`, `FM-31` |
| DG-05 | `AR-18`, `AR-70` |
| DG-06 | `AR-15`, `AR-16`, `AR-17`, `AR-20`, `AR-21` |
| DG-07 | `AR-22`, `AR-23`, `AR-24`, `AR-25`, `AR-26`, `AR-30` |
| DG-08 | `AR-31`, `AR-32`, `AR-33`, `AR-34`, `AR-35`, `AR-38`, `AR-39`, `AR-75` |
| DG-09 | `AR-38`, `AR-39`, `AR-75` |
| DG-10 | `AR-37`, `AR-40`, `AR-41`, `AR-42`, `AR-43`, `AR-46`, `AR-47` |
| DG-11 | `AR-44`, `AR-45`, `AR-45A`, `AR-59`, `AR-60`, `AR-61`, `AR-67`, `AR-81`, `FM-32` |
| DG-12 | `AR-55`, `AR-56`, `AR-57`, `AR-58`, `AR-62` |
| DG-13 | `AR-59`, `AR-60`, `AR-61`, `AR-62`, `AR-63`, `FM-26` |
| DG-14 | `AR-48`, `AR-49`, `AR-50`, `AR-51`, `AR-52`, `AR-53`, `AR-54` |
| DG-15 | `AR-14`, `AR-35`, `AR-36`, `AR-51` |
| DG-16 | `AR-45A`, `AR-66`, `AR-67`, `AR-69` |
| DG-17 | `AR-18`, `AR-19`, `AR-20`, `AR-21`, `AR-60` |
| DG-18 | `AR-64`, `AR-65`, `AR-66`, `AR-69` |
| DG-21 | `AR-18`, `AR-19`, `AR-70`, `FM-14` |
| DG-22 | `AR-12`, `AR-16`, `AR-19`, `AR-50`, `FM-15` |
| DG-23 | `AR-18`, `AR-20`, `AR-60`, `FM-16`, `FM-23` |
| DG-24 | `AR-14`, `AR-55`, `AR-58`, `AR-60`, `FM-17` |
| DG-25 | `AR-14`, `AR-18`, `AR-57`, `FM-18` |
| DG-26 | `AR-59`, `AR-60`, `AR-62`, `FM-19` |
| DG-28 | `AR-09`, `AR-13`, `FM-21` |
| DG-29 | `AR-18`, `AR-20`, `FM-22`, `FM-23` |
| DG-30 | `AR-25`, `AR-80`, `FM-24`, `FM-31` |
| DG-31 | `AR-18`, `AR-31`, `AR-33`, `FM-25` |
| DG-32 | `AR-20`, `AR-55`, `AR-60`, `AR-66`, `AR-77`, `FM-06`, `FM-26` |
| DG-33 | `AR-17`, `AR-23`, `FM-28` |
| DG-34 | `AR-76` |
| DG-35 | `AR-23`, `AR-25`, `AR-27`, `FM-05` |
| DG-36 | `AR-40`, `AR-41`, `AR-76`, `FM-13` |
| DG-37 | `AR-48`, `AR-64`, `AR-66`, `AR-76`, `AR-78`, `FM-29`, `T-COMP-01`, `T-COMP-02` |
| DG-38 | `AR-31`, `AR-33`, `AR-35`, `AR-36`, `AR-79`, `FM-07`, `FM-30`, `T-HUM-06` |
| DG-39 | `AR-25`, `AR-28`, `AR-60`, `AR-80`, `FM-24`, `FM-31`, `T-BACK-02`, `T-DURESS-05`, `T-PROTO-06` |
| DG-40 | `AR-44`, `AR-45`, `AR-59`, `AR-61`, `AR-81`, `FM-32`, `T-DATA-02` |
| DG-41 | `AR-25`, `AR-60`, `FM-16`, `T-EOP-01` |


## Residual risk

Security depends on trusted hardware, peer cooperation, service availability
and timely rollover. Rules for resuming after a chain reorg, choosing timing
parameters, activating Boomletwo, retrying an interrupted backup import and
recovering a signed PSBT fragment lost after Boomlet cleanup remain unspecified.


## Status and source keys

- `Explicit`: required directly by the current specification or named design.
- `Implicit`: required for the security claim but not enforced as a protocol rule.
- `Open`: a protocol rule or policy remains unspecified.
- `Resolved`: closed by the cited specification, ADR, and evidence commit.
- `Hardware assumption`: required device capabilities and isolation.
- `Operational assumption`: required behavior of users, operators or external services.

Source keys: `SPEC` = `spec/SPEC.md`; `R` = repository `README.md`;
`D` = `DESIGN.md`; `SM` = `security_models/README.md`; `FD` =
`security_models/forced_determinism.md`; `ADR` = `adr/*.md`; `SU` =
`diagrams/setup/README.md`; `WD` = `diagrams/withdrawal/README.md`;
`ST` = `operational notes/secure_terminal/README.md`; `SP` =
`diagrams/setup/setup_diagram_without_states.puml`; `WI` =
`diagrams/withdrawal/initiator_withdrawal_diagram_without_states.puml`;
`WN` = `diagrams/withdrawal/non_initiator_withdrawal_diagram_without_states.puml`;
`BD` = `diagrams/withdrawal/block_constraints.{svg,pdf}`;
`HW-BB` and `HW-H7` are the referenced Secure Terminal prototype
datasheets.
