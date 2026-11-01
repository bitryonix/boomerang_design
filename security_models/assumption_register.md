# Boomerang threat-model assumptions and design gaps

> **Last change — 2026-07-14:** synced with the latest SPEC.md .

Boomerang's security claims depend on explicit, implicit, operational, and
prototype-level assumptions. `SPEC.md` is authoritative for protocol behavior.

## Security posture

The protocol does not enforce Boomlet behavior, ST integrity, peer cooperation,
WT and SAR availability, or timely rollover. Setup and withdrawal IDs, chained
checkpoints, directional channel contexts, approval-set attestations, and exact
SAR acknowledgements bind messages to the intended setup and ceremony.

Freshness checks bind accepted messages to the active ceremony, sequence, and
chain view, but reorg recovery and evidence retention remain open. There is
also a mismatch between the appliance-like trust placed in ST and the
prototype hardware, which exposes boot, debug, and general-purpose interfaces.
Forced determinism, Boomletwo activation, operating procedures,
jurisdiction-dependent rescue, and parameter selection remain open.

## Assumption register

### A. Baseline security scope and trust boundaries

| ID | Assumption | Sources | If False | Status |
| --- | --- | --- | --- | --- |
| AR-03 | Security claims apply to the exactly five-peer, 5-of-5 Boomerang profile. | `SPEC`, `R`, `D`, `SM` | Claims about one honest peer preserving Boomerang guarantees would not generalize. | Explicit |
| AR-04 | Implementations preserve the protocol's trust boundaries and message semantics. | `R`, `D` | An implementation may satisfy a different security model while claiming protocol conformance. | Implicit |
| AR-05 | Ancillary procedures, implementation choices, and operator discipline are sufficient for properties that the protocol does not enforce. | `D`, `SM` | Real-world safety and integrity may depend on controls that are absent or ineffective. | Open |

### B. Bitcoin, descriptor, and timing model

| ID | Assumption | Sources | If False | Status |
| --- | --- | --- | --- | --- |
| AR-06 | Bitcoin consensus, timelocks, and Taproot spending conditions behave as modeled by the descriptor design. | `R`, `D`, `SM` | Both non-deterministic and fallback security claims may be wrong. | Explicit |
| AR-07 | `MilestoneBlocks` is strictly increasing and operationally sensible, not merely well-formed. | `SPEC`, `SM` | Users can enter fallback too early, or funds can become awkwardly or prematurely spendable. | Explicit structure; open policy |
| AR-08 | Withdrawal starts only after `milestone_block_0`, and every component evaluates that gate against an acceptable chain view. | `SPEC` | Protocol state can diverge or enter unintended pre-milestone behavior. | Explicit |
| AR-09 | Users will roll funds into a new Boomerang setup before deterministic fallback makes coercion timing predictable. | `FD`, `R`, `D` | Boomerang degrades into a known-timeline withdrawal system. | Explicit |
| AR-10 | Stalling on local or WT height decrease and material RPC/WT disagreement is sufficient until a reorg-recovery policy exists. | `SPEC`, `SM` | Freshness checks, milestone gating, and counter progression can be evaluated on stale or reversible chain state, or remain stalled indefinitely. | Open |
| AR-11 | Niso and WT obtain accurate enough block height and chain context from their configured Bitcoin node(s). | `R`, `SU`, `WD`, `SM` | A bad node can skew timing, approvals, and liveness decisions. | Implicit |
| AR-12 | Fee estimation and PSBT hydration remain viable even after a possibly long withdrawal ceremony. | `WD`, `D` | A valid intent may become economically or practically unsignable by the time signing starts. | Implicit |
| AR-13 | The ping-pong "digging game" usually terminates before deterministic fallback if users start in time. | `R`, `D`, `FD`, `BD` | The main coercion-resistance promise collapses into forced determinism. | Open |
| AR-14 | `FRESHNESS_TOLERANCES`, ping/pong spacing, height-jump limits, and service timeouts can cover ordinary delay without admitting unsafe stale progress. | `SPEC`, `SM` | Honest executions fail freshness checks, or malicious delay becomes too easy. | Open |

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
| AR-23 | Boomlet's active-withdrawal `counter`, `mystery`, reach state, and sequence state cannot be read, rolled back, cloned, or externally accelerated. | `SPEC`, `SM`, `FD` | Current-ceremony timing becomes knowable or controllable by an attacker. | Explicit boundary |
| AR-24 | Boomlet faithfully enforces one-way state such as "reached mystery" and does not regress after a flag is set. | `WD`, `WI`, `WN` | Peers can be desynchronized or coerced into incorrect liveness behavior. | Implicit |
| AR-25 | Boomlet preserves one active withdrawal's state through stalls and retries, never regenerates its threshold inside that ceremony, and clears volatile state on export, explicit abort, or unrecoverable failure. | `SPEC`, `ADR` | The ceremony can fail midstream, reuse a disclosed threshold, or leak state into a later withdrawal. | Explicit |
| AR-26 | Java Card-class hardware can perform the required cryptography and state updates within practical time and endurance limits. | `D`, `FD` | The protocol becomes too slow, too fragile, or impossible to execute as designed. | Open |
| AR-27 | Setup backup contains only long-lived setup authority and replay state; neither Boomlet nor Boomletwo creates or imports a `mystery` during setup. | `SPEC`, `ADR` | Backup can clone or disclose a future timing threshold. | Resolved |
| AR-28 | Only one of Boomlet and Boomletwo will ever be active, although activation/deactivation is unspecified. | `R`, `SM`, `D` | Duplicate active devices or confused ownership can undermine both safety and liveness. | Open |
| AR-29 | Losing both Boomlet and Boomletwo is rare enough that fallback-only recovery remains acceptable. | `FD`, `D` | Funds protection depends too heavily on rare but catastrophic device loss patterns. | Implicit |
| AR-30 | Secure-element supply chain, lifecycle, and side-channel properties are acceptable for a Bitcoin custody system. | `R`, `D`, `SM` | A hidden hardware weakness can destroy the central Boomerang guarantee. | Explicit |

### E. Secure Terminal and human interface

| ID | Assumption | Sources | If False | Status |
| --- | --- | --- | --- | --- |
| AR-31 | ST is the trusted UI for verification and duress input; Niso is not. | `R`, `ST`, `SM` | Duress signaling and user verification reduce to whatever the online host shows. | Explicit |
| AR-32 | ST securely generates, stores, and uses its long-term keys. | `ST`, `SP` | An attacker can read or alter the ST-Boomlet channel. | Explicit |
| AR-33 | ST displays exact Boomlet-authored content and its joystick records exact user intent. | `ST`, `WI`, `WN` | Users can be tricked into false safe or false duress outcomes. | Explicit |
| AR-34 | ST remains air-gapped except for the intended QR exchange path. | `R`, `ST` | Malware-capable connectivity can directly exfiltrate consent-set or duress data. | Explicit |
| AR-35 | An attacker cannot practically observe both the ST display and the user's selection behavior well enough to learn the consent set under coercion. | `ST` | Plausible deniability degrades, and attackers can demand the true safe response. | Explicit |
| AR-36 | Users can memorize the consent set and reliably reproduce it during legitimate safe use, including under stress and fatigue. | `SPEC` | Valid but mistaken selections trigger false duress and unnecessary rescue escalation. | Implicit |
| AR-37 | One ST per peer is enough for ongoing operation, although ST replacement remains ancillary. | `D` | ST loss or compromise becomes an operational dead end. | Open |
| AR-38 | A DIY Portenta-based ST can be made tamper-evident and tamper-resistant enough to satisfy the ST trust boundary. | `ST`, `HW-BB`, `HW-H7` | The assumed trusted UI is only a prototype dev kit with insufficient hardening. | Prototype tension |
| AR-39 | Prototype attack surface exposed by bootloader, USB, JTAG, Ethernet, SD, camera, and other breakout/debug features can be controlled by build and ops choices. | `HW-BB`, `HW-H7`, `ST` | The proposed ST platform may remain much easier to tamper with than the security model assumes. | Prototype tension |

### F. Iso, Niso, and phone environments

| ID | Assumption | Sources | If False | Status |
| --- | --- | --- | --- | --- |
| AR-40 | Iso remains trusted and isolated throughout installation, setup, Boomletwo backup, and final signing; compromise or substitution during setup is excluded. | `SPEC`, `SM` | An attacker can install a chosen `normal_pubkey`, `doxing_key`, or SAR, steal mnemonic material, or authorize a malicious backup target. | Explicit |
| AR-41 | User entropy, BIP39/BIP32 derivation, mnemonic handling, and passphrase handling on Iso are correct and secret. | `SPEC` | Normal-key security collapses independently of Boomlet. | Implicit |
| AR-42 | Niso may be malicious without authorizing a spend because Boomlet and ST verify critical state; Niso can still censor, misroute, corrupt presentation, or supply unsafe chain data. | `SPEC`, `SM` | The online host can gain more authorization power than the protocol assigns it. | Explicit boundary |
| AR-43 | Niso's RPC endpoint is honest, reachable, and configured correctly. | `R`, `SU`, `WD`, `SM` | Block height, satisfiability, and freshness decisions can be manipulated. | Implicit |
| AR-44 | Phone OS/app security is good enough for dynamic doxing collection and SAR registration. | `R`, `D`, `SU` | Rescue data can be stale, forged, leaked, or suppressed. | Explicit |
| AR-45 | Phone-to-SAR registration, payment, and synchronization cannot be silently redirected or spoofed in a way the user will miss. | `SPEC` | The user can think they are covered while SAR never received valid rescue data. | Implicit |
| AR-45A | `doxing_password` is user-chosen; Boomlet receives `doxing_key` and does not enforce password policy. | `SPEC`, `ADR`, `SM` | Setup or backup may depend on a second mnemonic-length secret, or components may diverge on password-policy enforcement. | Explicit |
| AR-46 | Users can safely shuttle Boomlet between Iso and Niso without plugging the wrong device into the wrong host or mixing peer hardware. | `SU`, `WD`, `D` | Cross-device confusion can undermine both safety and liveness. | Implicit |
| AR-47 | Ancillary procedures can replace a Phone, Niso, or ST without breaking protocol state or security. | `D` | Device loss or replacement can make the system unsafe or unusable. | Open |

### G. Peers, users, and human operations

| ID | Assumption | Sources | If False | Status |
| --- | --- | --- | --- | --- |
| AR-48 | At least one honest peer exists in the N-of-N Boomerang regime. | `R`, `SM`, `D` | The "one honest peer preserves the promise" argument disappears. | Explicit |
| AR-49 | Peers honestly exchange and verify peer IDs and Tor addresses out of band. | `R`, `SU`, `SP`, `SM` | Impersonation or routing attacks can be injected before Tor is even used. | Explicit |
| AR-50 | Users and peers meaningfully verify ordered peer setup records, WT order, milestone blocks, `setup_instance_id`, transaction identifiers, and peer data when prompted. | `SPEC`, `SU`, `WD`, `SP`, `WI`, `WN` | Human checks become ceremonial only, leaving host-mediated substitution attacks alive. | Implicit |
| AR-51 | Peers remain available across randomized delays, recurring duress checks, and long withdrawal timelines. | `D`, `FD`, `BD` | Honest liveness fails under normal operational conditions. | Implicit |
| AR-52 | Non-cooperation is rare enough that N-of-N remains acceptable despite known liveness cost. | `R`, `FD`, `D` | The system regularly falls back to deterministic recovery or deadlock. | Explicit |
| AR-53 | Peers will not exploit deterministic fallback once they can predict or wait out Boomerang timing. | `FD`, `D` | The normal regime becomes an insider bypass path rather than only a liveness valve. | Implicit |
| AR-54 | Users will maintain safe backups, heed rollover timing, and follow operational guidance consistently. | `R`, `FD`, `D` | Security rests on a discipline level that the design itself does not enforce. | Explicit |

### H. WT and SAR service assumptions

| ID | Assumption | Sources | If False | Status |
| --- | --- | --- | --- | --- |
| AR-55 | The one active WT stays available and responsive for the entire setup or withdrawal ceremony. | `SPEC`, `SM` | Coordination and counter progression stop even when peers are honest. | Explicit |
| AR-56 | WT signs only correct protocol statements and forwards messages without selective censorship or bias. | `WD`, `WI`, `WN`, `SM` | A non-custodial service can force desynchronization or liveness failure. | Implicit |
| AR-57 | WT's block-height view is trustworthy enough to act as the protocol heartbeat. | `R`, `D`, `SM`, `WD` | Freshness windows and progress rules rest on an attacker-controlled clock. | Explicit |
| AR-58 | Ancillary procedures can provide WT redundancy, switching, and recovery without changing protocol state or trust assumptions. | `SM`, `D` | A single WT remains a critical service dependency. | Open |
| AR-59 | SAR securely stores encrypted doxing data, identifiers, and related metadata over long periods, assuming offline cracking resistance is bounded by the user-chosen `doxing_password`. | `R`, `SM`, `SU`, `ADR` | Off-chain safety data become a privacy or rescue failure point; weak passwords make leaked ciphertext more exposed to offline guessing. | Implicit |
| AR-60 | Each setup-bound SAR classifies valid placeholders, commits the same-shaped durable record by the fixed deadline, handles repeats idempotently, and signs the exact encrypted placeholder. | `SPEC`, `SM` | Positive duress may be missed, observable processing may reveal it, or Boomlet may continue under false delivery assumptions. | Explicit |
| AR-61 | Effective, lawful, correctly directed, and non-escalatory SAR action is required for physical intervention but is not guaranteed by protocol acknowledgement. | `R`, `D`, `SM` | Authenticated signaling can succeed while rescue fails, harms the wrong person, or escalates violence. | Explicit |
| AR-62 | Reputation and SAR operator selection are an acceptable control for WT and SAR social trust. | `D`, `SM` | Operational trust remains an assumption rather than a designed control. | Explicit |
| AR-63 | SAR will not become a later attacker after identity is revealed during a rescue event. | `D`, `SM` | Duress rescue itself may create a future targeting or coercion channel. | Implicit |
| AR-77 | SAR has a reliable monotonic clock, a fixed positive `sar_placeholder_ack_delay` longer than bounded worst-case pre-acknowledgement work, and a matching WT timeout. | `SPEC` | Deadline misses stall the ceremony, while a conditional or variable release path can reveal duress. | Open deployment assumption |

### I. Network, metadata, and privacy

| ID | Assumption | Sources | If False | Status |
| --- | --- | --- | --- | --- |
| AR-64 | Tor provides enough reachability and anti-correlation benefit for peer, WT, and SAR communication. | `R`, `SM`, `SU` | Operators can be deanonymized or the protocol can be DoS'd more easily than assumed. | Explicit |
| AR-65 | Signed onion addresses are sufficient identity binding for peer communications. | `SU`, `SP`, `SM` | Attackers can redirect peer traffic despite signed address exchange. | Implicit |
| AR-66 | Metadata leakage to WT, SAR, Phone infrastructure, RPC providers, and network observers is acceptable for the target threat model. | `SPEC`, `SM` | The system may reduce coercion timing risk while increasing targeting risk. | Explicit |
| AR-67 | Static and dynamic doxing data remain confidential until a real duress event, subject to the entropy and uniqueness of the user-chosen `doxing_password`. | `SPEC`, `ADR`, `SM` | The rescue channel becomes a high-value doxing database; weak or reused passwords reduce confidentiality after ciphertext exposure. | Explicit |
| AR-68 | The data classification covers the major secrets, safety data, identifiers, transcripts, service records, logs, and timing metadata exposed by the current protocol. | `SPEC`, `SM` | Passive privacy exposure is larger than the risk analysis assumes. | Open |
| AR-69 | Direct Bitcoin node RPC access outside Tor does not create unacceptable integrity or targeting risk. | `R`, `SM`, `SU` | Node traffic becomes another deanonymization or chain-view manipulation vector. | Implicit |

### J. Recovery, uniqueness, and design evolution

| ID | Assumption | Sources | If False | Status |
| --- | --- | --- | --- | --- |
| AR-70 | Setup-instance uniqueness is provided by `peer_setup_nonce`, canonical `setup_instance_id`, ST recomputation and review, strict record ordering, and chained setup checkpoints. | `SPEC`, `ADR` | No setup-identity protocol gap remains. | Resolved |
| AR-71 | Ancillary procedures can be deferred: WT switch, Boomletwo activation, Phone change, Niso change, ST change, response to setup-bound SAR unavailability, timeout handling, blame handling. | `D`, `SM` | Realistic operations break long before the core withdrawal logic is reached. | Open |
| AR-72 | The protocol has no unmodeled interaction that creates a failure beyond the listed gaps. | `R`, `D`, `SM` | Emergent behavior can invalidate safety or liveness arguments based on individual transitions. | Explicit |
| AR-73 | Timing and liveness claims require simulation, formal analysis, conformance tests, and implementation evidence. | `SPEC`, `SM` | The timing and liveness claims may not be operationally defensible. | Open |
| AR-74 | Open parameter choices for `mystery`, intervals, and tolerances can be resolved without changing the core security properties. | `SM`, `D`, `FD`, `BD` | The promised deterrence/liveness balance may depend on values that do not actually exist. | Open |
| AR-75 | Exposed prototyping and debug features do not invalidate the security claims made for the trusted UI. | `ST`, `HW-BB`, `HW-H7` | The trusted UI may rest on a platform that is materially weaker than assumed. | Prototype tension |
| AR-76 | Build, update, deployment, and hardening controls prevent compromise of Iso, Niso, ST, Boomlet, WT, and SAR. | `SM`, `R`, `D` | Malicious updates, compromised build pipelines, or service flaws can bypass protocol guarantees. | Open |

### K. Cross-cutting protocol boundaries

| ID | Assumption | Sources | If False | Status |
| --- | --- | --- | --- | --- |
| AR-78 | Five peer identities and N-of-N authorization provide useful separation only if devices, firmware, provisioning, RPC sources, services, payment trails, and legal exposure do not fail through one common cause or cooperating coalition. | `SPEC`, `SM` | One shared compromise can defeat checks that assume independent peers or services. | Open |
| AR-79 | A coercer cannot record a legitimate safe ST interaction and reuse the learned consent response for the lifetime of the setup; no rotation procedure exists after suspected observation. | `SPEC`, `SM` | The attacker can force future checks to evaluate as safe. | Open |
| AR-80 | Backup import, SAR acknowledgement release, and signed-fragment export recover from every interruption without duplicating authority, losing resumable state, releasing an inconsistent acknowledgement, or clearing the only usable fragment. | `SPEC`, `SM` | A crash or lost response can strand setup, lose withdrawal progress, expose a duress distinction, or create ambiguous authority. | Open |
| AR-81 | SAR can determine which authenticated dynamic rescue update is current and reject stale, replayed, future-dated, or conflicting data despite the absence of canonical ordering and expiry rules. | `SPEC`, `SM` | Rescue may use an old or attacker-selected location while all envelope checks pass. | Open |

## Formal-analysis obligations

No executable formal model or model-checker configuration is present in the
repository. The `FM-*` identifiers name proof obligations and abstraction
limits; they are not evidence that a property has been checked.

| ID | Required property or abstraction | Boundary |
| --- | --- | --- |
| FM-01 | Stable principal identity binding | Actions attributed to a peer, WT, or SAR must authenticate as that principal; N-of-N and one-honest-peer arguments fail under impersonation. |
| FM-02 | Unforgeable approvals, commitments, service statements, and signatures | A proof may treat authenticated protocol events as principal-authored only if the concrete signature and envelope checks are preserved. |
| FM-03 | Confidential protected payloads until protocol-authorized disclosure | Placeholder plaintext and rescue data may be idealized as opaque, but logs, traffic metadata, password guessing, and endpoint compromise remain outside that abstraction. |
| FM-04 | Honest, attributable freshness evidence | Abstract block heights must preserve local Niso view, WT view, acceptance time, decreases, disagreement, and `CHAIN_VIEW_UNSAFE`; reorg recovery cannot be assumed. |
| FM-05 | Fresh per-withdrawal mystery state | Each Boomlet creates `mystery` once on entry to `DIGGING`, keeps it fixed through retries, and erases it with active state. WT can learn the reached transition for the current ceremony; no claim should hide that disclosure. |
| FM-06 | Public-observation equivalence for safe and duress executions | The observation must include message shape, size class, routing, fixed release deadline, retry schedule, failure class, logs, metrics, queue behavior, and durable-write behavior. |
| FM-07 | Faithful capture of user approval and duress intent | ST, Boomlet, and the physical interaction are trusted inputs unless compromised-device and observed-user behavior are modeled explicitly. |
| FM-08 | Realistic WT and SAR knowledge | Service state must include registration, payment, routing, timing, replay tuples, logs, and operational side channels; ciphertext opacity alone is insufficient. |
| FM-09 | Replay resistance under one active withdrawal | Scope IDs, nonces, IVs, sequences, state transitions, and replay memory must be modeled. Concurrent withdrawals are outside the current SPEC profile and must not be inferred safe. |
| FM-10 | Truthful fallback availability | Fallback must depend only on Taproot milestones and valid normal-key authorization. Boomlet loss, backup activation, and operator fallback procedures require explicit environment transitions. |
| FM-11 | Byzantine transport behavior | Loss, duplication, censorship, reordering, equivocation, and attacker-selected delivery must be modeled explicitly before deriving hostile-network guarantees from protocol state transitions. |
| FM-13 | Correct setup-to-withdrawal handoff | Withdrawal reasoning starts with valid pairing, consent enrollment, setup records, checkpoints, keys, descriptor, SAR binding, and trusted Iso installation. |
| FM-14 | Two-stage withdrawal identity binding | `withdrawal_id` binds approval fan-out; `approved_withdrawal_id` binds the unanimous set and every later commitment, placeholder, ping, pong, signing, export, and replay scope. |
| FM-15 | Transaction-intent continuity | The model must preserve independent `tx_id` approval, canonical approval-set checks, allowed hydration changes, `SIGHASH_DEFAULT`, descriptor checks, signing-package verification, and final broadcast equality. |
| FM-16 | Exact placeholder acknowledgement | Progress may depend only on a SAR signature over the byte-for-byte encrypted placeholder sent for the active `approved_withdrawal_id`. |
| FM-17 | Explicit liveness fairness | Safety does not imply eventual approval, acknowledgement, universal reach, signing, relay, or broadcast. Any liveness result must name peer, WT, SAR, network, and scheduler fairness. |
| FM-18 | Preserved freshness provenance | Later transitions must remain grounded in the signer, sequence, ceremony ID, accepted chain view, and state that made the earlier evidence fresh. |
| FM-19 | Exactly one setup-bound SAR per peer | Each setup fixes one SAR identity per peer. Selection, replacement, or identity failover within that setup is prohibited. Changing SAR requires a fresh setup. |
| FM-20 | Full MuSig2 session safety | A transaction-level signing abstraction does not cover nonce generation, aggregate-key binding, public-nonce handling, partial signatures, erasure, or session-confusion failures required by BIP327. |
| FM-21 | Separate deterministic fallback ceremony | Boundary-level milestone availability does not establish the end-to-end safety, privacy, or liveness of a fallback spend. |
| FM-22 | Unique placeholder-instance identity | Fresh IV, Boomlet identity, and `approved_withdrawal_id` must distinguish every security-relevant instance. |
| FM-23 | Idempotent SAR replay handling | SAR must authenticate before tuple lookup, commit new duress activation before acknowledgement release, and give repeated valid tuples the same observable acknowledgement behavior. |
| FM-24 | Safe reset and interrupted export | One active withdrawal and reset-separated later withdrawals may be assumed only after accounting for interruption between signing, export, Boomlet cleanup, Niso relay, WT aggregation, and broadcast. |
| FM-25 | Literal ST/Boomlet transcript semantics | A proof of the nonce exchange must preserve transaction-review content, phase, outstanding nonce, five-column country mapping, index validation, and exact response equality; prompt rendering remains outside the SPEC. |
| FM-26 | End-to-end duress timing and storage equivalence | The fixed SAR deadline, identical bounded pre-acknowledgement sequence, same-shaped durable record, deadline-overrun failure, asynchronous rescue work, and attacker-visible telemetry must be represented. No delivery property may be promoted into a claim of effective or safe physical response. |
| FM-28 | Recurring-duress selection | The SPEC requires an unbiased draw for the configured interval but does not fix the domain, selection predicate, PRNG state lifetime, or cross-peer independence. A model must state any choice as an assumption. |
| FM-29 | Correlated compromise and coalitions | Principal identities must not be treated as independent failure domains. Shared trusted components and cooperating WT, RPC, SAR, Phone, peer, payment, or legal actors require explicit adversary transitions. |
| FM-30 | Observed consent interaction | A human model must allow screen and hand observation, attacker-controlled input, and reuse of a learned safe response. Privacy of the physical interaction cannot be assumed silently. |
| FM-31 | Crash-consistent handoffs | The state space must include interruption before and after backup import, `BackupDone`, SAR activation commit, acknowledgement release, signed-fragment export, receiver receipt, and local cleanup. |
| FM-32 | Dynamic rescue-data freshness | Authentication alone is insufficient. Any claim about current rescue data must model ordering, timestamp trust, expiry, rollback, duplicates, and conflicting updates. |

## Design-gap provenance

Canonical gap descriptions, priorities, statuses, and resolution evidence:
[README.md Appendix D](README.md#appendix-d-detailed-design-gaps). Each `DG-*` entry maps
to its source assumptions and formal boundaries; direct threat-catalog
mappings appear in the same row.

| Gap ID | Derived from |
| --- | --- |
| DG-01 | `AR-10`, `AR-11`, `AR-14`, `AR-57`, `AR-69` |
| DG-02 | `AR-07`, `AR-09`, `AR-13`, `AR-14`, `AR-74` |
| DG-03 | `AR-09`, `AR-13`, `AR-29`, `AR-52`, `AR-53`, `AR-54` |
| DG-04 | `AR-27`, `AR-28`, `AR-71`, `AR-80`, `FM-31` |
| DG-05 | `AR-18`, `AR-70` |
| DG-06 | `AR-15`, `AR-16`, `AR-17`, `AR-20`, `AR-21` |
| DG-07 | `AR-22`, `AR-23`, `AR-24`, `AR-25`, `AR-26`, `AR-30` |
| DG-08 | `AR-31`, `AR-32`, `AR-33`, `AR-34`, `AR-35`, `AR-38`, `AR-39`, `AR-75` |
| DG-09 | `AR-38`, `AR-39`, `AR-75` |
| DG-10 | `AR-37`, `AR-40`, `AR-41`, `AR-42`, `AR-43`, `AR-46`, `AR-47`, `AR-71` |
| DG-11 | `AR-44`, `AR-45`, `AR-45A`, `AR-59`, `AR-60`, `AR-61`, `AR-67`, `AR-81`, `FM-32` |
| DG-12 | `AR-55`, `AR-56`, `AR-57`, `AR-58`, `AR-62` |
| DG-13 | `AR-59`, `AR-60`, `AR-61`, `AR-62`, `AR-63`, `FM-26` |
| DG-14 | `AR-48`, `AR-49`, `AR-50`, `AR-51`, `AR-52`, `AR-53`, `AR-54`, `AR-71` |
| DG-15 | `AR-14`, `AR-35`, `AR-36`, `AR-51`, `AR-74` |
| DG-16 | `AR-45A`, `AR-66`, `AR-67`, `AR-68`, `AR-69` |
| DG-17 | `AR-18`, `AR-19`, `AR-20`, `AR-21`, `AR-60` |
| DG-18 | `AR-64`, `AR-65`, `AR-66`, `AR-69` |
| DG-19 | `AR-72`, `AR-73`, `AR-74` |
| DG-20 | `AR-05`, `AR-71`, `AR-72` |
| DG-21 | `AR-18`, `AR-19`, `AR-70`, `FM-14` |
| DG-22 | `AR-12`, `AR-16`, `AR-19`, `AR-50`, `FM-15` |
| DG-23 | `AR-18`, `AR-20`, `AR-60`, `FM-16`, `FM-23` |
| DG-24 | `AR-14`, `AR-55`, `AR-58`, `AR-60`, `AR-71`, `FM-17` |
| DG-25 | `AR-14`, `AR-18`, `AR-57`, `FM-18` |
| DG-26 | `AR-59`, `AR-60`, `AR-62`, `FM-19` |
| DG-28 | `AR-09`, `AR-13`, `AR-71`, `FM-21` |
| DG-29 | `AR-18`, `AR-20`, `FM-22`, `FM-23` |
| DG-30 | `AR-25`, `AR-71`, `AR-80`, `FM-24`, `FM-31` |
| DG-31 | `AR-18`, `AR-31`, `AR-33`, `FM-25` |
| DG-32 | `AR-20`, `AR-55`, `AR-60`, `AR-66`, `AR-77`, `FM-06`, `FM-26` |
| DG-33 | `AR-17`, `AR-23`, `AR-74`, `FM-28` |
| DG-34 | `AR-76` |
| DG-35 | `AR-23`, `AR-25`, `AR-27`, `FM-05` |
| DG-36 | `AR-40`, `AR-41`, `AR-76`, `FM-13` |
| DG-37 | `AR-48`, `AR-64`, `AR-66`, `AR-76`, `AR-78`, `FM-29`, `T-COMP-01`, `T-COMP-02` |
| DG-38 | `AR-31`, `AR-33`, `AR-35`, `AR-36`, `AR-79`, `FM-07`, `FM-30`, `T-HUM-06` |
| DG-39 | `AR-25`, `AR-28`, `AR-60`, `AR-71`, `AR-80`, `FM-24`, `FM-31`, `T-BACK-02`, `T-DURESS-05`, `T-PROTO-06` |
| DG-40 | `AR-44`, `AR-45`, `AR-59`, `AR-61`, `AR-81`, `FM-32`, `T-DATA-02` |

## Residual risk

On-chain authorization has the clearest rules. Trusted-hardware assurance,
long-running coordination, timing parameters, WT and SAR dependencies, and
operator-run lifecycle procedures still have high-priority gaps.


## Status and source keys

- `Explicit`: required directly by the current specification or named design.
- `Implicit`: required for the security claim but not enforced as a protocol rule.
- `Open`: unresolved, deferred, or dependent on deployment policy.
- `Resolved`: closed by the cited specification, ADR, and evidence commit.
- `Prototype tension`: stronger assurance is assumed than the named prototype establishes.

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
