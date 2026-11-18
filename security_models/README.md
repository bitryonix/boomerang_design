# Boomerang Threat Model

> **Last change — 2026-11-16:** aligned threat coverage with rescue-data history, consent rotation and optional backup; separated protocol gaps from external assumptions.

Related material: [architecture](architecture.md), [attack trees](attack_trees.md),
and [threat mappings](audit_mappings.md). Appendix D remains the canonical
record of protocol gaps, assumptions and resolutions.

## Scope and assumptions

### System summary

Boomerang is a Bitcoin cold-storage protocol for cases where physical coercion
is a credible threat. Each private threshold is sampled from a bounded range,
but the signing point is not known in advance. The ceremony itself can stall on
peer, WT, SAR, Tor, RPC, chain-view, or device failure. Duress signaling runs
through the ordinary withdrawal flow.

The profile in `spec/SPEC.md` has exactly five peers and a 5-of-5 Boomerang
branch. A ceremony uses one WT; each peer selects one SAR, has one active
Boomlet, and may keep one inactive Boomletwo backup. Backup choices may differ
between peers without changing thresholds or timelocks. A different peer count,
threshold, fallback tree, cryptographic profile, or checkpoint sequence is a
different protocol profile.

There are two spending paths. In the Boomerang path, spending starts after
`milestone_block_0` and only after every Boomlet reaches its private `mystery`
value through the freshness-checked ping/pong process. The fallback path uses
normal keys and a timelocked waterfall whose threshold eventually falls from
5-of-5 to 1-of-5.

Each on-chain `boom_pubkey_i` combines two keys with MuSig2: a mnemonic-backed
normal key used by Iso and a non-exportable share held by Boomlet. Compromising
the mnemonic is therefore not enough to sign through the Boomerang path.

ST presents the duress challenge and Boomlet evaluates the answer. Checks run
at commitment and at randomized points in the digging game. A safe placeholder
decrypts to `zero_bytes_32`; a duress placeholder decrypts to the setup-bound
`doxing_key_for_sar`. WT relays the placeholder to SAR, and SAR signs an
acknowledgement of the exact placeholder. SAR releases every valid safe or
duress acknowledgement at the same fixed deployment deadline after completing
the same bounded durable-write path. For duress, SAR stores an activation record
before acknowledging the exact placeholder. Timely, lawful, correctly directed,
effective and safe intervention remains an external dependency.

Setup state is tied together by `setup_instance_id`, signed `PeerSetupRecord`
values, and chained `setup_checkpoint` values. Withdrawal approval uses
`withdrawal_id`; everything after unanimous approval uses
`approved_withdrawal_id`, including commitments, placeholders, pings, pongs,
reached-ping collections, signing, export, and replay state. WT may not forward
commit placeholders to SAR until it has verified the non-initiators'
approval-set attestations.

### In-scope components

Peers are joint custodians of the bitcoin involved. Per peer or operator, with exactly five peers in the profile:

- **User:** the human operator.
- **Iso, offline:** trusted key-derivation, Boomlet installation, backup authorization and verification, and final-signing environment. Compromise or substitution during setup is outside the SPEC threat boundary.
- **Boomlet, secure element:** holds key share, enforces non-determinism, and runs duress logic.
- **Boomletwo, optional backup secure element:** authenticated target-bound import of long-lived Boomlet setup state. Setup and backup carry no `mystery`; a fresh threshold is generated only when an activated device enters `DIGGING`. Activation is unspecified.
- **ST, Secure Terminal:** air-gapped, tamper-evident UI for user verification and duress challenges.
- **Niso, online:** networked coordinator per peer; handles Tor communications, talks to Bitcoin node RPC, interfaces with WT, and mediates QR flows to ST.
- **Phone:** used for SAR registration and the encrypted dynamic doxing feed.

External dependencies and services:

- **Bitcoin network / miners:** confirmation, timelocks, reorgs, fee market
- **Bitcoin node RPC endpoint(s)** used by each peer’s Niso
- **Tor network**
- **WT (Watchtower):** one active coordination and height-reporting service per ceremony, plus the relay for approvals, placeholders, pings, signing fragments, and broadcast
- **SAR (Search & Rescue):** one selected SAR per peer; receives encrypted doxing data, detects duress, signs exact placeholder acknowledgements, and initiates physical response

### Out-of-scope

The protocol security argument does not protect:

- Compromise or substitution of Iso during setup
- SAR’s real-world response procedures
- Detailed organizational governance
- Manufacturing processes for Boomlet / ST hardware
- Wallet UX beyond what is specified by the protocol messages

### Primary assets

**Custody / value assets**
- **Bitcoin funds** locked by the Boomerang Taproot output
- **Transaction authorization correctness:** PSBT contents, destination, amounts, fees

**Key material & cryptographic assets**
- `mnemonic`, `passphrase`, and derived **normal private keys**
- Boomlet-held `boom_musig2_privkey_share`, non-exportable
- Boomlet identity keypair for signing and encryption
- ST identity keypair for the Boomlet↔ST secure channel
- Tor onion-service secret keys for peer Niso addresses
- WT and SAR long-term signing / encryption keys

**Duress & safety assets**
- `duress_consent_set`
- user-chosen `doxing_password`, derived `doxing_key`, and derived `doxing_data_identifier`
- **Static doxing data**, retained **dynamic upload history**, upload authentication key, and signed receipts

**Protocol state assets**
- Per-withdrawal, per-Boomlet `mystery`, `counter`, reached state, and `ping_seq_num`
- `boomerang_params` and `boomerang_descriptor`
- `setup_instance_id`, `setup_checkpoint`, `withdrawal_id`, and `approved_withdrawal_id`
- Approval collections, approval-set attestations, commits, pings, pongs, reached-ping collections, and signing-session state
- Consent counters, sticky duress flags, and retained rotation and reset records
- SAR placeholder replay tuples: `{approved_withdrawal_id, boomlet_identity_pubkey, duress_placeholder.iv}`
- Protocol transcripts
- Milestone schedule, encoded as `MilestoneBlocks`

### Security objectives

**Funds protection**
- Prevent unauthorized spending in both regimes.
- Prevent silent transaction tampering.
- Ensure recoverability in the normal regime under loss scenarios, while managing the associated risk.

**Coercion resistance**
- Under coercion, reduce attacker confidence in *time to cash-out*.
- Provide recurring, plausibly deniable duress signaling without observable protocol divergence.
- Deliver an authenticated duress signal before signing without claiming that
  external intervention will succeed or avoid harm.

**Privacy & safety**
- Minimize metadata that could enable targeting of key holders.
- Protect doxing data and duress signals from unauthorized disclosure.
- Resist correlation attacks that deanonymize peers, schedules, or duress events.

**Integrity & liveness**
- Prevent protocol-state desynchronization and replay.
- Ensure withdrawal completes when honest peers cooperate and dependencies are available.
- Ensure failures degrade safely, preferably fail-closed in the Boomerang regime.

### High-impact assumptions

The risk ratings assume:

- Boomlet's secure-element boundary holds;
- ST remains trustworthy;
- Iso remains trusted and isolated throughout installation, setup, backup, and
  final signing;
- the cryptography used by the message and signing flows holds;
- build, update, and deployment paths for Iso, ST, Boomlet, WT, and SAR
  are trustworthy enough;
- at least one peer is honest;
- Bitcoin timelocks behave as expected;
- Niso and WT obtain chain views trustworthy enough for milestone gates,
  freshness checks, and counter advancement;
- setup IDs, withdrawal IDs, checkpoints, freshness checks, and signed
  transcripts keep each setup and ceremony separate;
- `tx_id` continuity, approval-set attestations, PSBT checks, signing-package
  verification, and final transaction validation preserve the operator's
  intent;
- SAR uses a monotonic clock, an adequate fixed acknowledgement delay, the same
  durable-write path for safe and duress records, and an acknowledgement bound
  to the exact placeholder;
- effective and safe physical intervention remains an external dependency, not
  a protocol guarantee;
- peer, device, service, and infrastructure failures are independent enough
  that one common cause does not defeat all five peers and their checks;
- only one of Boomlet and Boomletwo is active for a peer identity;
- WT and SAR remain available; and
- peers have a secure out-of-band channel.

### Open protocol choices

Remaining protocol choices include:

- Exact values and selection policy for the non-determinism and duress-check parameters
- Joint SAR processing bounds, acknowledgement delay and WT timeouts
- Reorg-handling policy
- WT failover under the setup binding
- Canonical version type and value, backup replay encoding, and numeric size and nesting limits
- Boomletwo activation / recovery protocol

### Protocol controls and boundaries

Protocol controls depend on the stated trust and operating assumptions.

| Boundary | Why it matters |
| --- | --- |
| **Chain view and reorg policy** | The protocol detects local or WT height decreases and material RPC/WT disagreement and stalls with `CHAIN_VIEW_UNSAFE`; reorg recovery, chain-source trust, and divergent-node policy remain undefined. |
| **Setup / withdrawal binding** | Core binding is specified through `setup_instance_id`, chained setup checkpoints, `withdrawal_id`, and `approved_withdrawal_id`. |
| **Hydrated PSBT authorization contract** | Transaction authorization is specified through `tx_id` continuity, signing-package verification, descriptor and transaction checks, and final broadcast `tx_id` checks. |
| **Boomlet and Boomletwo activation semantics** | Target-bound backup export and import exist. Procedures for activating or revoking Boomletwo and preventing both devices from signing concurrently are undefined. No mystery exists at backup time. |
| **Duress-placeholder lifecycle** | Fresh envelopes, `approved_withdrawal_id` context, exact-envelope SAR signatures, replay tuples, a fixed release deadline, and the same pre-acknowledgement durable-write path bind the lifecycle. |
| **Operational transitions** | Phone replacement is specified; Niso and ST replacement, WT failover and resuming after prolonged setup-bound SAR unavailability are not. Users must connect devices to the correct hosts. |
| **Honest-path liveness / fairness policy** | Failure classes, retry restrictions, stall, and explicit abandonment are defined. Interoperable timeouts, blame, peer non-cooperation, WT failover, and response to prolonged SAR unavailability remain undefined. |
| **Parameter selection and timing policy** | Security depends on workable choices for milestones, mystery range, duress cadence, and freshness tolerances. |
| **Cryptographic profile and canonical transcript encoding** | The profile specifies AES-256-CBC with PKCS#7, AES-CMAC encrypt-then-MAC, SP 800-108 CMAC KDF context separation, directional channel schedules, canonical object bytes, schema IDs, and scope-specific contexts. Version encoding, backup replay encoding and numeric limits remain open. |
| **Trusted Iso setup boundary** | Iso chooses and installs `normal_pubkey`, `doxing_key`, and the selected SAR, then authorizes backup with the matching normal private key. The SPEC excludes setup-time Iso compromise or substitution. |
| **Trusted hardware** | Boomlet protects keys and protocol state; ST preserves keys, display and local input integrity. |
| **Build and update integrity** | Trusted components run authorized code and protect their keys; malicious builds or updates violate this assumption. |
| **WT / SAR service governance and accountability** | Service governance, protection against operator misuse and jurisdictional fit depend on operators. WT failover and resuming after prolonged setup-bound SAR unavailability are unspecified; changing SAR requires a fresh setup. |
| **Privacy and network exposure policy** | Tor, RPC, WT, SAR, and doxing-data flows lack a final minimization policy for metadata leakage, correlation, and acceptable exposure. |
| **Failure-domain independence** | Five peer identities do not establish five independent device, firmware, provisioning, RPC, WT, SAR, payment, or legal failure domains. |
| **Interrupted backup import and signed PSBT export** | A lost `BackupDone` can leave Boomlet and Boomletwo disagreeing about setup completion. Niso or WT can lose the signed PSBT fragment after Boomlet clears its copy. |
| **Dynamic rescue-data freshness** | Authenticated uploads append to retained history. Sequence numbers claim per-device submission order; payload truth, capture time, relevance and cross-device order remain application concerns. |
| **Consent-set recovery** | ST-approved rotation can replace an observed set and synchronize any existing backup. Private input is required; replacing a set after Boomlet is lost remains undefined. |

Iso is trusted throughout setup and backup; Boomlet and ST remain trusted
devices during operation.

### Accepted security decisions

| Decision | Security effect | Risk / gap links |
| --- | --- | --- |
| [ADR 0001](../adr/0001-setup-replay-and-phase-checkpoints.md) | Fresh peer setup nonces, deterministic setup identity, and chained phase checkpoints separate setup attempts and enforce phase order. | R-10, R-17 / DG-05 |
| [ADR 0002](../adr/0002-java-card-cryptographic-profile.md) | AES-256-CBC/PKCS#7 with AES-CMAC encrypt-then-MAC and SP 800-108 CMAC KDF defines the Java Card-compatible symmetric profile. | R-01, R-10, R-14 / DG-06 |
| [ADR 0003](../adr/0003-single-sar-per-peer.md) | Exactly one SAR identity is selected per peer and bound during setup; changing SAR requires a fresh setup and fund rollover. | R-06, R-13, R-26 / DG-13, DG-26 |
| [ADR 0004](../adr/0004-per-channel-directional-keys.md) | One directional four-key schedule per endpoint pair prevents reflection while envelope contexts separate messages and scopes. | R-10, R-14 / DG-17 |
| [ADR 0005](../adr/0005-user-chosen-doxing-password.md) | A user-chosen password is accepted without a protocol entropy threshold; offline-guessing risk is accepted. | R-07 / DG-11, DG-16 |
| [ADR 0006](../adr/0006-per-withdrawal-mystery-generation.md) | A fresh mystery is generated at `DIGGING` entry and erased with active withdrawal state, preventing a reached threshold from carrying into another ceremony. | R-18, R-25 / DG-30, DG-35 |
| [ADR 0009](../adr/0009-niso-relayed-consent-rotation.md) | Signed ST approval authorizes rotation through Niso, preserving duress state and receipt gates. | R-12, R-18, R-22 / DG-38, DG-39 |
| [ADR 0010](../adr/0010-consent-counter-display.md) | ST shows the counter as pending after recorded approval and as applied after authenticated completion; unchanged sets and cancellation after approval also advance it. | R-12, R-21 / DG-31, DG-38 |

### Protocol phase coverage

| Phase | Security-critical behavior | Main risks and gaps |
| --- | --- | --- |
| SAR registration | Phone authenticates the selected SAR over Tor. Registration atomically commits payment-bound static data, protocol version, upload key, first upload and receipt; later authenticated uploads append without replacing history. | R-06, R-07, R-20, R-26 / DG-11, DG-13, DG-16, DG-40 |
| Boomlet installation | Trusted Iso installs `normal_pubkey`, `doxing_key`, and one selected SAR; Boomlet generates identity and MuSig2-share keys. | R-01, R-19, R-24, R-27 / DG-07, DG-10, DG-34, DG-36 |
| ST enrollment | Two fresh nonce-bound country-grid challenges must resolve to the same five-element consent set. | R-02, R-11, R-12, R-21 / DG-08, DG-09, DG-31 |
| Peer records and setup ID | Signed records are sorted by canonical Boomlet identity bytes; Boomlet and ST independently recompute and verify `setup_instance_id`. | R-10, R-17, R-19 / DG-05, DG-06, DG-17 |
| Agreement and checkpoints | All five Boomlets sign one parameter fingerprint; WT, SAR, and backup import or signed skip advance a chained setup checkpoint. The final checkpoint does not establish backup presence. | R-05, R-10, R-14, R-17 / DG-04, DG-05, DG-12, DG-17 |
| WT and SAR finalization | WT is bound to the accepted setup; each peer's chosen SAR and identifier are setup-bound, checked by SAR, WT, and Boomlet. | R-05, R-06, R-13, R-26 / DG-12, DG-13, DG-24, DG-26 |
| Boomletwo backup | Iso signs the backup target or a request to skip backup. Import is target-bound and returns `BackupDone`; no `mystery` exists during setup. Retrying an interrupted import or a lost `BackupDone` remains unspecified. | R-18, R-19, R-27 / DG-04, DG-30, DG-36, DG-39 |
| Withdrawal review and approval | Every user independently verifies `tx_id`; `withdrawal_id` binds the initiator approval, and `approved_withdrawal_id` binds the canonical unanimous set. | R-08, R-10, R-15 / DG-21, DG-22, DG-31 |
| Commitment and initial duress | Every commit carries a fresh placeholder; WT waits for the exact setup-bound SAR acknowledgement before distributing commits. | R-12, R-13, R-21, R-22 / DG-23, DG-24, DG-29, DG-32 |
| Digging | Each Boomlet samples one fresh mystery on entry. Counter advancement is evaluated before every valid pong performs bounded `last_seen_block` catch-up, and occurs only when the local chain view advanced and every other peer's ping is within tolerance around the current local Niso height under a safe chain view. Reached peers keep participating. | R-03, R-05, R-09, R-13, R-25 / DG-01, DG-02, DG-24, DG-25, DG-33, DG-35 |
| Consent rotation | Signed ST approval binds the nonce and counter; any backup is synchronized within the setup-bound pair. Without a backup, completion commits locally. DIGGING pauses until fresh input and the exact resumption SAR receipt; IDLE rotation can defer delivery until a later approved withdrawal. | R-12, R-13, R-18, R-21, R-22 / DG-04, DG-30, DG-38, DG-39 |
| Hydration and MuSig2 signing | Hydration preserves the transaction and `SIGHASH_DEFAULT`; Boomlet revalidates the ceremony, while Iso verifies the local signing package and BIP327 session. | R-01, R-08, R-14, R-15 / DG-06, DG-22, DG-30 |
| Export, aggregation, and reset | Final SAR delivery and any paired reset precede clearing. No durable Niso or WT fragment receipt is required before cleanup; aggregation and broadcast follow. | R-05, R-13, R-14, R-25 / DG-24, DG-30, DG-39 |

## Architecture and trust boundaries

Detailed boundary table, flows, diagrams, data classification, parameters, and
binding rules: [architecture.md](architecture.md). Authorization and duress
depend on Boomlet and ST. Liveness and chain state depend on WT and Bitcoin
RPC. Coercion resistance also depends on physical security and operator
behavior.

<a id="human-physical-risk"></a>
## Human and physical risk

Physical coercion is part of the threat model. Human operators, facilities,
travel patterns, and operating discipline are all part of the attack surface.

### Human/physical threat scenarios

| Scenario                                                            | Trust boundaries involved          | Primary assets at risk                   | What can go wrong                                                                                    | Candidate mitigations, design and operations                                                                                                 |
| ------------------------------------------------------------------- | ---------------------------------- | ---------------------------------------- | ---------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| **Coercion or wrench attack** on one peer during withdrawal         | TB-PHYS-PEER, TB-ST, TB-BOOMLET    | Funds, user safety, duress secrecy       | Attacker forces approvals or continuation, tries to detect or bypass duress, or holds the victim until completion | Boomerang non-determinism; duress checks via ST; authenticated SAR delivery; external response; safe-room procedures; operator drills; split-knowledge procedures |
| **Coercion on multiple peers, kidnapping, or round-up**             | TB-PHYS-PEER, TB-OOB               | Funds, user safety                       | Attacker captures multiple peers, or learns enough identity and schedule data to stage a simultaneous round-up | Geographic dispersion; time-zone dispersion; compartmentalized peer identity sharing; travel OPSEC; independent alerting paths |
| **Insider collusion** among peers                                   | TB-OOB, TB-PHYS-PEER               | Funds, governance integrity              | Colluding peers wait out waterfall timelocks; or coordinate coercion                                 | Legal agreements; choose milestones to make late-stage far future; monitoring and alerts |
| **Social engineering, including phishing and fake ceremonies**      | TB-OOB, TB-PHYS-PEER, TB-SAR       | Normal keys, peer set integrity, rescue readiness, privacy | Steals mnemonics; reroutes peer or WT Tor addresses; tricks the operator into fake SAR enrollment or ceremony steps | Multi-channel fingerprint checks; pinned keys; out-of-band SAR identity checks and authenticated registration receipts; training; anti-phishing playbooks |
| **Credential theft, including mnemonic or passphrase compromise**   | TB-PHYS-PEER, TB-ISO               | Normal keys                              | Theft or coerced disclosure enables later deterministic theft once normal-key stages open            | Strong passphrases; split backups; secure storage; anti-phishing discipline |
| **Process non-compliance, such as missed rollover or skipped checks** | TB-PHYS-PEER, TB-OOB             | Funds, coercion resistance               | Users miss rollover windows or skip required checks, allowing the protocol to drift into weaker deterministic stages | Mandatory rollover schedule; automated reminders; rehearsals; documented exception handling and sign-off |
| **Operator error, including wrong milestones, outputs, or network** | TB-PHYS-PEER, TB-ISO, TB-ST        | Funds, liveness                          | Setup produces the wrong descriptor or milestones; withdrawal signs the wrong transaction or wrong network | Strict checklists; independent operator-side transaction review on dedicated hardware; testnet rehearsals; sanity checks and invariants in Iso |
| **Travel and commute risk, including surveillance or ambush**       | TB-PHYS-PEER                       | User safety, anonymity                   | Attacker identifies peer movements and targets vulnerable moments                                    | Vary routines; secure transport; avoid role disclosure; compartmentalize identities; personal-security procedures |
| **Site access failures, including unauthorized storage access**     | TB-PHYS-PEER                       | Devices, backups                         | Evil-maid installs implants or swaps devices; steals backups                                         | Access control, alarms, cameras; tamper seals; inventory |
| **Device theft or tampering of ST or Boomlet**                      | TB-PHYS-PEER, TB-BOOMLET, TB-ST    | Duress secrecy, protocol integrity, keys | Attacker tampers with trusted devices, learns consent-set material, or destroys Boomerang-state devices to force fallback | Certified secure element; tamper resistance; secure storage; incident response |
| **Environmental disasters, including fire, flood, or EMP**          | TB-PHYS-PEER                       | Devices, backups                         | Damage or destruction forces fallback or makes recovery impossible                                   | Geo-redundant storage; fireproof safes; Faraday protection; disaster recovery drills |
| **Power interruption during signing or backup procedures**          | TB-PHYS-PEER, TB-BOOMLET, TB-ISO   | Signing state, liveness                  | Interrupted sessions cause partial state, nonce reuse, or lost progress                              | Atomic signing sessions; anti-rollback; UPS; restart tests for interrupted signing and backup import |
| **Jurisdictional or legal coercion of SAR or operators**            | TB-JURIS, TB-SAR                   | PII, user safety, service availability   | SAR forced to reveal data, delay action, or refuse service; operator compelled disclosure            | Minimize PII; legal counsel; documented policies; transparency reports |

### Human-safety hazards

Financial risk scores do not represent injury or loss of life. Each hazard
below has a catastrophic worst case even when its probability is unknown.

| Hazard ID | Initiating condition | Worst credible outcome | Protocol bearing | Related risks and threats |
| --- | --- | --- | --- | --- |
| H-01 | A coercer observes or infers a duress signal. | Retaliation, escalation, serious injury, or death to the user or household members. | Message indistinguishability reduces protocol leakage but cannot control physical observation or an attacker's reaction. | R-12, R-22 / T-INFO-10, T-HUM-06 |
| H-02 | User error or a manipulated interaction produces a valid non-consent response. | Wrongful intervention, detention, injury, or exposure of an uninvolved user. | Boomlet rejects malformed input and records duress for a valid non-consent response. | R-21 / T-HUM-03, T-HUM-06 |
| H-03 | SAR acts on stale, false, incomplete or misattributed rescue data. | Response to the wrong location or person, with harm to the user, household members, responders, or bystanders. | Authenticated append preserves accepted encrypted uploads, but neither signatures nor receipts establish truthful, current payloads or complete SAR disclosure. | R-06, R-26 / T-DATA-02, T-PHONE-01 |
| H-04 | A SAR insider, compromised operator, or compelled authority obtains identity and rescue data. | Targeting, coercion, unlawful detention, retaliation, or later abuse. | SAR authority, conduct, jurisdiction, and retention lie outside the cryptographic guarantee. | R-06, R-20 / T-INFO-01, T-LEGAL-01 |
| H-05 | Coercion continues while the ceremony stalls or after a valid signal is acknowledged. | Prolonged detention, injury, death, or harm to other captives despite correct protocol behavior. | Non-determinism and delivery increase uncertainty; they do not guarantee release, rescue, or de-escalation. | R-03, R-13, R-22 / T-DOS-01, T-DOS-02, T-DOS-03 |

### Duress mechanism assumptions

The duress mechanism relies on three assumptions:

- The attacker cannot observe the ST display and the user's selection behavior
  well enough to learn the consent set.
- ST does not leak consent-set material.
- Users can reproduce the consent set reliably during legitimate safe use,
  including under stress or fatigue.

Cameras, multiple attackers, or a controlled environment can expose the
interaction. Shielding, private environments, and operator drills reduce that
risk but are not protocol controls.

A coercer can reuse an observed consent set until the user privately replaces
it with a different set under [SPEC Section 16.7](../spec/SPEC.md#167-consent-rotation).
Replacement requires Boomlet, ST and any existing backup.

SAR acknowledgement establishes receipt of the exact placeholder and durable
activation for a valid duress tuple. It does not establish successful rescue,
correct location, lawful authority, or a non-escalatory outcome. During IDLE
rotation, a wrong answer can record duress without contacting SAR until a later
approved withdrawal.

Primary coercion campaign: [Tree 0](attack_trees.md#tree-0-primary-attacker-campaign-steal-funds-under-coercion-or-force-deterministic-fallback).

### Operational hardening checklist

**People**
- Run regular coercion, recovery, and rollover drills.
- Minimize who knows peer identities and custody roles.

**Facilities**
- Use secure, access-controlled environments for setup, backup, and withdrawal ceremonies.
- Maintain tamper-evident seals and inspection logs for ST, Boomlet, and any Boomletwo.
- Keep Boomlet, any Boomletwo, and mnemonic backups in geographically separated storage.

**Travel**
- Avoid predictable patterns and role disclosure.
- Use secure transport and avoid carrying all critical devices or backups together.

**Incident response**
- Predefine device-loss, coercion-suspected, and missed-rollover playbooks.
- Maintain a rehearsed rollover or fund-movement plan before deterministic stages.
- Before every deposit, check final setup completion and remaining milestone runway.
  Later deposits inherit the same absolute heights; old UTXOs remain under the
  old descriptor until migration confirms.



## Threat catalog

Detailed `T-*` scenarios and STRIDE mappings:
[audit_mappings.md](audit_mappings.md). The `R-*` entries prioritize risk at the
design level.


<a id="risk-register"></a>
## Risk register

Risk fields follow NIST SP 800-30r1: threat event, vulnerability or
predisposing condition, likelihood, impact, and response. Each risk maps to
threat IDs and relevant custody-security controls.

### Scoring method

- **Likelihood (1–5):** Rare → Almost certain
- **Impact (1–5):** Negligible → Catastrophic
- **Risk score:** `Likelihood × Impact`
  - 1–5 Low, 6–10 Medium, 11–15 High, 16–25 Critical

Scores reflect SPEC controls and the stated trust assumptions.
Proposed roadmap work does not reduce a score.

`Status` indicates whether the present posture depends mainly on controls, assumptions, or unresolved design gaps.

The score ranks custody and protocol risk. It does not reduce the human-safety
hazards in H-01 through H-05 to financial likelihood and impact.

### Risk register table

| Risk ID | Risk | Score | Status | Canonical DG links | Notes |
| --- | --- | --- | --- | --- | --- |
| R-01 | Boomlet secure element compromise: key extraction or logic tamper | 15 | Assumption | DG-06, DG-07 | Assumes JavaCard-class extraction resistance, non-exportable Boomlet key shares, and fallback timelocks. |
| R-02 | ST compromise: tampering, code injection, or key extraction enabling duress bypass or consent-set exfiltration | 12 | Assumption | DG-08, DG-09, DG-31 | Depends on trusted ST firmware, non-extractable ST keys, air-gapped QR exchange, and tamper evidence. |
| R-03 | Forced determinism from device loss, bricking, or deliberate stalling | 15 | Gap | DG-03, DG-04, DG-30 | Without backup, one lost Boomlet can block the primary path; normal-key recovery cannot restore its share. Backup activation remains undefined; timely rollover remains critical. |
| R-04 | Malicious or compromised peer exploits later waterfall stages or governance gaps | 10 | Control | DG-03, DG-14, DG-15 | Early stages require 5-of-5; later recoverability stages need governance and rollover discipline. |
| R-05 | Watchtower compromise or malicious operation | 12 | Gap | DG-01, DG-12, DG-18, DG-34 | Active WT behavior remains central to freshness, routing, coordination, liveness, and metadata exposure. |
| R-06 | SAR compromise, coercion, or rescue-data misuse | 12 | Assumption | DG-11, DG-13, DG-16 | SAR stores encrypted history; a malicious SAR can still hide, lose or misuse it. Receipts prove individual acceptance, not truthful data or complete retention. Governance and jurisdiction remain external. |
| R-07 | Password-based `doxing_key` brute-force after SAR database or ciphertext exposure | 9 | Accepted by ADR 0005 | DG-11, DG-16 | ADR 0005 accepts user-chosen passwords without a protocol entropy threshold. Leaked identifiers, envelopes or stored upload authentication keys permit offline password checks. |
| R-08 | Niso compromise tampers PSBT presentation, hydration, or Tor coordination | 16 | Control | DG-10, DG-22, DG-31 | ST checks approved `tx_id`; Boomlet constrains hydration and revalidates before signing; Iso verifies its local signing package. |
| R-09 | False block-height or eclipse attacks manipulate freshness checks and counters | 12 | Gap | DG-01, DG-25 | Depends on chain-view trust, WT height, Niso RPC, and explicit reorg/freshness policy. |
| R-10 | Replay, spoofing, serialization, or desync of protocol messages | 12 | Control/Gaps | DG-05, DG-17, DG-21, DG-23, DG-24, DG-25, DG-29 | Core binding fields are specified. Parser safety depends on implementation; rules for retaining signed withdrawal messages, service timeouts and responses to peer silence remain unspecified. |
| R-11 | QR-code channel attacks through malformed payloads, parser bugs, swapping, or camera trickery | 12 | Control | DG-10, DG-31, DG-34 | Air-gap, signatures, and nonces help; parser hardening and operational QR controls remain necessary. |
| R-12 | Duress and rescue-signal leakage under observation, side channels, or coercion | 12 | Assumption | DG-15, DG-23, DG-29, DG-32, DG-38 | Safe and duress equivalence extends through rotation, cancellation, restart, status and reset. Private consent replacement can limit observed-answer reuse; continued observation or trusted-device compromise can defeat it. |
| R-13 | WT, SAR, peer availability, or fairness gaps block withdrawal or rescue acknowledgement | 12 | Gap | DG-12, DG-13, DG-14, DG-24, DG-39 | Withdrawal depends on WT, the setup-bound SAR and all peers. Rotation depends on ST and any existing backup; Niso can censor relays. Recorded rotation state and replies survive restart; timeouts, WT failover and recovery after interrupted backup import or PSBT export remain unspecified. |
| R-14 | Implementation bugs in cryptography, signing, or state binding | 15 | Assumption | DG-06, DG-17, DG-21, DG-22, DG-23, DG-29 | Depends on correct cryptography, nonce discipline, canonical decoding and scope checks. |
| R-15 | Operator and ceremony errors cause loss or forced determinism | 16 | Control | DG-02, DG-03, DG-10, DG-14, DG-15, DG-30 | User verification, signed fingerprints, multiparty agreement, backup, runbooks, and monitoring carry major risk. |
| R-16 | Deanonymization and targeting through Tor correlation, metadata leakage, or routine exposure | 8 | Control | DG-16, DG-18 | Tor and encrypted PSBTs reduce exposure; metadata minimization and payment unlinkability remain open. |
| R-17 | Out-of-band peer exchange compromise causes key substitution or MITM | 15 | Control | DG-05, DG-14, DG-17 | Signed `PeerSetupRecord` values, canonical Boomlet ordering, and independent Boomlet/ST recomputation of `setup_instance_id` still depend on users obtaining the intended records out of band. |
| R-18 | Boomletwo backup misuse bypasses non-determinism or clones state | 12 | Gap | DG-04, DG-30, DG-35, DG-39 | Applies where a backup exists. Setup backup contains no `mystery`. Interrupted rotation may leave older consent on Boomletwo. Activating Boomletwo without allowing both devices to sign, and retrying interrupted imports, remain unspecified. |
| R-19 | Device substitution during setup or withdrawal | 8 | Control | DG-07, DG-08, DG-09, DG-10 | Identity binding and user verification exist; hardware provenance and tamper evidence remain important. |
| R-20 | Logging and monitoring data leaks from WT, SAR, or Niso | 12 | Assumption | DG-16, DG-18, DG-34 | Logging, retention, and monitoring controls depend mainly on operator practice and service hardening. |
| R-21 | False duress from user mistake triggers SAR escalation | 6 | Control | DG-13, DG-32, DG-38 | Boomlet rejects malformed index sets and records duress for any valid set unequal to the consent set. Observation, stress errors, false-positive handling, and the consequences of intervention remain external. |
| R-22 | False negative duress: delivery, acknowledgement, or response fails | 15 | Gap | DG-13, DG-23, DG-24, DG-29, DG-32, DG-39, DG-40, DG-41 | SAR acknowledges exact placeholders; IDLE duress delivery waits for a later approved withdrawal. SPEC must explicitly require verification of the SAR receipt for Boomlet's last reached-ping placeholder before signing. SAR availability, data interpretation and physical intervention remain safety-critical. |
| R-23 | WT service-fee payment metadata links peer identity to custody participation | 9 | Control | DG-12, DG-16, DG-18 | WT invoices and receipts create privacy pressure; payment unlinkability remains an operational control. |
| R-24 | Software or firmware supply-chain compromise or malicious updates | 10 | Assumption | DG-07, DG-08, DG-34 | Build provenance, release signing, update integrity, and dependency control are trusted at this stage. |
| R-25 | Digging-game timing, reorg recovery, or stalling edge cases | 12 | Gap | DG-01, DG-02, DG-24, DG-25, DG-33, DG-35 | Full-peer evidence and bounded catch-up fix the recorded stale-lock and counter-acceleration defects. Reached peers continue participating. Joint timing constraints and reorg recovery remain undefined. |
| R-26 | SAR enrollment or stale rescue data creates false coverage | 12 | Control and assumptions | DG-11, DG-13, DG-26, DG-40 | Registration replay and history rollback gaps resolved in bf74a71241c3c3e5f39ec5cdd5108f031ebc4639. Setup binds SAR and static data; backup verifies them when performed. Data truth, freshness and effective SAR response remain assumptions. |
| R-27 | Iso compromise or substitution during setup installs attacker-chosen authority or rescue state | 15 | Assumption | DG-10, DG-34, DG-36 | Iso supplies `normal_pubkey`, `doxing_key`, and the selected SAR, and later authorizes backup. The SPEC explicitly excludes setup-time Iso compromise from its threat boundary. |
| R-28 | Common-mode failure or a cross-boundary coalition defeats assumed separation | 20 | Assumption | DG-12, DG-13, DG-18, DG-34, DG-37 | DG-37 records completed modeling coverage. Independent secrets and failure domains remain assumptions; shared devices, software, services or legal exposure can compromise several peers. Shared mnemonics can reduce effective thresholds. |


## Attacker paths

Attack-tree source: [attack_trees.md](attack_trees.md). Covered campaigns:

1. theft under coercion or forced deterministic fallback;
2. PSBT tampering and broken transaction-intent continuity;
3. deliberate entry into deterministic fallback;
4. suppression or abuse of duress signaling and rescue;
5. peer deanonymization followed by targeted coercion;
6. Boomlet, ST, or Iso supply-chain compromise; and
7. common-mode compromise or a cross-boundary coalition.


<a id="mitigations-roadmap"></a>
## Mitigations & roadmap

### Mitigation roadmap

| Stage | Workstream | DG / Risk links | Verification focus | Resolution |
| :--- | :--- | :--- | :--- | :--- |
| Completed | Generate each `mystery` at `DIGGING` entry and erase it with active withdrawal state. | DG-35 / R-25,R-18 | Abort/retry tests, state-lifetime tests, and checks that no setup or backup object contains `mystery`. | Resolved in f74a8f0ce2563d0f2b68c50ff876dfa14f62f398 by ADR 0006 and the per-withdrawal state transition. |
| Now | Specify the canonical version, backup replay encoding and numeric parser limits. | DG-06 / R-14,R-10,R-25 | Independent crypto review, interop vectors, strict-decoder fuzzing, and per-schema limit tests. | The primitive and binding profile was fixed in 96734855a35db6af7c6d8be06d72d452433bbc6b; normal-key derivation was fixed in ec72f17fc97b31871e1b6c72f0c08e83a30ce436; canonical schema encoding was completed in 5616ca0efffa7f2f6f73008240a225064e6aaf77; remaining SPEC/diagram mismatches in normal derivation and the WT-to-SAR response were corrected in 4a44d069c7412637a9020951d831d1594d344276. Version encoding, backup `replay_state` encoding and numeric size and nesting limits remain open (SPEC Sections 10 and 22). |
| Completed | Preserve the ST and Boomlet checks that hold transaction intent from approval through broadcast. | DG-22,DG-31 / R-08,R-15 | UX review, adversarial `tx_id` tests, SIGHASH tests, and operator verification tests. | Resolved in 96734855a35db6af7c6d8be06d72d452433bbc6b by withdrawal IDs, PSBT hydration constraints, and signing-package verification; `SIGHASH_DEFAULT` was fixed in 267ae5d5219b85f7a94e763ad073a24e2524838c. |
| Now | Harden Niso as a dedicated appliance. | DG-10,DG-22,DG-34 / R-08,R-20,R-11 | Hardening benchmark, penetration test, and malware simulation. | Implementation control; Niso remains an untrusted host. |
| Now | Harden trusted Iso provisioning and make setup authority independently auditable. | DG-10,DG-34,DG-36 / R-19,R-24,R-27 | Media provenance, device-authentication tests, derived-key and SAR cross-checks, and adversarial provisioning review. | Trusted Iso installation, setup, backup and signing are explicit assumptions under SPEC Section 5.3. |
| Now | Define and rehearse incident response for coercion, device loss, WT/SAR outage, and approaching milestones. | DG-03,DG-12,DG-13,DG-15,DG-24 / R-15,R-03,R-22,R-13 | Tabletop exercises, runbook reviews, and drills. | Operational control. |
| Now | Formalize rollover policy, milestone monitoring, alerts, reminders, and dashboards. | DG-02,DG-03,DG-15 / R-15,R-03,R-04 | Operational audit, testnet rollover drill, and monitoring tests. | Operational control; parameter selection remains open under DG-02. |
| Before production | Define recovery from WT outages under the setup binding. | DG-12,DG-16,DG-18,DG-34,DG-37 / R-05,R-13,R-16,R-28 | Outage and restart tests, setup-binding checks, WT security audit and coalition scenarios. | Open. |
| Before production | Design redundant SAR operations where policy permits, with audited access and uncertain-duress playbooks. | DG-13,DG-23,DG-24,DG-32,DG-37 / R-06,R-22,R-21,R-28 | Fixed-deadline and deadline-overrun tests, durable-path equivalence, end-to-end duress drills, privacy audit, and false-positive exercises. | Exact acknowledgement binding and observable-behavior rules were resolved in 96734855a35db6af7c6d8be06d72d452433bbc6b; the fixed release deadline and identical durable-write path were added in 40e09c3b4c9162d6d9870092e49f3d6241e82e48. SAR conduct and physical response remain external dependencies. |
| Before production | Formalize Boomletwo activation and recovery while preserving the one-active-device invariant. | DG-04,DG-30,DG-39 / R-18,R-03 | First-target retention before export, conflicting targets, exact retries, lost `BackupDone`, stale consent and single-active-device recovery. | Open. |
| Before production | Define recovery for signed PSBT fragments lost after Boomlet cleanup; specify verification of the SAR receipt for Boomlet's last reached-ping placeholder before signing. | DG-30,DG-39,DG-41 / R-13,R-14,R-22,R-25 | Crash-point and fragment-retention tests; withhold the last reached-ping SAR receipt before signing, including restart and direct calls; test WT completion during rotation and exact reset receipts. | SAR activation and replay resolved in 40e09c3b4c9162d6d9870092e49f3d6241e82e48; signed-fragment recovery remains open. DG-41 tracks the separate signing check. |
| Completed | Specify authenticated rescue-upload history, conflict handling and retries. | DG-40 / R-06,R-22,R-26 | Concurrent append, original-receipt retries, out-of-order acceptance, old and replacement Phones, supported-failure restoration, full-history access and shared-resource leakage. | Resolved in bf74a71241c3c3e5f39ec5cdd5108f031ebc4639 by retained authenticated history, conflict rejection and exact-retry receipts (SPEC Section 13.1). |
| Completed | Specify consent-set replacement and model common-mode and coalition attacks. | DG-37,DG-38 / R-12,R-21,R-28 | With and without backup, including mixed peer choices; private and observed rotation; identical sets; cancellation and status budgets; rollback, backup-presence metadata, lost ST replies and withheld backup or SAR receipts. | Rotation resolved in bf74a71241c3c3e5f39ec5cdd5108f031ebc4639 and extended to optional backup in f47c26a5f477cfd00e186b49bedc6dd5df9fe335; modeling coverage completed in ada19baa5c74d225280a307574f178cfb652c1ea. |
| Later | Add advanced privacy protections for traffic shape, payment unlinkability, and anonymity-set management. | DG-16,DG-18 / R-16,R-23,R-20 | Traffic analysis tests, chain analysis, and privacy audit. | Open. |

### Component-specific hardening summary

**Boomlet**
- Secure-element selection and evaluation
- Anti-rollback counters
- Strict parsing
- Signed applets
- Minimal command surface
- Protect retained transcript bindings and replay state

**ST**
- Secure boot
- Signed firmware
- No radios
- Map protocol primitives and key protection to the actual hardware and software.
- Hardened QR parser
- Anti-shoulder-surf UI
- Physical tamper evidence

**Iso**
- True air-gap
- Minimal OS
- Deterministic builds
- Controlled media only
- Deterministic key-derivation test vectors and fingerprint displays

**Niso**
- Hardened appliance
- Minimal services
- Strict network egress
- Disk encryption
- Log minimization
- Safe QR display and capture handling

**WT**
- Key protection
- Metadata minimization
- Rate limiting
- Onion-service hardening
- Privacy-preserving logging and retention limits
- Outage monitoring and recovery planning

**SAR**
- PII minimization
- Group related activation records for dispatch without deleting events
- Retain uploads and receipts while a related setup may remain active
- Key protection and audited signing-key management
- Audited access
- Coverage-confirmation checks
- Redundant on-call coverage where policy permits


<a id="audit-mapping-summary"></a>
## Audit framework

- **STRIDE:** Spoofing and tampering center on identity binding, setup records, PSBT authorization, and WT-mediated collections.
- **Information disclosure:** Main exposure paths are SAR/WT metadata, Tor correlation, doxing-data handling, and Niso or phone logs.
- **Denial of service:** WT/SAR availability, peer non-cooperation, Tor availability, Bitcoin fee pressure, and Boomlet/ST failure drive liveness risk.
- **Elevation of privilege:** Secure-element compromise, ST firmware compromise, QR parser bugs, and host-driven state-machine abuse dominate escalation risk.
- **OWASP:** WT, SAR, the phone app, and networked Niso inherit ordinary application-security duties: secure design, input validation, access control, logging, DoS resistance, update integrity, and secure configuration.
- **CCSS:** Boomerang targets CCSS Level III expectations for key generation, key storage, backup, service-provider controls, operator environment, auditing, updates, incident response, and data retention.


<a id="appendix-design-gaps"></a>
## Appendix D: Detailed design gaps

### Protocol, timing, and transaction semantics

| Gap ID | Design gap | Why it matters | Priority | Status / Resolution |
| --- | --- | --- | --- | --- |
| DG-01 | Reorg and divergent-chain recovery are incomplete | Boomlet stalls with `CHAIN_VIEW_UNSAFE` on height decrease or material RPC/WT disagreement, but the trusted chain-source policy and recovery procedure are not defined. | Critical | Partly resolved by the explicit unsafe-chain checks in 96734855a35db6af7c6d8be06d72d452433bbc6b; rules for Boomlet to resume from `CHAIN_VIEW_UNSAFE` remain unspecified. |
| DG-02 | No parameter-selection framework for milestones, mystery range, freshness windows, and SAR acknowledgement delay | Milestones, mystery bounds, catch-up, freshness and SAR deadlines need joint constraints and production values, including a SAR processing bound and matching WT timeouts. | Critical | Open |
| DG-03 | Forced determinism is controlled mostly by user discipline and peer behavior | Late start, device loss, and non-cooperation are collapse paths, and the mitigation depends mainly on timely rollover and careful operation. | Critical | Protocol limitation; timely rollover and peer cooperation are operational assumptions. |
| DG-05 | Setup uniqueness and anti-replay | `setup_instance_id`, `peer_setup_nonce`, and chained setup checkpoints separate one setup from another. | High | Resolved in 96734855a35db6af7c6d8be06d72d452433bbc6b by `setup_instance_id`, `peer_setup_nonce`, and setup checkpoints. |
| DG-06 | Cryptographic conformance evidence is incomplete | The cryptographic suite, canonical schemas and context binding are specified. `PROTOCOL_VERSION` lacks a fixed canonical type and value; backup `replay_state` lacks a defined encoding, and numeric size and nesting limits remain unselected. | Critical | Primitive and binding choices were resolved in 96734855a35db6af7c6d8be06d72d452433bbc6b; normal-key derivation was fixed in ec72f17fc97b31871e1b6c72f0c08e83a30ce436; canonical schema encoding was added in 5616ca0efffa7f2f6f73008240a225064e6aaf77; remaining SPEC/diagram mismatches were corrected in 4a44d069c7412637a9020951d831d1594d344276. Version encoding, backup `replay_state` encoding and numeric size and nesting limits remain open (SPEC Sections 10 and 22). |
| DG-17 | Message binding | Canonical envelope contexts, scope IDs, and directional channel keys bind messages to their channel and purpose. | High | Resolved in 96734855a35db6af7c6d8be06d72d452433bbc6b by canonical envelope contexts, scope IDs, and directional channel keys. |
| DG-21 | Withdrawal-session binding | `withdrawal_id` and `approved_withdrawal_id` keep approvals, commitments, pings, signing readiness, and replay scope on the same withdrawal attempt. | Critical | Resolved in 96734855a35db6af7c6d8be06d72d452433bbc6b by `withdrawal_id` and `approved_withdrawal_id`. |
| DG-22 | Transaction-authorization binding | PSBT hydration constraints, reached-collection checks, signing-package verification, `SIGHASH_DEFAULT`, and final `tx_id` equality preserve the approved transaction. | High | Resolved in 96734855a35db6af7c6d8be06d72d452433bbc6b by PSBT hydration constraints and signing-package verification; strengthened in 267ae5d5219b85f7a94e763ad073a24e2524838c by the fixed sighash policy. |
| DG-23 | SAR acknowledgement binding | SAR signs the exact encrypted placeholder under the active withdrawal context, with retry and replay rules. | High | Resolved in 96734855a35db6af7c6d8be06d72d452433bbc6b by exact placeholder acknowledgements, retry/failure handling, and replay memory. |
| DG-24 | Honest-path liveness lacks interoperable timeout and fairness policy | Failure classes, retry restrictions, stalling, and explicit abandonment are defined, but peer/service progress, timeout schedules, blame, WT failover, and response to prolonged SAR unavailability are not. The setup-bound SAR cannot be replaced within the existing setup. | High | Partly resolved in 96734855a35db6af7c6d8be06d72d452433bbc6b; timeout and fairness policy remain open. |
| DG-25 | Freshness recovery and evidence retention are incomplete | Boomlet stalls on unsafe chain views. How peers resume after that stall, and which signed messages and block heights they retain, are unspecified. | High | Progression and bounded catch-up corrected in 4a549ddbbdf0db028cbacd3a46e094fd761bbb3b and bf74a71241c3c3e5f39ec5cdd5108f031ebc4639 (SPEC Section 15.9); recovery from unsafe chain views and transcript retention remain unspecified. |
| DG-28 | Mid-ceremony interaction between boomerang progress and fallback opening is under-specified | The design does not define one canonical operator procedure for overlap between boomerang progress and deterministic fallback availability. | High | Open |
| DG-29 | Placeholder-instance lifecycle | Every placeholder has a fresh IV, an `approved_withdrawal_id`, and a SAR replay tuple. | High | Resolved in 96734855a35db6af7c6d8be06d72d452433bbc6b by per-instance IVs, `approved_withdrawal_id` binding, and SAR replay tuples. |
| DG-30 | Post-withdrawal cleanup and reset semantics are partly specified | For approved withdrawals, Boomlet clears withdrawal state only after SAR acknowledges the final ping's placeholder and any required Boomletwo reset completes. Niso or WT can still lose the exported signed PSBT fragment. | High | Mystery cleanup resolved in f74a8f0ce2563d0f2b68c50ff876dfa14f62f398. Final-delivery and paired-reset gates added in bf74a71241c3c3e5f39ec5cdd5108f031ebc4639 and updated for optional backup in f47c26a5f477cfd00e186b49bedc6dd5df9fe335 (SPEC Section 16.7). Recovery is unspecified if Niso or WT loses the signed PSBT fragment after Boomlet clears its copy. |
| DG-31 | ST prompt and display conformance are incomplete | The SPEC defines nonce-bound transaction review, five-column country mapping, indexed responses and the shared 193-label vocabulary. | High | Resolved at protocol level in 96734855a35db6af7c6d8be06d72d452433bbc6b; response typing and index validation corrected in 935149873b94a10b9e9452dc6572a0f8c82071bc (SPEC Sections 9.8, 13.3, 15.1 and 16.1–16.2). |
| DG-32 | End-to-end duress indistinguishability timing and error discipline | Safe and duress flows require equivalent lengths, timing, durable writes, retries, failures and telemetry, including rotation, cancellation, status recovery and reset. | Critical | Resolved at protocol level in 96734855a35db6af7c6d8be06d72d452433bbc6b; strengthened in 40e09c3b4c9162d6d9870092e49f3d6241e82e48 by fixed deadlines and identical durable writes, and extended to rotation and recovery in bf74a71241c3c3e5f39ec5cdd5108f031ebc4639 (SPEC Section 16.7). |
| DG-33 | Recurring-duress random trigger semantics are not protocol-canonical | SPEC Section 15.10 defines one uniform draw from 1 through the configured interval per new valid pong, a check on 1, and retry reuse. | High | Resolved in f47c26a5f477cfd00e186b49bedc6dd5df9fe335 by the explicit draw domain, trigger and retry rules (SPEC Sections 9.2 and 15.10). |
| DG-35 | Mystery lifetime across withdrawal attempts | A threshold learned from `reached_mystery_flag` must not survive abort, reset, backup, or a later ceremony. | Critical | Resolved in f74a8f0ce2563d0f2b68c50ff876dfa14f62f398 by generating `mystery` only on entry to `DIGGING` and erasing it with active withdrawal state. |
| DG-38 | Consent-set compromise has no rotation or recovery path | ST-approved rotation replaces the consent set, completing locally without backup or synchronizing an existing backup. | Critical | Resolved in bf74a71241c3c3e5f39ec5cdd5108f031ebc4639 by SPEC Section 16.7 and ADRs 0009–0010; extended to operation without backup in f47c26a5f477cfd00e186b49bedc6dd5df9fe335. |
| DG-39 | Crash-consistent protocol handoffs are incomplete | Registration, append, rotation and SAR activation have durability and retry rules. Recovery after interrupted Boomletwo import or loss of an exported signed PSBT fragment is unspecified. | Critical | Partly resolved in 40e09c3b4c9162d6d9870092e49f3d6241e82e48 for SAR durable activation, bf74a71241c3c3e5f39ec5cdd5108f031ebc4639 for registration, append and rotation, and f47c26a5f477cfd00e186b49bedc6dd5df9fe335 for atomic rotation recovery outputs (SPEC Sections 13.1 and 16.4–16.7); retrying an interrupted Boomletwo import and recovering a signed PSBT fragment lost after Boomlet cleanup remain unspecified. |
| DG-41 | Ordinary last reached-ping SAR gate needs explicit normative coverage | The diagram requires Boomlet to verify pending SAR acknowledgments before signing. SPEC Sections 15.11–15.13 need an explicit SAR-receipt check for the placeholder in Boomlet's last reached ping before MuSig2 signing. | High | Open; reconcile with withdrawal diagram step 61. |

### Boomlet, ST, and endpoint platform assurance

| Gap ID | Design gap | Why it matters | Priority | Status / Resolution |
| --- | --- | --- | --- | --- |
| DG-04 | Boomletwo activation, anti-clone, and interrupted-import semantics are unspecified | After an interrupted import or lost `BackupDone`, retries must preserve Boomlet's accepted target and request. Procedures for activating or revoking Boomletwo and preventing both devices signing concurrently remain undefined. Reconcile Section 17's backup-receipt prerequisite with Section 13.10's signed-skip path. | Critical | Open |
| DG-07 | Boomlet assurance is assumed, not demonstrated | Boomlet is assumed to protect keys, mystery and duress state against extraction, rollback, bias and unauthorized use. | Critical | Assumption; AR-22, AR-23 and AR-30. |
| DG-08 | The ST trust model is stronger than the current prototype evidence | ST is assumed to provide the required cryptography, protected keys and faithful display and local input. Compromise can reveal consent or falsify user intent. | Critical | Assumption; AR-31, AR-32, AR-33 and AR-38. |
| DG-09 | Prototype attack surface is not converted into a hardened ST build profile | Boot, debug and peripheral interfaces must not bypass ST key protection or trusted input. | High | Assumption; AR-34, AR-39 and AR-75. |
| DG-10 | Endpoint transition and replacement procedures are missing | SPEC Section 13.1 defines replacement Phone append with a fresh device ID, but retains the shared upload key. Niso and ST replacement are undefined; moving devices between hosts relies on correct user connections. | High | Phone replacement specified in bf74a71241c3c3e5f39ec5cdd5108f031ebc4639 (SPEC Section 13.1); Niso and ST replacement remain open. |
| DG-36 | Trusted Iso setup and provisioning boundary lacks assurance | Iso installs `normal_pubkey`, `doxing_key`, and the selected SAR, handles setup and backup secrets, and authorizes backup with `normal_privkey`. A substituted Iso can establish attacker-chosen long-lived authority before later checks begin. | Critical | Assumption; setup-time Iso compromise is excluded by SPEC Section 5.3, made explicit in 2fb53eaa79e0f5dddf7e98b7b4f1edca82f6674e. |

### Services, operators, and recovery

| Gap ID | Design gap | Why it matters | Priority | Status / Resolution |
| --- | --- | --- | --- | --- |
| DG-34 | Software supply chain and service implementation security remain under-specified | Trusted components run authorized code and protect service keys. Compromised builds or update authority can bypass protocol checks. | High | Assumption; AR-76. |
| DG-11 | Dynamic doxing data integrity and confidentiality are largely delegated to Phone, password choice, and SAR | Replacing newer rescue data with an old valid upload can misdirect rescue. Data truth and confidentiality also depend on Phone, SAR and password choice. | High | Partly resolved in bf74a71241c3c3e5f39ec5cdd5108f031ebc4639 by authenticated append, retained history and conflict rejection (SPEC Section 13.1; DG-40). Payload truth and SAR conduct remain assumptions; password risk is accepted by ADR 0005. |
| DG-12 | WT is a critical coordination dependency without a finalized failover or accountability model | WT is non-custodial but central to liveness, freshness, routing, and metadata exposure, and switching or redundancy procedures remain deferred. | Critical | Open |
| DG-13 | SAR conduct, effectiveness, and jurisdiction remain external assumptions | For duress, SAR acknowledges delivery only after storing an activation record. Lawful, timely, correctly directed, effective and non-escalatory intervention depends on SAR operations. SAR misuse, compulsion, insider abuse, and harmful response remain possible. | Critical | External dependency; physical SAR response is outside protocol guarantees. |
| DG-14 | Peer governance is under-specified for blame, timeout, and non-cooperation | All five peers must approve and continue participating. Refusal stalls progress; governance and responses to non-cooperation are external. | High | External dependency; cooperation and dispute handling are outside cryptographic enforcement. |
| DG-15 | Human availability is not reconciled with random duress checks and freshness limits | The design wants unpredictable checks, but users sleep, travel, live in different time zones, and can be delayed under ordinary conditions. | High | Operational assumption; users must be available for challenges within the chosen freshness limits. |
| DG-26 | Single-SAR-per-peer contract | Each peer selects one SAR identity, and SAR routing is bound during setup. | High | Resolved in 96734855a35db6af7c6d8be06d72d452433bbc6b by ADR 0003 and setup-bound SAR routing. |
| DG-37 | Common-mode failures and cross-boundary coalitions are not modeled | N-of-N resists independent peer compromise, but shared devices, software, provisioning, RPC sources, services, payment trails, or legal authority can correlate failures. | Critical | Modeling coverage completed in ada19baa5c74d225280a307574f178cfb652c1ea by Tree 6, T-COMP-01, T-COMP-02 and FM-29. Failure-domain independence remains an operational assumption under AR-78. |
| DG-40 | Dynamic rescue-data freshness and rollback semantics are undefined | Authenticated append, upload identities, conflict rejection, submission sequences and exact retry receipts define history. | Critical | Resolved in bf74a71241c3c3e5f39ec5cdd5108f031ebc4639 by authenticated append, conflict rejection and exact retry receipts (SPEC Section 13.1). |

### Privacy and network exposure

| Gap ID | Design gap | Why it matters | Priority | Status / Resolution |
| --- | --- | --- | --- | --- |
| DG-16 | Privacy leakage is not governed by a minimization strategy | SPEC Section 20 limits collection and excludes secrets from logs. Operators choose how long logs are kept and who can read them; traffic and payments can still link peers. | High | Baseline privacy rules specified in 96734855a35db6af7c6d8be06d72d452433bbc6b and strengthened for uploads and logs in bf74a71241c3c3e5f39ec5cdd5108f031ebc4639 (SPEC Section 20); rules for log retention, log access and service metadata collection remain unspecified. |
| DG-18 | The network trust model is incomplete | Reachability, traffic-correlation resistance and RPC integrity are external dependencies. Authenticated addresses do not establish operator identity. | Medium | Assumption; AR-64, AR-65 and AR-69. |
