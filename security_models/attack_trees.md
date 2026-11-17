# Boomerang attack trees

> **Last change — 2026-11-16:** updated attack paths for rescue data and optional backup; added consent-maintenance abuse coverage.

Boomletwo branches apply only where a peer chose backup. Without one, losing
the active Boomlet can block the primary path.

### Tree 0: Primary attacker campaign: steal funds under coercion or force deterministic fallback

Green nodes are rules enforced by the protocol. Yellow nodes are assumptions,
external dependencies, or unresolved gaps.

```mermaid
flowchart TD
  A["Goal: Steal funds under coercion or by forcing deterministic fallback"] --> OR0{OR}
  OR0 --> B["Force spend in Boomerang regime"]
  OR0 --> C["Force spend in normal regime at an attainable threshold"]

  B --> AND1((AND))
  AND1 --> B0["Reach milestone_block_0 (or wait until it is reached)"]
  AND1 --> B1["Control all N required peers"]
  AND1 --> B2["Obtain all required signing material and ceremony/device access"]
  AND1 --> B3["Sustain coercive control until all mystery thresholds are reached"]

  B1 --> AND2((AND))
  AND2 --> B1a["Deanonymize and locate peer set"]
  AND2 --> B1b["Physically capture or control each required peer"]
  AND2 --> B1c["Prevent refusal, escape, or loss of control over any one peer"]

  B2 --> AND3((AND))
  AND3 --> B2a["Force access to Boomlet, ST, and operator workflow"]
  AND3 --> B2b["Force disclosure or use of mnemonic and passphrase for each required peer"]
  AND3 --> B2c["Compel repeated approvals and participation through the ceremony"]

  B3 --> AND4((AND))
  AND4 --> B3a["Sustain detention and logistics through any remaining digging delay"]
  AND4 --> B3b["Prevent abort from timeout, ops failure, or peer loss"]

  C --> AND5((AND))
  AND5 --> C1["Control K normal keys at spend time"]
  AND5 --> C2["Reach a milestone where normal-regime threshold ≤ K"]
  AND5 --> C3["Ensure defenders do not successfully exit earlier while threshold > K"]

  C1 --> AND6((AND))
  AND6 --> C1a["Coerce K normal-key holders or mnemonic custodians"]
  AND6 --> C1b["Retain extracted key material until spend"]

  C3 --> OR1{OR}
  OR1 --> C3a["Begin coercion only after the threshold has already degraded to K"]
  OR1 --> C3b["Start earlier and block earlier defender exit paths until threshold degrades"]

  C3b --> AND7((AND))
  AND7 --> C3b1["Prevent timely rollover before the milestone"]
  AND7 --> C3b2["Prevent earlier normal-regime withdrawal while threshold > K"]
  AND7 --> C3b3["If needed, prevent Boomerang-regime completion"]

  C3b3 --> OR2{OR}
  OR2 --> C3b3a["Destroy or steal a required Boomlet"]
  OR2 --> C3b3b["Induce or exploit peer non-cooperation"]
  OR2 --> C3b3c["Rely on prior operational failure or late withdrawal start"]

  D1["Dependency: peer anonymity and OPSEC"] -.-> B1a
  D2["Dependency: geographic and temporal dispersion"] -.-> B1b
  D3["Protocol rule: N-of-N Boomerang authorization"] -.-> B1
  D4["Protocol rule: separate device roles and signed ceremony checks"] -.-> B2a
  D5["Dependency: secrecy and custody of mnemonics and passphrases"] -.-> B2b
  D6["Protocol rule: fresh per-withdrawal mystery generated once at DIGGING entry"] -.-> B3a
  D7["Protocol rule: recurring duress checks and exact fixed-deadline SAR acknowledgement"] -.-> B3
  D8["Dependency: normal-key custody"] -.-> C1
  D9["Dependency: rollover before milestones"] -.-> C3b1
  D10["Boundary: activating Boomletwo after Boomlet loss remains unspecified"] -.-> C3b3a
  D11["Dependency: SAR intervention is effective, lawful, correctly directed, and non-escalatory"] -.-> B3
  D12["Boundary: counters may have advanced or reached mystery before coercion of the same approved transaction"] -.-> B3a

  classDef goal fill:#ffffff,stroke:#222,stroke-width:2px,color:#000;
  classDef attack fill:#ffcccc,stroke:#cc0000,color:#000;
  classDef control fill:#ccffcc,stroke:#009900,color:#000;
  classDef open fill:#fff2cc,stroke:#b38f00,color:#000;
  classDef gate fill:#ffffff,stroke:#333,stroke-width:2px,color:#000;

  class A goal;
  class B,C,B0,B1,B2,B3,B1a,B1b,B1c,B2a,B2b,B2c,B3a,B3b,C1,C2,C3,C1a,C1b,C3a,C3b,C3b1,C3b2,C3b3,C3b3a,C3b3b,C3b3c attack;
  class D3,D4,D6,D7 control;
  class D1,D2,D5,D8,D9,D10,D11,D12 open;
  class OR0,OR1,OR2,AND1,AND2,AND3,AND4,AND5,AND6,AND7 gate;

```

### Tree 1: Steal funds by tampering PSBT or breaking intent continuity

```mermaid
flowchart TD
  A["Goal: Steal funds by tampering the transaction or breaking intent continuity"] --> OR0{OR}

  OR0 --> B["Get a malicious PSBT approved by all required peers"]
  OR0 --> C["Cause a different PSBT or transaction to be signed than the one approved"]
  OR0 --> D["Exploit signing, parser, or state-machine implementation bugs"]

  B --> AND1((AND))
  AND1 --> B1["Modify or substitute the PSBT before peer approval"]
  AND1 --> B2["Defeat or mislead operator verification at every required peer"]
  AND1 --> B3["Keep the malicious PSBT and tx_id presentation consistent across all approval synchronization steps"]

  B1 --> OR1{OR}
  OR1 --> B1a["Malware alters outputs, change, or fees before first verification"]
  OR1 --> B1b["Swap the PSBT or per-peer encrypted PSBT payload before recipient peer decrypts it"]

  B2 --> OR2{OR}
  OR2 --> B2a["Compromise initiator watch-only wallet or Niso view so the malicious PSBT appears intended"]
  OR2 --> B2b["Compromise non-initiator Niso verification views so the same malicious PSBT is accepted"]
  OR2 --> B2c["Exploit operator-review weakness so tx_id approval is given for a malicious but consistently presented PSBT"]

  C --> AND2((AND))
  AND2 --> C1["Defeat tx_id and continuity protections across the withdrawal state machine"]
  AND2 --> C2["Make the final signed PSBT or transaction differ from the operator-approved one"]

  C1 --> OR3{OR}
  OR3 --> C1a["Exploit inconsistent PSBT parsing, tx_id derivation, or serialization across components"]
  OR3 --> C1b["Exploit replay or state-confusion bug despite nonce, freshness, and sequence checks"]
  OR3 --> C1c["Exploit approval-to-commit, commit-to-ping, or reached-state binding flaw"]

  C2 --> OR4{OR}
  OR4 --> C2a["Exploit PSBT hydration mismatch at finalization"]
  OR4 --> C2b["Exploit message substitution between reached-state verification and Iso/Boomlet signing"]

  D --> OR5{OR}
  OR5 --> D1["PSBT parser or serializer bug causes signing of a different transaction"]
  OR5 --> D2["MuSig2 implementation bug: nonce, session, or transcript binding failure"]
  OR5 --> D3["State-machine bug skips or misapplies prerequisite approval or reached-state checks"]

  F1["Control: independent PSBT verification on operator tooling and ST tx_id approval at each peer"] -.-> B2
  F2["Control: tx_id-bound withdrawal_id, unanimous approved_withdrawal_id, and reached-state checks"] -.-> C1
  F3["Control: nonces, recency, and sequence-number checks to block replay and stale-state reuse"] -.-> C1b
  F4["Control: hydration allow-list, SIGHASH_DEFAULT, Boomlet revalidation, Iso signing-package checks, and final tx_id equality"] -.-> C2
  F5["Assumption: correct PSBT and MuSig2 implementation on isolated Iso"] -.-> D1
  F6["Control: BIP327 session binding and fresh MuSig2 nonces"] -.-> D2
  F7["Assumption: Boomlet enforces the specified signing prerequisites"] -.-> D3

  classDef goal fill:#ffffff,stroke:#222,stroke-width:2px,color:#000;
  classDef attack fill:#ffcccc,stroke:#cc0000,color:#000;
  classDef control fill:#ccffcc,stroke:#009900,color:#000;
  classDef open fill:#fff2cc,stroke:#b38f00,color:#000;
  classDef gate fill:#ffffff,stroke:#333,stroke-width:2px,color:#000;

  class A goal;
  class B,C,D,B1,B2,B3,B1a,B1b,B2a,B2b,B2c,C1,C2,C1a,C1b,C1c,C2a,C2b,D1,D2,D3 attack;
  class F1,F2,F3,F4,F6 control;
  class F5,F7 open;
  class OR0,OR1,OR2,OR3,OR4,OR5,AND1,AND2 gate;

```


### Tree 2: Reach deterministic fallback and then steal with normal keys

```mermaid
flowchart TD
  A["Goal: Steal funds via the deterministic fallback regime"] --> AND0((AND))

  AND0 --> B["Control K normal keys at spend time"]
  AND0 --> C["Reach a milestone where the active normal-regime threshold ≤ K"]
  AND0 --> D["Ensure defenders do not successfully exit earlier while threshold > K"]

  B --> OR1{OR}
  OR1 --> B1["Phish or steal mnemonic and passphrase backups"]
  OR1 --> B2["Compel disclosure under coercion"]
  OR1 --> B3["Insider or compromised peer retains their own normal key and waits"]

  D --> OR2{OR}
  OR2 --> D1["Begin the attack only after the threshold has already degraded to K"]
  OR2 --> D2["Start earlier and block earlier defender exit until the threshold degrades"]

  D2 --> AND1((AND))
  AND1 --> D2a["Prevent timely rollover before the degrading milestone"]
  AND1 --> D2b["Prevent earlier normal-regime withdrawal while threshold > K"]
  AND1 --> D2c["If needed, prevent Boomerang-regime completion"]

  D2c --> OR3{OR}
  OR3 --> D2c1["Destroy or steal a required Boomlet"]
  OR3 --> D2c2["Induce or exploit peer non-cooperation"]
  OR3 --> D2c3["Exploit coordination failure or dependency outage to delay the ceremony"]
  OR3 --> D2c4["Exploit Boomlet or backup bug to brick or erase required state"]
  OR3 --> D2c5["Exploit prior operational failure or late withdrawal start"]

  E1["Dependency: mnemonic and passphrase custody"] -.-> B1
  E2["Dependency: peer anonymity and geographic dispersion"] -.-> B2
  E3["Dependency: insider-risk governance"] -.-> B3
  E4["Dependency: rollover before deterministic milestones"] -.-> D2a
  E5["Boundary: threshold schedule must keep low-threshold stages remote"] -.-> C
  E6["Dependency: secure and tamper-evident Boomlet custody"] -.-> D2c1
  E7["Boundary: activating Boomletwo after Boomlet loss remains unspecified"] -.-> D2c1
  E8["Boundary: WT redundancy and failover remain unresolved"] -.-> D2c3

  classDef goal fill:#ffffff,stroke:#222,stroke-width:2px,color:#000;
  classDef attack fill:#ffcccc,stroke:#cc0000,color:#000;
  classDef control fill:#ccffcc,stroke:#009900,color:#000;
  classDef open fill:#fff2cc,stroke:#b38f00,color:#000;
  classDef gate fill:#ffffff,stroke:#333,stroke-width:2px,color:#000;

  class A goal;
  class B,C,D,B1,B2,B3,D1,D2,D2a,D2b,D2c,D2c1,D2c2,D2c3,D2c4,D2c5 attack;
  class E1,E2,E3,E4,E5,E6,E7,E8 open;
  class AND0,AND1,OR1,OR2,OR3 gate;

```

### Tree 3: Suppress or abuse duress signaling and rescue

```mermaid
flowchart TD
  A["Goal: Suppress or abuse duress signaling and rescue"] --> OR0{OR}

  OR0 --> B["Cause duress checks to evaluate as safe"]
  OR0 --> C["Prevent a true duress signal from producing actionable SAR response"]
  OR0 --> D["Infer hidden duress and react before rescue can disrupt the attack"]
  OR0 --> F["Force unwanted consent changes or duress"]

  B --> OR1{OR}
  OR1 --> B1["Observe the consent pattern or duress responses in a non-private environment"]
  OR1 --> B2["Compromise ST or exploit a validation or parsing flaw in host-relayed input"]
  OR1 --> B3["Compromise Boomlet so it reveals the consent pattern or mis-evaluates duress"]

  B1 --> OR4{OR}
  OR4 --> B1a["Record the ST display and the user's hand or input device"]
  OR4 --> B1b["Have an attacker operate the input while compelling the user to identify the safe choices"]
  OR4 --> B1c["Reuse an observed consent set until it is privately replaced with a different set"]

  C --> OR2{OR}
  OR2 --> C1["Compromise SAR infrastructure or operators"]
  OR2 --> C2["Sabotage prior SAR registration so activation is not actionable"]
  OR2 --> C3["Falsify or stop Phone uploads, hide or lose SAR history, or misinterpret accepted data"]
  OR2 --> C4["Block a later approved withdrawal so duress recorded during IDLE rotation remains undelivered"]
  OR2 --> C5["Exploit faulty rotation or early cleanup to erase recorded duress"]
  OR2 --> C6["Censor rotation traffic before its duress placeholder reaches SAR"]

  D --> AND1((AND))
  AND1 --> D1["Infer hidden duress from side channels despite intended unchanged protocol flow"]
  AND1 --> D2["React before rescue meaningfully disrupts the withdrawal"]

  D2 --> OR3{OR}
  OR3 --> D2a["Escalate coercion or violence to suppress further signaling"]
  OR3 --> D2b["Complete the remaining attacker-controlled steps before intervention lands"]
  OR3 --> D2c["Relocate or isolate the victim before intervention lands"]

  F --> OR5{OR}
  OR5 --> F1["Use captured Boomlet and ST to approve rotation and choose a replacement"]
  OR5 --> F2["Coerce valid wrong current-set input or attacker-chosen selection and confirmation"]

  E1["Dependency: the physical interaction is not observed"] -.-> B1
  E2["Dependency: ST faithfully preserves input and local confirmation"] -.-> B2
  E3["Assumption: Boomlet resists extraction and duress-state compromise"] -.-> B3
  E4["Assumption: SAR protects its keys and handles alarms correctly"] -.-> C1
  E5["Control: authenticated SAR destination, payment binding and atomic registration receipt"] -.-> C2
  E6["Assumption: Phone supplies relevant data and SAR exposes retained history"] -.-> C3
  E7["Control: safe and duress equivalence through withdrawal, rotation, recovery and reset"] -.-> D1
  E8["Control: ST-approved rotation binds nonce, counter and exact receipts"] -.-> B1c
  E9["Boundary: private input and available Boomlet, ST and any existing backup"] -.-> E8
  E10["Boundary: ST approval does not establish ownership or freedom from coercion"] -.-> F
  E11["Control: sticky duress, protected rotation state and receipt-gated clearing"] -.-> C5
  E12["Boundary: IDLE rotation defers SAR delivery until a later approved withdrawal"] -.-> C4

  classDef goal fill:#ffffff,stroke:#222,stroke-width:2px,color:#000;
  classDef attack fill:#ffcccc,stroke:#cc0000,color:#000;
  classDef control fill:#ccffcc,stroke:#009900,color:#000;
  classDef open fill:#fff2cc,stroke:#b38f00,color:#000;
  classDef gate fill:#ffffff,stroke:#333,stroke-width:2px,color:#000;

  class A goal;
  class B,C,D,F,B1,B1a,B1b,B1c,B2,B3,C1,C2,C3,C4,C5,C6,D1,D2,D2a,D2b,D2c,F1,F2 attack;
  class E5,E7,E8,E11 control;
  class E1,E2,E3,E4,E6,E9,E10,E12 open;
  class OR0,OR1,OR2,OR3,OR4,OR5,AND1 gate;

```

### Tree 4: Deanonymize peers and target them with coercion

```mermaid
flowchart TD
  A["Goal: Identify at least one peer operator to target physically"] --> OR0{OR}

  OR0 --> B["Exploit network and communication metadata"]
  OR0 --> C["Exploit WT metadata or service-provider records"]
  OR0 --> D["Exploit SAR registration, upload or payment records"]
  OR0 --> E["Exploit out-of-band peer-data exchange or operator OPSEC failure"]
  OR0 --> F["Exploit insider leakage"]

  B --> OR1{OR}
  OR1 --> B1["Tor traffic or endpoint correlation against peer-to-peer or peer-to-WT communications"]
  OR1 --> B2["Compromise Niso, host, or local network environment to reveal peer communication metadata"]

  C --> OR2{OR}
  OR2 --> C1["WT logs or retained metadata leak, subpoena, or compromise"]
  OR2 --> C2["WT registration or coordination records reveal peer IDs, params, or communication relationships"]

  D --> OR3{OR}
  OR3 --> D1["SAR payment invoice or receipt records leak, subpoena, or compromise"]
  OR3 --> D2["SAR registration or upload metadata leak, subpoena, or compromise"]
  OR3 --> D3["SAR learns identity on duress, then turns rogue or later leaks that knowledge"]

  E --> OR4{OR}
  OR4 --> E1["Intercept or compromise out-of-band sharing of peer IDs and signed Tor addresses"]
  OR4 --> E2["Operator reuses identifiable channels, accounts, or devices during peer coordination"]
  OR4 --> E3["Compromise a peer device that stores peer address collections or signed peer data"]

  F --> OR5{OR}
  OR5 --> F1["Malicious peer reveals peer contacts or identities"]
  OR5 --> F2["WT insider sells or discloses metadata"]
  OR5 --> F3["SAR insider sells or discloses registration or rescue data"]

  G1["Dependency: Tor hygiene, endpoint hardening, and communication-metadata minimization"] -.-> B
  G2["Boundary: WT log minimization, retention limits, and encryption"] -.-> C1
  G3["Boundary: minimize WT-visible metadata and compartmentalize identifiers"] -.-> C2
  G4["Boundary: privacy-preserving SAR payment and record minimization"] -.-> D1
  G5["Boundary: minimize SAR registration and upload metadata and compartmentalize identifiers"] -.-> D2
  G6["Dependency: secure out-of-band exchange discipline and operator OPSEC"] -.-> E
  G7["Dependency: compartmentalized peer knowledge and governance"] -.-> F1

  classDef goal fill:#ffffff,stroke:#222,stroke-width:2px,color:#000;
  classDef attack fill:#ffcccc,stroke:#cc0000,color:#000;
  classDef control fill:#ccffcc,stroke:#009900,color:#000;
  classDef open fill:#fff2cc,stroke:#b38f00,color:#000;
  classDef gate fill:#ffffff,stroke:#333,stroke-width:2px,color:#000;

  class A goal;
  class B,C,D,E,F,B1,B2,C1,C2,D1,D2,D3,E1,E2,E3,F1,F2,F3 attack;
  class G1,G2,G3,G4,G5,G6,G7 open;
  class OR0,OR1,OR2,OR3,OR4,OR5 gate;

```

### Tree 5: Supply-chain compromise of Boomlet and ST

```mermaid
flowchart TD
  A["Goal: Implant a backdoor into signing or duress hardware or its provisioning path"] --> OR0{OR}

  OR0 --> B["Compromise Boomlet or Boomletwo supply chain"]
  OR0 --> C["Compromise ST supply chain"]
  OR0 --> D["Compromise provisioning artifacts or installation environment"]

  B --> OR1{OR}
  OR1 --> B1["Malicious or substituted secure-element or JavaCard platform"]
  OR1 --> B2["Malicious Boomlet applet installed before or during setup"]
  OR1 --> B3["Malicious Boomletwo backup applet installed before or during setup"]

  C --> OR2{OR}
  OR2 --> C1["Backdoored ST firmware or software"]
  OR2 --> C2["Malicious ST hardware with hidden capture or exfiltration capability"]

  D --> OR3{OR}
  OR3 --> D1["Compromise build artifacts so Iso installs malicious Boomlet, Boomletwo, or ST code"]
  OR3 --> D2["Compromise or substitute Iso during installation"]
  OR3 --> D3["Use compromised Iso to install attacker-chosen normal_pubkey, doxing_key, SAR, or backup target"]

  E1["Boundary: vetted sourcing and hardware evaluation for secure elements"] -.-> B1
  E2["Dependency: applet verification and controlled provisioning"] -.-> B2
  E3["Assumption: backup installation preserves target identity, imported secrets and inactive role"] -.-> B3
  E4["Dependency: tamper-evident ST design and independent inspection"] -.-> C
  E5["Boundary: reproducible artifacts, review, and provenance checks"] -.-> D1
  E6["Assumption: Iso is trusted and isolated throughout setup and backup"] -.-> D2
  E7["Assumption: intended hardware and trusted installation media are used"] -.-> D3

  classDef goal fill:#ffffff,stroke:#222,stroke-width:2px,color:#000;
  classDef attack fill:#ffcccc,stroke:#cc0000,color:#000;
  classDef control fill:#ccffcc,stroke:#009900,color:#000;
  classDef open fill:#fff2cc,stroke:#b38f00,color:#000;
  classDef gate fill:#ffffff,stroke:#333,stroke-width:2px,color:#000;

  class A goal;
  class B,C,D,B1,B2,B3,C1,C2,D1,D2,D3 attack;
  class E1,E2,E3,E4,E5,E6,E7 open;
  class OR0,OR1,OR2,OR3 gate;

```

### Tree 6: Defeat separation through a common cause or coalition

```mermaid
flowchart TD
  A["Goal: Defeat controls that assume independent peers, devices, or services"] --> OR0{OR}

  OR0 --> B["Compromise a dependency shared by several or all peers"]
  OR0 --> C["Combine WT control with a compromised RPC or chain source"]
  OR0 --> D["Combine SAR control with Phone compromise or stale rescue data"]
  OR0 --> E["Combine captured-peer knowledge with WT metadata"]
  OR0 --> F["Correlate or compel WT, SAR, and payment records through shared legal authority"]

  B --> OR1{OR}
  OR1 --> B1["Shared Boomlet applet or secure-element flaw"]
  OR1 --> B2["Shared ST firmware or provisioning compromise"]
  OR1 --> B3["Shared Iso image or setup substitution"]

  C --> AND1((AND))
  AND1 --> C1["WT censors, delays, or selects protocol traffic"]
  AND1 --> C2["RPC source supplies a matching false or stale chain view"]

  D --> OR2{OR}
  OR2 --> D1["Phone supplies attacker-selected rescue data that SAR accepts"]
  OR2 --> D2["SAR and Phone suppress coverage while preserving valid-looking setup state"]

  P1["Protocol rule: signed objects and scope IDs bind content and ceremony"] -.-> C1
  P2["Protocol rule: chain-view disagreement stalls with CHAIN_VIEW_UNSAFE"] -.-> C
  P3["Assumption: peers use sufficiently independent devices and services"] -.-> B
  P5["Boundary: authenticated append does not prove truthful payloads or complete history disclosure"] -.-> D
  P6["Dependency: service, payment, metadata, and legal compartmentalization are external"] -.-> E
  P6 -.-> F

  classDef goal fill:#ffffff,stroke:#222,stroke-width:2px,color:#000;
  classDef attack fill:#ffcccc,stroke:#cc0000,color:#000;
  classDef control fill:#ccffcc,stroke:#009900,color:#000;
  classDef open fill:#fff2cc,stroke:#b38f00,color:#000;
  classDef gate fill:#ffffff,stroke:#333,stroke-width:2px,color:#000;

  class A goal;
  class B,C,D,E,F,B1,B2,B3,C1,C2,D1,D2 attack;
  class P1,P2 control;
  class P3,P5,P6 open;
  class OR0,OR1,OR2,AND1 gate;
```
