# Boomerang Threat Model

> **Last change — 2026-07-14:** synced with the latest SPEC.md

Related material: [architecture](architecture.md), [attack trees](attack_trees.md),
and [threat mappings](audit_mappings.md). Appendix D remains the canonical
record of design-gap status and resolution evidence.

## Scope and assumptions

### System summary

Boomerang is a Bitcoin cold-storage protocol for cases where physical coercion
is a credible threat. Each private threshold is sampled from a bounded range,
but the signing point is not known in advance. The ceremony itself can stall on
peer, WT, SAR, Tor, RPC, chain-view, or device failure. Duress signaling runs
through the ordinary withdrawal flow.

The profile in `spec/SPEC.md` has exactly five peers and a 5-of-5 Boomerang
branch. A ceremony uses one WT; each peer selects one SAR, has one active
Boomlet, and may keep one inactive Boomletwo backup. A different peer count,
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
the same bounded durable-write path. The acknowledgement proves protocol-level
delivery and durable activation, not that intervention will be timely, lawful,
correctly directed, effective, or safe.

Setup state is tied together by `setup_instance_id`, signed `PeerSetupRecord`
values, and chained `setup_checkpoint` values. Withdrawal approval uses
`withdrawal_id`; everything after unanimous approval uses
`approved_withdrawal_id`, including commitments, placeholders, pings, pongs,
reached-ping collections, signing, export, and replay state. WT may not advance
to commit and SAR processing until it has verified the non-initiators'
approval-set attestations.

### In-scope components

Peers are joint custodians of the bitcoin involved. Per peer or operator, with exactly five peers in the profile:

- **User:** the human operator.
- **Iso, offline:** trusted key-derivation, Boomlet installation, backup authorization and verification, and final-signing environment. Compromise or substitution during setup is outside the SPEC threat boundary.
- **Boomlet, secure element:** holds key share, enforces non-determinism, and runs duress logic.
- **Boomletwo, backup secure element:** authenticated target-bound import of long-lived Boomlet setup state. Setup and backup carry no `mystery`; a fresh threshold is generated only when an activated device enters `DIGGING`. Activation remains under design.
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
- **Static doxing data** and **dynamic doxing feed**

**Protocol state assets**
- Per-withdrawal, per-Boomlet `mystery`, `counter`, reached state, and `ping_seq_num`
- `boomerang_params` and `boomerang_descriptor`
- `setup_instance_id`, `setup_checkpoint`, `withdrawal_id`, and `approved_withdrawal_id`
- Approval collections, approval-set attestations, commits, pings, pongs, reached-ping collections, and signing-session state
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
- build, update, and deployment paths for Iso, Niso, ST, Boomlet, WT, and SAR
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

### Key unknowns blocking production readiness

Production readiness requires:

- Exact values and selection policy for the non-determinism and duress-check parameters
- A safe `sar_placeholder_ack_delay`, matching WT timeout, and demonstrated
  worst-case SAR pre-acknowledgement processing bound
- Reorg-handling policy
- WT / SAR redundancy model
- Cryptographic implementation evidence: canonical-encoding conformance, CBC/CMAC/KDF test vectors, interoperable signature/envelope vectors, and side-channel-safe implementation profiles
- Boomletwo activation / recovery protocol
- Software supply chain and update mechanisms
- Monitoring, alerting, and incident-response workflows

### Open design boundaries

The protocol leaves these boundaries open or dependent on deployment controls.

| Boundary | Why it matters |
| --- | --- |
| **Chain view and reorg policy** | The protocol detects local or WT height decreases and material RPC/WT disagreement and stalls with `CHAIN_VIEW_UNSAFE`; reorg recovery, chain-source trust, and divergent-node policy remain undefined. |
| **Setup / withdrawal binding** | Core binding is specified through `setup_instance_id`, chained setup checkpoints, `withdrawal_id`, and `approved_withdrawal_id`. |
| **Hydrated PSBT authorization contract** | Transaction authorization is specified through `tx_id` continuity, signing-package verification, descriptor and transaction checks, and final broadcast `tx_id` checks. |
| **Boomlet and Boomletwo activation semantics** | Target-bound backup export and import exist, but activation, deactivation, revocation, and the one-active-device invariant remain undefined. No mystery exists at backup time. |
| **Duress-placeholder lifecycle** | Fresh envelopes, `approved_withdrawal_id` context, exact-envelope SAR signatures, replay tuples, a fixed release deadline, and the same pre-acknowledgement durable-write path bind the lifecycle. Deployment timing evidence remains necessary. |
| **Operational transitions** | Device movement, replacement, and recovery for Phone, Niso, ST, and WT selection remain only partly specified. Response to prolonged unavailability of the fixed setup-bound SAR is also incomplete. |
| **Honest-path liveness / fairness policy** | Failure classes, retry restrictions, stall, and explicit abandonment are defined. Interoperable timeouts, blame, peer non-cooperation, WT failover, and response to prolonged SAR unavailability remain undefined. |
| **Parameter selection and timing policy** | Security depends on workable choices for milestones, mystery range, duress cadence, and freshness tolerances. |
| **Cryptographic profile and canonical transcript encoding** | The profile specifies AES-256-CBC with PKCS#7, AES-CMAC encrypt-then-MAC, SP 800-108 CMAC KDF context separation, directional channel schedules, canonical object bytes, schema IDs, and scope-specific contexts; complete vectors and per-schema limits remain open. |
| **Trusted Iso setup boundary** | Iso chooses and installs `normal_pubkey`, `doxing_key`, and the selected SAR, then authorizes backup with the matching normal private key. The SPEC excludes setup-time Iso compromise or substitution. |
| **Trusted-hardware assurance boundary** | Boomlet and ST are treated as trusted boundaries; hardware assurance remains outside protocol design. |
| **Software supply chain and service implementation security** | Build, update, deployment, and ordinary service-hardening controls remain outside protocol design. |
| **WT / SAR service governance and accountability** | WT and SAR behavior, WT failover, response to setup-bound SAR unavailability, misuse resistance, and jurisdictional fit remain only partly specified. Changing SAR requires a fresh setup. |
| **Privacy and network exposure policy** | Tor, RPC, WT, SAR, and doxing-data flows lack a final minimization policy for metadata leakage, correlation, and acceptable exposure. |
| **Failure-domain independence** | Five peer identities do not establish five independent device, firmware, provisioning, RPC, WT, SAR, payment, or legal failure domains. |
| **Crash-consistent handoffs** | Backup import, SAR acknowledgement release, and signed-fragment export can cross a durable-state boundary before the sender knows that the receiver can resume safely. |
| **Dynamic rescue-data freshness** | `captured_at` records when Phone obtained dynamic data, but the protocol does not define ordering, expiry, rollback rejection, or conflict resolution. |
| **Consent-set recovery** | A consent set learned through observation remains valid for the setup; no rotation or recovery procedure is defined. |

Iso is trusted throughout setup and backup; Boomlet and ST remain trusted
devices during operation. The cited prototype materials do not establish the
Boomlet or ST assurance assumed here.

### Accepted security decisions

| Decision | Security effect | Risk / gap links |
| --- | --- | --- |
| [ADR 0001](../adr/0001-setup-replay-and-phase-checkpoints.md) | Fresh peer setup nonces, deterministic setup identity, and chained phase checkpoints separate setup attempts and enforce phase order. | R-10, R-17 / DG-05 |
| [ADR 0002](../adr/0002-java-card-cryptographic-profile.md) | AES-256-CBC/PKCS#7 with AES-CMAC encrypt-then-MAC and SP 800-108 CMAC KDF defines the Java Card-compatible symmetric profile. | R-01, R-10, R-14 / DG-06 |
| [ADR 0003](../adr/0003-single-sar-per-peer.md) | Exactly one SAR identity is selected per peer and bound during setup; redundancy and replacement are not provided. | R-06, R-13, R-26 / DG-13, DG-26 |
| [ADR 0004](../adr/0004-per-channel-directional-keys.md) | One directional four-key schedule per endpoint pair prevents reflection while envelope contexts separate messages and scopes. | R-10, R-14 / DG-17 |
| [ADR 0005](../adr/0005-user-chosen-doxing-password.md) | A user-chosen password is accepted without a protocol entropy threshold; offline-guessing risk is accepted. | R-07 / DG-11, DG-16 |
| [ADR 0006](../adr/0006-per-withdrawal-mystery-generation.md) | A fresh mystery is generated at `DIGGING` entry and erased with active withdrawal state, preventing a reached threshold from carrying into another ceremony. | R-18, R-25 / DG-30, DG-35 |

### Protocol phase coverage

| Phase | Security-critical behavior | Main risks and gaps |
| --- | --- | --- |
| SAR registration | Phone derives SAR-scoped rescue keys from the user password, pays the selected SAR, and synchronizes static and dynamic encrypted data; ordering and rollback policy remain open. | R-06, R-07, R-20, R-26 / DG-11, DG-13, DG-16, DG-40 |
| Boomlet installation | Trusted Iso installs `normal_pubkey`, `doxing_key`, and one selected SAR; Boomlet generates identity and MuSig2-share keys. | R-01, R-19, R-24, R-27 / DG-07, DG-10, DG-34, DG-36 |
| ST enrollment | Two fresh nonce-bound country-grid challenges must resolve to the same five-element consent set. | R-02, R-11, R-12, R-21 / DG-08, DG-09, DG-31 |
| Peer records and setup ID | Signed records are sorted by canonical Boomlet identity bytes; Boomlet and ST independently recompute and verify `setup_instance_id`. | R-10, R-17, R-19 / DG-05, DG-06, DG-17 |
| Agreement and checkpoints | All five Boomlets sign one parameter fingerprint; WT, SAR, and backup phases advance a chained setup checkpoint. | R-05, R-10, R-14, R-17 / DG-05, DG-12, DG-17 |
| WT and SAR finalization | WT is bound to the accepted setup; each peer's chosen SAR and identifier are setup-bound, checked by SAR, WT, and Boomlet. | R-05, R-06, R-13, R-26 / DG-12, DG-13, DG-24, DG-26 |
| Boomletwo backup | `normal_privkey` authorizes one target; the target-bound envelope imports long-lived setup state and returns `BackupDone`. The backup contains no `mystery`; interrupted import and lost-response recovery remain open. | R-18, R-19, R-27 / DG-04, DG-30, DG-36, DG-39 |
| Withdrawal review and approval | Every user independently verifies `tx_id`; `withdrawal_id` binds the initiator approval, and `approved_withdrawal_id` binds the canonical unanimous set. | R-08, R-10, R-15 / DG-21, DG-22, DG-31 |
| Commitment and initial duress | Every commit carries a fresh placeholder; WT waits for the exact setup-bound SAR acknowledgement before distributing commits. | R-12, R-13, R-21, R-22 / DG-23, DG-24, DG-29, DG-32 |
| Digging | Each Boomlet samples one fresh mystery on entry. Counter advancement is evaluated before every valid pong performs bounded `last_seen_block` catch-up, and occurs only when the local chain view advanced and every other peer's ping is within tolerance around the current local Niso height under a safe chain view. Reached peers keep participating. | R-03, R-05, R-09, R-13, R-25 / DG-01, DG-02, DG-24, DG-25, DG-33, DG-35 |
| Hydration and MuSig2 signing | Hydration preserves the transaction and `SIGHASH_DEFAULT`; Boomlet revalidates the ceremony, while Iso verifies the local signing package and BIP327 session. | R-01, R-08, R-14, R-15 / DG-06, DG-22, DG-30 |
| Export, aggregation, and reset | Boomlet exports its fragment and clears active state before Niso or WT confirms durable receipt; WT later aggregates, verifies the committed `tx_id`, and broadcasts. | R-05, R-13, R-14, R-25 / DG-24, DG-30, DG-39 |

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
| **Social engineering, including phishing and fake ceremonies**      | TB-OOB, TB-PHYS-PEER, TB-SAR       | Normal keys, peer set integrity, rescue readiness, privacy | Steals mnemonics; reroutes peer or WT Tor addresses; tricks the operator into fake SAR enrollment or ceremony steps | Multi-channel fingerprint checks; pinned keys; authenticated SAR directory; training; anti-phishing playbooks |
| **Credential theft, including mnemonic or passphrase compromise**   | TB-PHYS-PEER, TB-ISO               | Normal keys                              | Theft or coerced disclosure enables later deterministic theft once normal-key stages open            | Strong passphrases; split backups; secure storage; anti-phishing discipline |
| **Process non-compliance, such as missed rollover or skipped checks** | TB-PHYS-PEER, TB-OOB             | Funds, coercion resistance               | Users miss rollover windows or skip required checks, allowing the protocol to drift into weaker deterministic stages | Mandatory rollover schedule; automated reminders; rehearsals; documented exception handling and sign-off |
| **Operator error, including wrong milestones, outputs, or network** | TB-PHYS-PEER, TB-ISO, TB-ST        | Funds, liveness                          | Setup produces the wrong descriptor or milestones; withdrawal signs the wrong transaction or wrong network | Strict checklists; independent operator-side transaction review on dedicated hardware; testnet rehearsals; sanity checks and invariants in Iso |
| **Travel and commute risk, including surveillance or ambush**       | TB-PHYS-PEER                       | User safety, anonymity                   | Attacker identifies peer movements and targets vulnerable moments                                    | Vary routines; secure transport; avoid role disclosure; compartmentalize identities; personal-security procedures |
| **Site access failures, including unauthorized storage access**     | TB-PHYS-PEER                       | Devices, backups                         | Evil-maid installs implants or swaps devices; steals backups                                         | Access control, alarms, cameras; tamper seals; inventory |
| **Device theft or tampering of ST or Boomlet**                      | TB-PHYS-PEER, TB-BOOMLET, TB-ST    | Duress secrecy, protocol integrity, keys | Attacker tampers with trusted devices, learns consent-set material, or destroys Boomerang-state devices to force fallback | Certified secure element; tamper resistance; secure storage; spare devices; incident response |
| **Environmental disasters, including fire, flood, or EMP**          | TB-PHYS-PEER                       | Devices, backups                         | Damage or destruction forces fallback or makes recovery impossible                                   | Geo-redundant storage; fireproof safes; Faraday protection; disaster recovery drills |
| **Power interruption during signing or backup procedures**          | TB-PHYS-PEER, TB-BOOMLET, TB-ISO   | Signing state, liveness                  | Interrupted sessions cause partial state, nonce reuse, or lost progress                              | Atomic signing sessions; anti-rollback; UPS; recovery paths |
| **Jurisdictional or legal coercion of SAR or operators**            | TB-JURIS, TB-SAR                   | PII, user safety, service availability   | SAR forced to reveal data, delay action, or refuse service; operator compelled disclosure            | Minimize PII; legal counsel; documented policies; transparency reports |

### Human-safety hazards

Financial risk scores do not represent injury or loss of life. Each hazard
below has a catastrophic worst case even when its probability is unknown.

| Hazard ID | Initiating condition | Worst credible outcome | Protocol bearing | Related risks and threats |
| --- | --- | --- | --- | --- |
| H-01 | A coercer observes or infers a duress signal. | Retaliation, escalation, serious injury, or death to the user or household members. | Message indistinguishability reduces protocol leakage but cannot control physical observation or an attacker's reaction. | R-12, R-22 / T-INFO-10, T-HUM-06 |
| H-02 | User error or a manipulated interaction produces a valid non-consent response. | Wrongful intervention, detention, injury, or exposure of an uninvolved user. | Boomlet rejects malformed input, but every other valid five-element set means duress. | R-21 / T-HUM-03, T-HUM-06 |
| H-03 | SAR acts on stale, rolled-back, forged, or misattributed rescue data. | Response to the wrong location or person, with harm to the user, household members, responders, or bystanders. | Encryption authenticates an envelope but does not establish that its dynamic contents are current. | R-06, R-26 / T-DATA-02, T-PHONE-01 |
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

A coercer who records one legitimate safe interaction can learn a response that
remains valid for the setup. The protocol defines no consent-set rotation or
recovery procedure after suspected observation.

SAR acknowledgement establishes receipt of the exact placeholder and durable
activation for a valid duress tuple. It does not establish successful rescue,
correct location, lawful authority, or a non-escalatory outcome.

Primary coercion campaign: [Tree 0](attack_trees.md#tree-0-primary-attacker-campaign-steal-funds-under-coercion-or-force-deterministic-fallback).

### Operational hardening checklist

**People**
- Run regular coercion, recovery, and rollover drills.
- Minimize who knows peer identities and custody roles.

**Facilities**
- Use secure, access-controlled environments for setup, backup, and withdrawal ceremonies.
- Maintain tamper-evident seals and inspection logs for ST, Boomlet, and Boomletwo.
- Keep Boomlet, Boomletwo, and mnemonic backups in geographically separated storage.

**Travel**
- Avoid predictable patterns and role disclosure.
- Use secure transport and avoid carrying all critical devices or backups together.

**Incident response**
- Predefine device-loss, coercion-suspected, and missed-rollover playbooks.
- Maintain a rehearsed rollover or fund-movement plan before deterministic stages.



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

Scores reflect the current specification and only implemented or normative
controls. Proposed roadmap work does not reduce a score.

`Status` indicates whether the present posture depends mainly on controls, assumptions, or unresolved design gaps.

The score ranks custody and protocol risk. It does not reduce the human-safety
hazards in H-01 through H-05 to financial likelihood and impact.

### Risk register table

| Risk ID | Risk | Score | Status | Canonical DG links | Notes |
| --- | --- | --- | --- | --- | --- |
| R-01 | Boomlet secure element compromise: key extraction or logic tamper | 15 | Assumption | DG-06, DG-07 | Assumes JavaCard-class extraction resistance, non-exportable Boomlet key shares, and fallback timelocks. |
| R-02 | ST compromise: tampering, code injection, or key extraction enabling duress bypass or consent-set exfiltration | 12 | Assumption | DG-08, DG-09, DG-31 | Depends on trusted ST firmware, non-extractable ST keys, air-gapped QR exchange, and tamper evidence. |
| R-03 | Forced determinism from device loss, bricking, or deliberate stalling | 15 | Gap | DG-03, DG-04, DG-30 | Waterfall timelocks and backup import exist; rollover policy and Boomletwo activation remain critical. |
| R-04 | Malicious or compromised peer exploits later waterfall stages or governance gaps | 10 | Control | DG-03, DG-14, DG-15 | Early stages require 5-of-5; later recoverability stages need governance and rollover discipline. |
| R-05 | Watchtower compromise or malicious operation | 12 | Gap | DG-01, DG-12, DG-18, DG-34 | Active WT behavior remains central to freshness, routing, coordination, liveness, and metadata exposure. |
| R-06 | SAR compromise, coercion, or rescue-data misuse | 12 | Gap | DG-11, DG-13, DG-16 | SAR sees encrypted rescue data until duress; governance, jurisdiction, and abuse resistance remain operational boundaries. |
| R-07 | Password-based `doxing_key` brute-force after SAR database or ciphertext exposure | 9 | Accepted by ADR 0005 | DG-11, DG-16 | User-chosen `doxing_password` is accepted by ADR 0005; no protocol-layer entropy threshold is enforced. |
| R-08 | Niso compromise tampers PSBT presentation, hydration, or Tor coordination | 16 | Gap | DG-10, DG-22, DG-31 | ST and Boomlet check `tx_id` continuity, while Niso hardening and hydrated-PSBT verification remain important. |
| R-09 | False block-height or eclipse attacks manipulate freshness checks and counters | 12 | Gap | DG-01, DG-25 | Depends on chain-view trust, WT height, Niso RPC, and explicit reorg/freshness policy. |
| R-10 | Replay, spoofing, serialization, or desync of protocol messages | 12 | Control/Gaps | DG-05, DG-17, DG-21, DG-23, DG-24, DG-25, DG-29 | Core binding fields are specified; parser hardening, evidence retention, timeout, and fairness remain boundaries. |
| R-11 | QR-code channel attacks through malformed payloads, parser bugs, swapping, or camera trickery | 12 | Control | DG-10, DG-31, DG-34 | Air-gap, signatures, and nonces help; parser hardening and operational QR controls remain necessary. |
| R-12 | Duress and rescue-signal leakage under observation, side channels, or coercion | 12 | Assumption | DG-15, DG-23, DG-29, DG-32, DG-38 | The SPEC fixes message shape, acknowledgement deadline, durable-write path, retries, failures, and visible telemetry; secrecy still depends on ST, user privacy, and an unobserved interaction. |
| R-13 | WT, SAR, peer availability, or fairness gaps block withdrawal or rescue acknowledgement | 12 | Gap | DG-12, DG-13, DG-14, DG-24, DG-39 | Active flow depends on one WT, one setup-bound SAR identity per peer, and all peers. Retry restrictions are specified; timeout, blame, WT failover, crash recovery, and response to prolonged SAR unavailability are not. |
| R-14 | Implementation bugs in cryptography, signing, or state binding | 15 | Assumption | DG-06, DG-17, DG-21, DG-22, DG-23, DG-29 | Cryptographic profile and binding are specified; implementation evidence and test vectors remain essential. |
| R-15 | Operator and ceremony errors cause loss or forced determinism | 16 | Control | DG-02, DG-03, DG-10, DG-14, DG-15, DG-30 | User verification, signed fingerprints, multiparty agreement, backup, runbooks, and monitoring carry major risk. |
| R-16 | Deanonymization and targeting through Tor correlation, metadata leakage, or routine exposure | 8 | Control | DG-16, DG-18 | Tor and encrypted PSBTs reduce exposure; metadata minimization and payment unlinkability remain open. |
| R-17 | Out-of-band peer exchange compromise causes key substitution or MITM | 15 | Control | DG-05, DG-14, DG-17 | Signed `PeerSetupRecord` values, canonical Boomlet ordering, and independent Boomlet/ST recomputation of `setup_instance_id` still depend on users obtaining the intended records out of band. |
| R-18 | Boomletwo backup misuse bypasses non-determinism or clones state | 12 | Gap | DG-04, DG-30, DG-35, DG-39 | Setup backup contains no `mystery`, preventing threshold cloning; activation, one-active-device semantics, and interrupted-import recovery remain canonical gaps. |
| R-19 | Device substitution during setup or withdrawal | 8 | Control | DG-07, DG-08, DG-09, DG-10 | Identity binding and user verification exist; hardware provenance and tamper evidence remain important. |
| R-20 | Logging and monitoring data leaks from WT, SAR, or Niso | 12 | Assumption | DG-16, DG-18, DG-34 | Logging, retention, and monitoring controls depend mainly on operator practice and service hardening. |
| R-21 | False duress from user mistake triggers SAR escalation | 6 | Control | DG-13, DG-32, DG-38 | Boomlet rejects malformed index sets, but any valid set unequal to the consent set means duress. Observation, stress errors, false-positive handling, and the consequences of intervention remain external. |
| R-22 | False negative duress: delivery, acknowledgement, or response fails | 15 | Gap | DG-13, DG-23, DG-24, DG-29, DG-32, DG-39, DG-40 | Boomlet verifies the exact SAR acknowledgement, and SAR commits new activation before the fixed deadline; crash recovery, data currency, service availability, and physical response remain safety-critical. |
| R-23 | WT service-fee payment metadata links peer identity to custody participation | 9 | Control | DG-12, DG-16, DG-18 | WT invoices and receipts create privacy pressure; payment unlinkability remains an operational control. |
| R-24 | Software or firmware supply-chain compromise or malicious updates | 10 | Assumption | DG-07, DG-08, DG-34 | Build provenance, release signing, update integrity, and dependency control are trusted at this stage. |
| R-25 | Digging-game timing, reorg recovery, or stalling edge cases | 12 | Gap | DG-01, DG-02, DG-24, DG-25, DG-33, DG-35 | Bounded no-progress height catch-up, full current-peer ping sets, monotonic reach state, and continued participation by reached peers close the known stale-lock, acceleration, and final-laggard paths. Adversarial timing, reorg recovery, and recurring-duress selection still need validation. |
| R-26 | SAR enrollment or stale rescue data creates false coverage | 12 | Gap | DG-11, DG-13, DG-26, DG-40 | Setup binds the selected SAR, identifier, static-data fingerprint, and IV, and Iso verifies them during backup. Dynamic-data ordering, expiry, rollback rejection, and effective response remain undefined. |
| R-27 | Iso compromise or substitution during setup installs attacker-chosen authority or rescue state | 15 | Assumption | DG-10, DG-34, DG-36 | Iso supplies `normal_pubkey`, `doxing_key`, and the selected SAR, and later authorizes backup. The SPEC explicitly excludes setup-time Iso compromise from its threat boundary. |
| R-28 | Common-mode failure or a cross-boundary coalition defeats assumed separation | 20 | Gap | DG-12, DG-13, DG-18, DG-34, DG-37 | Five peer identities do not guarantee independent devices, software, provisioning, RPC sources, services, payment trails, or legal exposure. One shared compromise or cooperating set of actors can bypass checks that only address isolated faults. |


## Attacker paths

Attack-tree source: [attack_trees.md](attack_trees.md). Covered campaigns:

1. theft under coercion or forced deterministic fallback;
2. PSBT tampering and broken transaction-intent continuity;
3. deliberate entry into deterministic fallback;
4. suppression or failure of duress-triggered rescue;
5. peer deanonymization followed by targeted coercion;
6. Boomlet, ST, or Iso supply-chain compromise; and
7. common-mode compromise or a cross-boundary coalition.


<a id="mitigations-roadmap"></a>
## Mitigations & roadmap

### Mitigation roadmap

| Stage | Workstream | DG / Risk links | Verification focus | Resolution |
| :--- | :--- | :--- | :--- | :--- |
| Completed | Generate each `mystery` at `DIGGING` entry and erase it with active withdrawal state. | DG-35 / R-25,R-18 | Abort/retry tests, state-lifetime tests, and checks that no setup or backup object contains `mystery`. | Resolved in f74a8f0ce2563d0f2b68c50ff876dfa14f62f398 by ADR 0006 and the per-withdrawal state transition. |
| Now | Validate and implement the specified cryptographic profile, canonical serialization, and test vectors. | DG-06 / R-14,R-10,R-25 | Independent crypto review, interop vectors, strict-decoder fuzzing, and per-schema limit tests. | The primitive and binding profile was fixed in 96734855a35db6af7c6d8be06d72d452433bbc6b; normal-key derivation was fixed in ec72f17fc97b31871e1b6c72f0c08e83a30ce436; canonical schema encoding was completed in 5616ca0efffa7f2f6f73008240a225064e6aaf77; remaining SPEC/diagram mismatches in normal derivation and the WT-to-SAR response were corrected in 4a44d069c7412637a9020951d831d1594d344276. Implementation evidence and complete vectors remain open. |
| Now | Preserve the ST and Boomlet checks that hold transaction intent from approval through broadcast. | DG-22,DG-31 / R-08,R-15 | UX review, adversarial `tx_id` tests, SIGHASH tests, and operator verification tests. | Resolved in 96734855a35db6af7c6d8be06d72d452433bbc6b by withdrawal IDs, PSBT hydration constraints, and signing-package verification; `SIGHASH_DEFAULT` was fixed in 267ae5d5219b85f7a94e763ad073a24e2524838c. |
| Now | Harden Niso as a dedicated appliance. | DG-10,DG-22,DG-34 / R-08,R-20,R-11 | Hardening benchmark, penetration test, and malware simulation. | Open. |
| Now | Harden trusted Iso provisioning and make setup authority independently auditable. | DG-10,DG-34,DG-36 / R-19,R-24,R-27 | Media provenance, device-authentication tests, derived-key and SAR cross-checks, and adversarial provisioning review. | Trusted Iso installation, setup, backup, and signing were made explicit in 2fb53eaa79e0f5dddf7e98b7b4f1edca82f6674e; platform assurance and ceremony controls remain open. |
| Now | Define and rehearse incident response for coercion, device loss, WT/SAR outage, and approaching milestones. | DG-03,DG-12,DG-13,DG-15,DG-24 / R-15,R-03,R-22,R-13 | Tabletop exercises, runbook reviews, and drills. | Open. |
| Now | Formalize rollover policy, milestone monitoring, alerts, reminders, and dashboards. | DG-02,DG-03,DG-15 / R-15,R-03,R-04 | Operational audit, testnet rollover drill, and monitoring tests. | Open. |
| Before production | Build multi-WT architecture with provider diversity, jurisdiction diversity, transparency logs, and HSM-backed WT keys. | DG-12,DG-16,DG-18,DG-34,DG-37 / R-05,R-13,R-16,R-28 | Failover tests, WT security audit, coalition scenarios, and chaos testing. | Open. |
| Before production | Design redundant SAR operations where policy permits, with audited access and uncertain-duress playbooks. | DG-13,DG-23,DG-24,DG-32,DG-37 / R-06,R-22,R-21,R-28 | Fixed-deadline and deadline-overrun tests, durable-path equivalence, end-to-end duress drills, privacy audit, and false-positive exercises. | Exact acknowledgement binding and observable-behavior rules were resolved in 96734855a35db6af7c6d8be06d72d452433bbc6b; the fixed release deadline and identical durable-write path were added in 40e09c3b4c9162d6d9870092e49f3d6241e82e48. Service governance remains open. |
| Before production | Evaluate Boomlet and ST hardware security for side channels, fault injection, supply chain, and tamper evidence. | DG-07,DG-08,DG-09 / R-01,R-02,R-19 | Independent lab reports, teardown reviews, and provenance controls. | Open. |
| Before production | Formalize Boomletwo activation and recovery while preserving the one-active-device invariant. | DG-04,DG-30,DG-39 / R-18,R-03 | Interrupted-import tests, lost-`BackupDone` recovery, adversarial recovery tests, and formal invariants. | Open. |
| Before production | Define crash-consistent SAR acknowledgement and signed-fragment handoffs. | DG-30,DG-39 / R-13,R-14,R-22,R-25 | Crash-point tests, durable receipt, idempotent replay, restart, and exactly-once cleanup checks. | Open. |
| Before production | Define dynamic rescue-data ordering, expiry, rollback rejection, and conflict handling. | DG-11,DG-40 / R-06,R-22,R-26 | Stale, duplicate, reordered, future-dated, and conflicting update tests. | Open. |
| Before production | Define the response to observed consent entry and common-mode compromise. | DG-37,DG-38 / R-12,R-21,R-28 | Continuous-observation exercises, consent-set recovery review, shared-failure scenarios, and coalition analysis. | Open. |
| Before production | Formally verify and simulate the digging-game state machine under adversarial delays, reorgs, equivocation, crashes, and correlated compromise. | DG-01,DG-19,DG-24,DG-25,DG-33,DG-37,DG-39 / R-25,R-09,R-10,R-28 | Reproducible model, bounded and liveness runs, simulation harness, and independent review. | Open. |
| Later | Add advanced privacy protections for traffic shape, payment unlinkability, and anonymity-set management. | DG-16,DG-18 / R-16,R-23,R-20 | Traffic analysis tests, chain analysis, and privacy audit. | Open. |

### Component-specific hardening summary

**Boomlet**
- Secure-element selection and evaluation
- Anti-rollback counters
- Strict parsing
- Signed applets
- Minimal command surface
- Persist transcript commitments to prevent replay or equivocation acceptance

**ST**
- Secure boot
- Signed firmware
- No radios
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
- Key protection and rotation
- Metadata minimization
- Rate limiting
- Onion-service hardening
- Privacy-preserving logging and retention limits
- Before-production failover design and transparency logs

**SAR**
- PII minimization
- Retention limits
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
| DG-01 | Reorg and divergent-chain recovery are incomplete | Boomlet stalls with `CHAIN_VIEW_UNSAFE` on height decrease or material RPC/WT disagreement, but the trusted chain-source policy and recovery procedure are not defined. | Critical | Partly resolved by the explicit unsafe-chain checks in 96734855a35db6af7c6d8be06d72d452433bbc6b; recovery remains open. |
| DG-02 | No parameter-selection framework for milestones, mystery range, freshness windows, and SAR acknowledgement delay | Security depends on suitable timing constants, a bounded SAR pre-acknowledgement path, and WT timeouts that accommodate the fixed release deadline, but no production selection or validation method is defined. | Critical | Open |
| DG-03 | Forced determinism is controlled mostly by user discipline and peer behavior | Late start, device loss, and non-cooperation are collapse paths, and the mitigation depends mainly on timely rollover and careful operation. | Critical | Open |
| DG-05 | Setup uniqueness and anti-replay | `setup_instance_id`, `peer_setup_nonce`, and chained setup checkpoints separate one setup from another. | High | Resolved in 96734855a35db6af7c6d8be06d72d452433bbc6b by `setup_instance_id`, `peer_setup_nonce`, and setup checkpoints. |
| DG-06 | Cryptographic conformance evidence is incomplete | The SPEC fixes AES-256-CBC/PKCS#7, AES-CMAC encrypt-then-MAC, SP 800-108 CMAC KDF, directional channel keys, BIP39/BIP32 normal-key derivation, canonical bytes, schema IDs, strict decoding, and scope binding. Complete wire schemas, byte vectors, per-schema bounds, and implementation evidence are still required. | Critical | Primitive and binding choices were resolved in 96734855a35db6af7c6d8be06d72d452433bbc6b; normal-key derivation was fixed in ec72f17fc97b31871e1b6c72f0c08e83a30ce436; canonical schema encoding was added in 5616ca0efffa7f2f6f73008240a225064e6aaf77; remaining SPEC/diagram mismatches were corrected in 4a44d069c7412637a9020951d831d1594d344276. Conformance evidence remains open. |
| DG-17 | Message binding | Canonical envelope contexts, scope IDs, and directional channel keys bind messages to their channel and purpose. | High | Resolved in 96734855a35db6af7c6d8be06d72d452433bbc6b by canonical envelope contexts, scope IDs, and directional channel keys. |
| DG-21 | Withdrawal-session binding | `withdrawal_id` and `approved_withdrawal_id` keep approvals, commitments, pings, signing readiness, and replay scope on the same withdrawal attempt. | Critical | Resolved in 96734855a35db6af7c6d8be06d72d452433bbc6b by `withdrawal_id` and `approved_withdrawal_id`. |
| DG-22 | Transaction-authorization binding | PSBT hydration constraints, reached-collection checks, signing-package verification, `SIGHASH_DEFAULT`, and final `tx_id` equality preserve the approved transaction. | High | Resolved in 96734855a35db6af7c6d8be06d72d452433bbc6b by PSBT hydration constraints and signing-package verification; strengthened in 267ae5d5219b85f7a94e763ad073a24e2524838c by the fixed sighash policy. |
| DG-23 | SAR acknowledgement binding | SAR acknowledges the exact placeholder, with defined retry and failure behavior and replay memory. | High | Resolved in 96734855a35db6af7c6d8be06d72d452433bbc6b by exact placeholder acknowledgements, retry/failure handling, and replay memory. |
| DG-24 | Honest-path liveness lacks interoperable timeout and fairness policy | Failure classes, retry restrictions, stalling, and explicit abandonment are defined, but peer/service progress, timeout schedules, blame, WT failover, and response to prolonged SAR unavailability are not. The setup-bound SAR cannot be replaced within the existing setup. | High | Partly resolved in 96734855a35db6af7c6d8be06d72d452433bbc6b; timeout and fairness policy remain open. |
| DG-25 | Freshness recovery and evidence retention are incomplete | Authenticated lagging pings remain valid for pong production, and every valid pong performs bounded `last_seen_block` catch-up without granting counter progress. Unsafe chain views still stall, and durable evidence retention or recovery after that stall is undefined. | High | Digging progression and lag catch-up are specified; chain-view recovery and audit retention remain open. |
| DG-28 | Mid-ceremony interaction between boomerang progress and fallback opening is under-specified | The design does not define one canonical operator procedure for overlap between boomerang progress and deterministic fallback availability. | High | Open |
| DG-29 | Placeholder-instance lifecycle | Every placeholder has a fresh IV, an `approved_withdrawal_id`, and a SAR replay tuple. | High | Resolved in 96734855a35db6af7c6d8be06d72d452433bbc6b by per-instance IVs, `approved_withdrawal_id` binding, and SAR replay tuples. |
| DG-30 | Post-withdrawal cleanup and reset semantics are partly specified | Boomlet clears active state immediately after exporting its signed fragment, before durable Niso or WT receipt is established. Recovery from interruption between export, relay, aggregation, broadcast, and cleanup is not defined. | High | Mystery cleanup was resolved in f74a8f0ce2563d0f2b68c50ff876dfa14f62f398; durable handoff and cross-component recovery remain open. |
| DG-31 | ST prompt and display conformance are incomplete | The SPEC defines nonce-bound transaction review, the exact duress challenge and response semantics, and the 193-label vocabulary. Exact prompt encoding, display-grid conformance, and independent `tx_id` tooling remain implementation boundaries. | High | Partly resolved in 96734855a35db6af7c6d8be06d72d452433bbc6b by nonce-bound semantics; prompt and display conformance remain open. |
| DG-32 | End-to-end duress indistinguishability timing and error discipline | Valid safe and duress placeholders use the same message shape, fixed acknowledgement release deadline, durable-write path, retry policy, and externally visible failure behavior. | Critical | Resolved in 96734855a35db6af7c6d8be06d72d452433bbc6b; strengthened in 40e09c3b4c9162d6d9870092e49f3d6241e82e48 by the fixed deadline and identical durable-write path. |
| DG-33 | Recurring-duress random trigger semantics are not protocol-canonical | The SPEC requires an unbiased draw for the configured interval but does not define the draw domain, selection predicate, PRNG state lifetime, or cross-peer independence requirements. | High | Open |
| DG-35 | Mystery lifetime across withdrawal attempts | A threshold learned from `reached_mystery_flag` must not survive abort, reset, backup, or a later ceremony. | Critical | Resolved in f74a8f0ce2563d0f2b68c50ff876dfa14f62f398 by generating `mystery` only on entry to `DIGGING` and erasing it with active withdrawal state. |
| DG-38 | Consent-set compromise has no rotation or recovery path | A coercer who observes a legitimate safe interaction can reuse the learned response for the lifetime of the setup. The protocol does not define replacement after suspected observation. | Critical | Open |
| DG-39 | Crash-consistent protocol handoffs are incomplete | Signed-fragment export, one-time backup import, and fixed-deadline SAR acknowledgement each cross a durable-state boundary. The protocol does not define recovery for every crash point, lost response, or restarted deadline. | Critical | Open |

### Boomlet, ST, and endpoint platform assurance

| Gap ID | Design gap | Why it matters | Priority | Status / Resolution |
| --- | --- | --- | --- | --- |
| DG-04 | Boomletwo activation, anti-clone, and interrupted-import semantics are unspecified | The SPEC defines authenticated, target-bound import of long-lived setup state. No mystery exists during setup backup. Safe deactivation, activation, revocation, duplicate-authority prevention, and recovery after import succeeds but `BackupDone` is lost remain undefined. | Critical | Open |
| DG-07 | Boomlet assurance is assumed, not demonstrated | The central trust anchor is expected to resist extraction, side channels, bias, cloning, and endurance failures, but attestation, audit, and hardware-evaluation evidence are absent. | Critical | Open |
| DG-08 | The ST trust model is stronger than the current prototype evidence | The ST is treated like a single-purpose trusted appliance, while the referenced Portenta and breakout materials emphasize prototyping, boot modes, debug access, exposed signals, and flexible I/O. | Critical | Open |
| DG-09 | Prototype attack surface is not converted into a hardened ST build profile | The design does not define which interfaces are removed, fused off, sealed, or monitored in the ST build despite the prototype exposing multiple attack-friendly interfaces. | High | Open |
| DG-10 | Endpoint transition and replacement procedures are missing | The design requires moving hardware between hosts and anticipates replacing Phone, Niso, and ST, but those procedures are ancillary only and not defined. | High | Open |
| DG-36 | Trusted Iso setup and provisioning boundary lacks assurance | Iso installs `normal_pubkey`, `doxing_key`, and the selected SAR, handles setup and backup secrets, and authorizes backup with `normal_privkey`. A substituted Iso can establish attacker-chosen long-lived authority before later checks begin. | Critical | The exclusion was made explicit in 2fb53eaa79e0f5dddf7e98b7b4f1edca82f6674e; platform assurance and ceremony controls remain open. |

### Services, operators, and recovery

| Gap ID | Design gap | Why it matters | Priority | Status / Resolution |
| --- | --- | --- | --- | --- |
| DG-34 | Software supply chain and service implementation security remain under-specified | A compromised build, update, WT, or SAR can bypass protocol checks. Build provenance, update authorization, deployment hardening, patching, and secure service operation are not defined. | High | Open |
| DG-11 | Dynamic doxing data integrity and confidentiality are largely delegated to Phone, password choice, and SAR | Rescue quality depends on Phone data staying correct and current; confidentiality after ciphertext exposure depends on the user-chosen `doxing_password`. Fresh authenticated envelopes do not prevent an old valid update from replacing a newer one. | High | Open |
| DG-12 | WT is a critical coordination dependency without a finalized failover or accountability model | WT is non-custodial but central to liveness, freshness, routing, and metadata exposure, and switching or redundancy procedures remain deferred. | Critical | Open |
| DG-13 | SAR conduct, effectiveness, and jurisdiction remain external assumptions | Protocol acknowledgement proves delivery and durable activation, not lawful, timely, correctly directed, effective, or non-escalatory intervention. SAR misuse, compulsion, insider abuse, and harmful response remain possible. | Critical | Open |
| DG-14 | Peer governance is under-specified for blame, timeout, and non-cooperation | N-of-N is intentional, but the design lacks final procedures for peer delay, blame, expulsion, recovery from silence, or coordination breakdown. | High | Open |
| DG-15 | Human availability is not reconciled with random duress checks and freshness limits | The design wants unpredictable checks, but users sleep, travel, live in different time zones, and can be delayed under ordinary conditions. | High | Open |
| DG-19 | The protocol lacks formal or simulation-based validation of timing behavior | No formal model or simulation evidence establishes timing and liveness under adversarial delay, reorgs, crashes, or equivocation. | High | Open |
| DG-20 | Security-critical properties depend on future ancillaries or operator discipline | The design explains the cooperative path, but many real-world safety properties live outside the protocol core. | High | Open |
| DG-26 | Single-SAR-per-peer contract | Each peer selects one SAR identity, and SAR routing is bound during setup. | High | Resolved in 96734855a35db6af7c6d8be06d72d452433bbc6b by ADR 0003 and setup-bound SAR routing. |
| DG-37 | Common-mode failures and cross-boundary coalitions are not modeled | N-of-N resists independent peer compromise, but shared devices, software, provisioning, RPC sources, services, payment trails, or legal authority can correlate failures. The protocol states no independence requirement or coalition policy. | Critical | Open |
| DG-40 | Dynamic rescue-data freshness and rollback semantics are undefined | `DynamicDoxingData.captured_at` records a source timestamp, but SAR has no canonical ordering, expiry, clock-skew, rollback-rejection, or conflict rule. | Critical | Open |

### Privacy and network exposure

| Gap ID | Design gap | Why it matters | Priority | Status / Resolution |
| --- | --- | --- | --- | --- |
| DG-16 | Privacy leakage is not governed by a minimization strategy | The protocol exposes identifiers, timing, payments, peer relationships, chain queries, transcripts, and password-bounded SAR ciphertext, but does not define which disclosures are acceptable, avoidable, or prohibited. | High | Open |
| DG-18 | The network trust model is incomplete | Tor is assumed for privacy and availability, direct RPC is assumed safe enough, and signed onion addresses are assumed sufficient, but correlation, censorship, and routing edge cases are not fully modeled. | Medium | Open |
