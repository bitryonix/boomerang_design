# Boomerang: Bitcoin Cold Storage With Built-in Coercion Resistance

>#### TL;DR  - What is Boomerang?
>
>Boomerang is a high-security Bitcoin cold-storage system designed to protect funds not just from hackers, but from real-world coercion.
>
>The threat it addresses is simple: attackers don’t always break encryption; sometimes they pressure people into authorizing withdrawals. Most wallets can’t tell the difference; a forced signature still works.
>
>Boomerang redesigns how withdrawals happen. Instead of a single signing action, moving funds requires a structured, multi-step process across independent devices and timed checks. As part of that process, users explicitly complete a duress challenge. If they indicate distress, the system recognizes coercion and triggers protective responses, while maintaining a controlled withdrawal flow.
>
><em><u>In short:</u></em>
>
><strong> Boomerang turns bitcoin custody into a coordinated safety protocol that resists forced authorization; protecting against human threats, not just technical ones.</strong>
>
>For high-value holders and organizations, this fills a major gap in existing custody solutions, creating a new class of security focused on operational resilience as much as cryptography.

**Boomerang** is a Bitcoin cold-storage protocol that introduces strong coercion resistance without requiring any changes to Bitcoin consensus. It uses a deliberately non-deterministic withdrawal ceremony enforced by secure hardware to create an unpredictable signing process with embedded, plausibly deniable duress signaling and search-and-rescue escalation.

The key idea is that **neither the user nor an attacker can exactly or within an operationally significant range, know in advance how long withdrawal will take**, and that duress signaling is both unavoidable (checks recur) and plausibly deniable.

**Boomerang** is suitable for enterprises and individuals with large sums of bitcoin in treasury mode or as a secure fallback for deleted key covenant structures like [Ajolote](https://arxiv.org/abs/2310.11911) or [Revault](https://wizardsardine.com/revault/).

Apart from its performance in case of coercion, we want **Boomerang** to act as a deterrent as well, to prevent an attack in the first place.

**Naming**: The protocol is named after the Boomerang Nebula. The coldest place in the known universe. Has nothing to do with boomerang the throwing object.

## Table of Contents

- [Boomerang: Bitcoin Cold Storage With Built-in Coercion Resistance](#boomerang-bitcoin-cold-storage-with-built-in-coercion-resistance)
  - [Table of Contents](#table-of-contents)
  - [Motivation](#motivation)
  - [Our Solution](#our-solution)
    - [Predictability As An Attack Vector](#predictability-as-an-attack-vector)
    - [Design Goals](#design-goals)
    - [Solution Description](#solution-description)
    - [Scope And Positioning](#scope-and-positioning)
  - [Protocol Overview](#protocol-overview)
    - [Entities](#entities)
    - [Descriptor](#descriptor)
    - [Overarching Overview Of The Protocol](#overarching-overview-of-the-protocol)
    - [Setup And Withdrawal Steps In Code](#setup-and-withdrawal-steps-in-code)
    - [Overview Of The Boomerang Setup Procedure](#overview-of-the-boomerang-setup-procedure)
    - [Overview Of The Boomerang Withdrawal Procedure](#overview-of-the-boomerang-withdrawal-procedure)
    - [Design Decisions](#design-decisions)
  - [Duress Protection Mechanism](#duress-protection-mechanism)
  - [Economics Of Coercion](#economics-of-coercion)
    - [A Simple Game-Theoretic Model](#a-simple-game-theoretic-model)
    - [How Boomerang Shifts Incentives](#how-boomerang-shifts-incentives)
    - [Parameter Knobs As Economic Levers](#parameter-knobs-as-economic-levers)
  - [Game-Theoretic Threat Analysis Under Realistic Cost Assumptions](#game-theoretic-threat-analysis-under-realistic-cost-assumptions)
    - [Expected Withdrawal Duration Under Boomerang](#expected-withdrawal-duration-under-boomerang)
    - [Realistic Cost of Sustained Coercion](#realistic-cost-of-sustained-coercion)
    - [Escalation Probability Under Weekly Duress Checks](#escalation-probability-under-weekly-duress-checks)
    - [Two Escalation Regimes](#two-escalation-regimes)
    - [Attacker Decision Dynamics](#attacker-decision-dynamics)
    - [Deterrence Threshold](#deterrence-threshold)
    - [Core Insight](#core-insight)
    - [Practical Implication](#practical-implication)
    - [Final Assessment](#final-assessment)
  - [Security Model](#security-model)
  - [Ancillaries](#ancillaries)
  - [Concerns](#concerns)
  - [Why Boomerang Matters](#why-boomerang-matters)
    - [Current Designs Secures Keys. They Do Not Secure People](#current-designs-secures-keys-they-do-not-secure-people)
    - [The Core Insight: Predictability Is The Attacker’s Advantage](#the-core-insight-predictability-is-the-attackers-advantage)
    - [What Boomerang Does Differently](#what-boomerang-does-differently)
    - [Why This Matters Even If You Never Use Boomerang](#why-this-matters-even-if-you-never-use-boomerang)
    - [What Boomerang Is *Not*](#what-boomerang-is-not)
    - [The Question Boomerang Asks](#the-question-boomerang-asks)
  - [Proof-Of-Concept](#proof-of-concept)
  - [Roadmap](#roadmap)
  - [Who Are We](#who-are-we)

## Motivation

Bitcoin provides strong cryptographic guarantees for sovereign ownership, but those guarantees largely end at the private key. Once a key holder is identified and physically coerced, most existing custody schemes fail in predictable ways.

Today’s best practices like hardware wallets, multisig, air, gapped signing, and timelocks, are primarily designed to defend against remote attackers. Under coercion, these systems typically assume a cooperative signer who is free to act, and they offer little resistance when that assumption breaks.

Physical attacks against Bitcoiners are not hypothetical. They are [documented](https://github.com/jlopp/physical-bitcoin-attacks), increasing, and economically rational. Addressing this class of threats requires treating *human coercion* as a first-class part of the threat model rather than an out-of-scope exception.

**Boomerang** is an attempt to address this gap **without changing Bitcoin consensus**.

## Our Solution

### Predictability As An Attack Vector

Most custody designs are highly deterministic:

- The attacker knows who must sign.
- The attacker knows what steps must be taken.
- The attacker knows approximately how long withdrawal will take.

This predictability enables attackers to plan pressure, escalation, and logistics with confidence. Even time-locked schemes typically reduce to a known waiting period, after which funds become spendable.

**Boomerang** is motivated by the observation that **time determinism itself is an attack surface**.

### Design Goals

**Boomerang** aims to reduce the effectiveness of physical coercion by introducing **protocol-enforced uncertainty** into the withdrawal process, while remaining fully compatible with Bitcoin consensus.

The protocol is designed so that:

- Withdrawal duration is intentionally non-deterministic.
- No participant, including the victim, can know in advance when signing becomes possible.
- Duress signaling is mandatory, recurring, and plausibly deniable.
- Observable protocol behavior remains unchanged whether or not duress is signaled.
- Funds remain recoverable over time via deterministic fallback paths if hardware is lost.

**Boomerang** does not claim to eliminate coercion risk. Its goal is to materially increase attacker uncertainty and operational cost, and to create a reaction window for external intervention when duress occurs.

### Solution Description

How can one be truly protected in a coercion situation? By partially, yet effectively, abandoning what an attacker comes for in the first place. Total control over the asset.

An average attacker assumes predictability of the withdrawal procedure. They also assume the ability of the victim to run the aforementioned procedure and expect it to be concluded in a rather bounded time period. Being sure of such deterministic factors, the attacker will be able to plan for resources and intensify pressure to expedite the withdrawal process for a prompt escape.

The core idea of our solution is to create a taproot with 2 regimes of spending. In **Boomerang** regime, all keys are MuSig2 keys constructed from a normal key which the users create during setup and keep back ups of in form of mnemonics and passphrase, and one other key that is generated inside a java card applet (never backed up or revealed to anyone) and is programmed not to sign anything unless a randomly selected number (drawn from a user specified range but not known to anyone other that the applet inside the java card itself) of steps are passed via a rather lengthy ceremony that may take a few months, between the peers and a watchtower. The other regime solely uses the same normal keys mentioned earlier and is named normal regime.

That lengthy withdrawal ceremony is put there to guarantee a degree of unpredictability, as well as a suitable reaction window to the duress signal. In withdrawal ceremony, duress checks happen at initial transaction commitment and later at random intervals. Nevertheless, an encrypted payload is part and parcel of every message in the ceremony after initial transaction approval. Those payloads then get directed to an entity named SAR (Search And Rescue). SAR can unpack that payload and should it contain a positive duress check, use that same signal to decrypt the doxing data shared by user and go ahead with the search and rescue operation.

### Scope And Positioning

**Boomerang**, at this current stage of design, is not intended to replace simple cold storage or multisig setups for general use. It targets extreme threat models involving high-value holdings, personal safety risks, or adversaries capable of physical coercion.

More broadly, the protocol serves as an exploration of:

- Non-determinism as a security primitive in Bitcoin custody
- Duress-aware signing workflows enforced by hardware
- Taproot spending policies without consensus changes

We hope even if **Boomerang** itself is not widely adopted, the problems it addresses and the techniques it explores, be relevant to the ongoing evolution of Bitcoin custody and protocol design.

## Protocol Overview

### Entities

1. **User / Peers**: 5 entities that collaborate as the main key holders of the protocol. We name the peer from whose POV we explain the protocol, the user.
2. **Iso**: A stateless piece of software that is run on an isolated computing environment.
3. **Niso**: A stateful piece of software that runs on a non-isolated computing environment with access to a full bitcoin node and TOR network.
4. **Boomlet**: A java card applet that is installed and run on a java card.
5. **Boomletwo**: A java card applet that is installed and run on a java card, acting as the back up of Boomlet.
6. **ST**: Short for Secure Terminal. A tamper evident, air-gapped hardware used for trust minimization in communications between user and boomlet. More on ST can be found in [secure terminal folder](secure_terminal).
7. **WT**: Short for Watchtower. An online entity that manages coordination of the peers and act as one of the roots of trust to provide the heartbeat of the protocol which is the latest bitcoin block height.
8. **SAR**: Short for Search And Rescue. Is an online entity that receives encrypted doxing data from user and checks for the duress signal. Should it detect a positive duress signal, the SAR uses that signal to decrypt the doxing data and perform a search and rescue operation based on the aforementioned data. Since SAR's operations are mostly focused on physical security, this entity is jurisdiction dependent in order to comply with the pertinent use-of-force rules and regulations.
9. **Phone**: User's phone that registers with the SAR and provides encrypted dynamic and static data to the SAR.

### Descriptor

**Boomerang** descriptor is a taproot descriptor with two regimes:

```rust
                 unspendable key path
                /                    \
               /                      \
probabilistic boomerang regime       waterfall deterministic regime with time locks
```

1. **Boomerang** regime: In which only **Boomerang** keys work.
2. Normal regime: In which **Boomerang** keys work, as well as normal keys. But since normal keys work in this regime, there is no obligation to the non-deterministic withdrawals. This must have a waterfall structure to allow defense against keys being lost.

The overall structure of the taproot descriptor consists of two major parts. The first part is a tap tree containing keys that are in fact the MuSig aggregated keys generated by 2 separate keys. The first key is the *normal_pubkey* and the second one is *boom_pubkey*. The other part of the descriptor is made out of only *normal_pubkey*s with time locks.

The current design is as follows:

```rust
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

*milestone_block_0*: Block number after which withdrawal from the protocol is possible.

*milestone_block_n*: Block number after which pertinent script is spendable.

Hence, the only essential requirement on the descriptor is that it contains scripts with probabilistic keys (**Boomerang** scripts), that are executable before scripts with normal keys (normal scripts).

We have chosen the script in **Boomerang** regime to be a 5-of-5 to guarantee that the **Boomerang** protocol is in effect, even if only one user is committed to the protocol.

### Overarching Overview Of The Protocol

**Boomerang** is designed to address a threat that conventional cold storage largely ignores: **physical coercion of key holders**. Instead of trying to make keys impossible to steal, **Boomerang** makes *control over time* and *predictability of withdrawal* the scarce resource.

At a high level:

- Funds are locked in a **Taproot output** with two regimes:

  - A **boomerang (probabilistic) regime**, where spending requires MuSig2 keys that include an unrecoverable hardware-held component.
  - A **normal (deterministic) regime**, which becomes available only after time-locked milestones and allows graceful degradation if keys are lost.

- Each participant holds:

  - A recoverable *normal key* (backed up via mnemonic).
  - A non-backupable *boomlet key* generated and enforced inside a Java Card–class secure element.

- During setup, all peers agree on:

  - Descriptor structure and timelocks.
  - Watchtowers (WTs) for coordination and block-height anchoring.
  - Search-and-Rescue entities (SARs) for duress response.
  - A **personalized duress-consent pattern** memorized by each user and known only to their boomlet.

### Setup And Withdrawal Steps In Code

All steps are laid out clearly in [setup.rs](https://github.com/bitryonix/boomerang/blob/main/poc/src/setup.rs) and [withdrawal.rs](https://github.com/bitryonix/boomerang/blob/main/poc/src/withdrawal.rs) files, exactly following the design message diagrams of [setup](setup/setup_diagram_without_states.svg), [initiator withdrawal](withdrawal/initiator_withdrawal_diagram_without_states.svg) and [non-initiator withdrawal](withdrawal/non_initiator_withdrawal_diagram_without_states.svg) design files.  

### Overview Of The Boomerang Setup Procedure

The setup procedure for **Boomerang**, as depicted in the [PlantUML sequence diagram](setup/setup_diagram_without_states.puml) and its [SVG render](setup/setup_diagram_without_states.svg), involves multiple entities (User, Phone, SARs, Iso, Boomlet, ST, Niso, Peers, WT, and Boomletwo) coordinating to establish a secure cold storage with coercion resistance. It is divided into logical groups for clarity, following the diagram's structure. Each step includes key actions, data exchanges, and assertions/verifications. The process ensures key generation, parameter agreement, duress setup, registrations, synchronizations, and backups, culminating in a shared state across peers.

**Group 1: SAR Sign-Up (Steps 1-8: User registers with Search and Rescue via Phone)**

1. **User inputs initial SAR data to Phone**: User provides `doxing_password`, `sar_ids_collection`, and `static_doxing_data` to Phone. Phone computes `doxing_key` (SHA256 of `doxing_password`) and `doxing_data_identifier` (SHA256 of `doxing_key`).
2. **Phone initializes setup with SAR**: Phone sends `doxing_data_identifier` to SAR.
3. **SAR responds with payment info**: SAR generates and sends `sar_service_fee_payment_info` (including invoice, deadline, and SAR ID) back to Phone.
4. **Phone forwards payment info to User**: Phone verifies SAR ID matches the collection and sends info to User.
5. **User pays and provides receipt**: User verifies, pays the invoice, generates `sar_service_fee_payment_receipts`, and inputs it back to Phone.
6. **Phone sends receipt and encrypted data to SAR**: Phone encrypts `static_doxing_data` and `dynamic_doxing_data` with `doxing_key`, then sends receipt, encrypted data, and identifier to SAR (dynamic data sent at intervals).
7. **SAR verifies and syncs**: SAR checks payment receipt and identifier, stores encrypted data, and responds with a sync confirmation magic string.
8. **Phone notifies User of SAR registration**: Phone informs User that it's synced with SAR.

**Group 2: Duress Setup (Steps 9-26: Key generation, duress consent set, and confirmation via Iso, Boomlet, and ST)**

9. **User initializes Iso**: User provides `network`, `entropy_bytes`, `passphrase`, `doxing_password`, and `sar_ids_collection` to Iso. Iso generates mnemonic, master extended private key (`master_xpriv_0`), derives purpose root key, and creates `normal_pubkey_0`.

10. **Iso installs Boomlet applet**: Iso sends `normal_pubkey_0`, `doxing_key`, `sar_ids_collection`, and `network` to Boomlet. Boomlet generates identity keys, MuSig2 shares, boom pubkey, TOR secret key, and peer ID; responds with identity pubkey.

11-13. **Key exchange between Boomlet and ST via Iso**: Iso forwards Boomlet's pubkey to ST; ST generates and responds with its own pubkey; Iso forwards to Boomlet.

14-15. **Boomlet initiates duress setup**: Boomlet generates a random `duress_check_space` (5 numbers from 1-195), adds nonce, encrypts for ST, and sends to Iso; Iso forwards to ST.

16-18. **ST processes and User selects consent set**: ST decrypts, maps numbers to countries (alphabetical order), displays to User; User selects 5 countries (consent set) via joystick; ST records indices, adds the same nonce received earlier, encrypts for Boomlet, and sends to Iso.

19-20. **Iso forwards and Boomlet stores consent set**: Iso sends encrypted user's selection of consent set to Boomlet; Boomlet decrypts, verifies nonce, extracts and stores `duress_consent_set` from indices.

21-26. **Confirmation round**: Boomlet generates a new `duress_check_space`, repeats encryption/display/selection process (steps 14-20); User replicates selection; Boomlet verifies match (retries if failed, up to limits); drops redundant data and notifies Iso of completion.

**Group 3: Iso Completion and Niso Initialization (Steps 27-28: Wrap up offline setup)**

27. **Iso returns mnemonic to User**: Iso receives duress completion magic and provides mnemonic for safe storage.

28. **User connects Boomlet to Niso**: User shuts down Iso, connects Boomlet to Niso, and provides `network`, `rpc_client_url`, `rpc_client_auth` for Bitcoin node access.

**Group 4: Peer ID and TOR Setup (Steps 29-33: Boomlet and Niso establish TOR identity)**

29-31. **Niso queries Boomlet for TOR details**: Niso initializes with network/RPC info; Boomlet derives/sends peer ID, TOR secret key, and signed TOR address; Niso verifies signature, runs TOR circuit, and checks address match.

32-33. **Niso forwards to ST for verification**: Niso sends peer ID and signed address to ST; ST checks Boomlet pubkey; displays to User for out-of-band sharing.

**Group 5: Peer Parameter Exchange and Verification (Steps 34-41: Agree on shared params)**

34. **User exchanges peer data out-of-band**: User shares/receives peer IDs and signed TOR addresses with Peers, forms collections.

35. **User inputs params to Niso**: Provides `peer_addresses_collection`, `wt_ids_collection`, `milestone_block_collection`; Niso verifies signatures, connects via TOR, and forwards to Boomlet.

36-38. **Boomlet verifies and seeks User confirmation via ST**: Boomlet checks data; generates/encrypts `boomerang_params_seed` (peer IDs, WT IDs, milestones) with nonce for ST; Niso forwards; ST decrypts and displays to User.

39-41. **User approves via ST**: User verifies; ST signs/encrypts; Niso forwards to Boomlet; Boomlet verifies signature and nonce.

**Group 6: Peer Synchronization on Boomerang Params (Steps 42-45: Sign and share params)**

42-45. **Boomlet signs and peers sync**: Boomlet signs `boomerang_params`; Niso shares with Peers; collects/verifies signed params from all; forwards collection to Boomlet; Boomlet verifies all match and signatures are valid.

**Group 7: Mystery Generation and WT Registration (Steps 46-53: Draw mystery, register with WT)**

46. **Boomlet draws mystery**: Uniformly random between min/max tries (based on milestones); sets counter to 0.

47-49. **Boomlet initiates WT registration**: Generates registration request (peer IDs, params, signed data); encrypts for WT; Niso forwards.

50-53. **WT processes and peers sync**: WT decrypts/verifies; responds with suffixed signed acknowledgment; Niso forwards to Boomlet; Boomlet hashes/sends shared state fingerprint; Niso syncs with Peers; collects/verifies; Boomlet confirms all registered.

**Group 8: SAR Activation via WT (Steps 54-70: Connect SARs and sync)**

54-61. **Boomlet sends SAR data to WT**: Boomlet signs/encrypts SAR IDs and doxing identifier; Niso forwards; WT verifies, forwards identifier to SAR.

62-66. **SAR responds and WT relays**: SAR decrypts/verifies, generates/sends signed setup response (identifier, fingerprint, IV); WT suffixes/signs; Niso forwards; Boomlet decrypts/verifies; hashes/sends SAR finalization fingerprint.

67-70. **Peers sync SAR activation**: Niso shares fingerprints; collects/verifies; Boomlet confirms all have active SAR protection.

**Group 9: Boomletwo Backup (Steps 71-87: Create and verify backup)**

71-74. **User initializes backup via Niso/Iso**: Niso notifies User; User connects Boomletwo to Iso in backup mode; Iso installs applet; Boomletwo generates keys and responds with pubkey; Iso notifies User to connect Boomlet.

75-78. **User inputs backup data**: Provides milestones, network, mnemonic, passphrase, static doxing data, doxing password; Iso generates/sends signed backup request to Boomlet; Boomlet encrypts backup (excluding mystery) and sends with params/response; Iso reconstructs/verifies descriptor, doxing data.

79-85. **Transfer and verify backup**: Iso notifies User to connect Boomletwo; sends backup data; Boomletwo decrypts/imports, draws own mystery, signs completion; Iso notifies User to reconnect Boomlet; forwards signed completion; Boomlet verifies.

86-87. **Iso completes backup**: Notifies User of success.

**Group 10: Final Synchronization and Setup Completion (Steps 88-94: Activate backup state across peers)**

88-89. **User reconnects to Niso**: Notifies Niso; Niso sends finish magic to Boomlet.

90-92. **Boomlet generates and peers sync backup fingerprint**: Boomlet hashes/sends signed backup state; Niso shares/collects from Peers; verifies; forwards to Boomlet; Boomlet confirms.

93-94. **Boomlet notifies completion**: Sends done magic to Niso; Niso informs User.

### Overview Of The Boomerang Withdrawal Procedure

The withdrawal procedure in **Boomerang**, as depicted in the PlantUML sequence diagram for [initiator peer](withdrawal/initiator_withdrawal_diagram_without_states.puml) and [non-initiator peers](withdrawal/non_initiator_withdrawal_diagram_without_states.puml) and their pertinent [SVG render for initiator](withdrawal/initiator_withdrawal_diagram_without_states.svg) anf [SVG render for non-initiators](withdrawal/non_initiator_withdrawal_diagram_without_states.svg), allows peers to collaboratively spend from the cold storage UTXO using a PSBT (Partially Signed Bitcoin Transaction). It is divided into phases: initiation, approval, commitment (with initial duress check), a non-deterministic "digging game" (ping-pong loop with random duress checks), and finalization. The process inherits state from setup (e.g., keys, params, mysteries).

The **initiator** is the peer who creates and starts the PSBT. **Non-initiators** (the other 4 peers) receive notifications and participate in parallel but with some differing initial steps. After commitment, all peers follow identical steps in the digging game and signing. Steps are numbered sequentially across both diagrams for coherence, with notes on differences. Asynchronous/parallel actions are indicated (e.g., non-initiators perform similar actions concurrently).

Key assumptions: Withdrawal starts after `milestone_block_0`; duress checks use the country-based mechanism from setup; the "mystery" (random threshold per Boomlet) creates unpredictable delays (potentially months via ping-pong increments).

**Group 1: Transaction Initiation (Steps 1-9: Initiator creates and approves PSBT; notifies WT)**

These steps are initiator-only; non-initiators join later.

1. **User creates PSBT and inputs to Niso**: Initiator User generates PSBT (using a watch-only wallet) and provides it to Niso. Niso verifies block height ≥ milestone_0, derives `tx_id`, checks PSBT satisfiability, gets current `niso_0_event_block_height`.

2. **Niso forwards PSBT to Boomlet**: Sends PSBT and block height; Boomlet verifies height ≥ milestone_0, derives `tx_id`.

3-4. **Boomlet initiates ST check for tx_id**: Boomlet adds nonce, encrypts `tx_id_with_nonce` for ST, sends to Niso; Niso forwards to ST.

5-6. **ST displays and User approves tx_id**: ST decrypts/extracts `tx_id`, shows to User; User verifies match with PSBT-derived ID, notifies ST; ST signs/encrypts for Boomlet, sends to Niso.

7. **Niso forwards approval to Boomlet**: Boomlet decrypts/verifies ST signature, nonce freshness.

8. **Boomlet approves and encrypts for WT/peers**: Generates signed `peer_0_tx_approval` ("approved", tx_id, height); encrypts for WT; encrypts PSBT for each non-initiator Boomlet; sends collection to Niso.

9. **Niso forwards to WT**: Sends approval and encrypted PSBTs.

**Group 2: Transaction Approval (Steps 10-23: WT coordinates; all peers approve PSBT)**

10. **Initiator sends to WT (parallel to non-initiator receipt)**: WT receives initiator's approval and encrypted PSBTs (from step 9), decrypts/verifies "approved", freshness; generates/sends its own signed `wt_tx_approval` ("approved", tx_id, height, initiator ID) to all non-initiators, plus initiator's approval and per-peer encrypted PSBT.

11. **Non-initiators receive from WT**: Each non-initiator Niso gets WT approval, initiator approval, and its encrypted PSBT. Verifies signatures, "approved" magic, tx_id match, freshness (within tolerances from initiator/WT heights), height ≥ milestone_0; derives initiator ID.

12. **Non-initiator Niso forwards to Boomlet**: Sends received data plus its `niso_i_event_block_height` (i=1-4).

13. **Non-initiator Boomlet verifies and decrypts**: Verifies signatures/matches/freshness as above; decrypts PSBT; sends plain PSBT to Niso.

14-15. **Non-initiator Niso checks and User approves PSBT**: Niso verifies PSBT satisfiability, tx_id match; outputs PSBT and initiator ID to User; User approves, notifies Niso; Niso notifies Boomlet.

16-23. **Non-initiator tx_id check and approval (mirrors initiator's steps 3-9)**: Boomlet encrypts `tx_id_with_nonce` for ST; Niso forwards; ST decrypts/shows to User; User verifies; ST signs/encrypts; Niso forwards; Boomlet verifies. Boomlet generates signed `peer_i_tx_approval`, encrypts for WT and each peer (including initiator); sends collections to Niso; Niso forwards to WT.

**Group 3: Peer Synchronization on Approvals (Steps 24-27: WT collects/distributes all approvals)**

24. **Non-initiators send approvals to WT**: WT receives each non-initiator's approval and encrypted peer approvals.

25. **WT verifies and distributes**: For each, WT decrypts/verifies signature, "approved" magic, tx_id, freshness (within tolerance from initiator); generates signed `peer_i_tx_approval_signed_by_wt`; collects all (including initiator's); sends full collection to each peer (initiator and non-initiators).

26. **All peers receive full approvals**: Each Niso gets collection; verifies WT signatures on each; forwards to Boomlet with current block height.

27. **All Boomlets verify approvals**: Each verifies WT signatures, "approved" magic, tx_id matches, freshness (e.g., overall process within `n` blocks from start).

**Group 4: Transaction Commitment with Duress Check (Steps 28-44: Commit to PSBT with initial duress signal)**

28-33. **Initial duress check for commitment (all peers, concurrent)**: Each Boomlet initiates duress via ST (mirrors setup: generates 5 random country lists, encrypts for ST; Niso forwards; ST decrypts/maps to countries, shows to User; User selects via indices; ST encrypts back; Niso forwards; Boomlet decrypts/extracts signal).

34. **Duress signal processing (all peers)**: If consent set matched, `duress_placeholder_plaintext` = padding (no duress); else = `doxing_key` (duress). Encrypts placeholder for SAR.

35-37. **Boomlet generates commitment**: Signs `peer_i_tx_commit` ("commit", tx_id, height); pads with duress placeholder; re-signs padded version; encrypts for WT; sends to Niso; Niso forwards to WT.

38-39. **WT processes commitments**: For each peer, decrypts/verifies outer signature; extracts/separates duress placeholder; saves signed commit; forwards placeholder and peer pubkey to SAR.

40-41. **SAR handles duress**: Decrypts placeholder; if not padding and new (checks IV/pubkey pair), computes identifier, decrypts doxing data, enters rescue mode. Always signs placeholder, encrypts for Boomlet, sends back to WT.

42-44. **WT verifies and distributes commitments**: Verifies inner commit signature, "commit" magic, tx_id, freshness; signs each commit; collects all signed commits; sends full collection to each peer, plus per-peer SAR-signed placeholder.

**Group 5: Commitment Verification and Digging Game Start (Steps 45-47: Verify commitments; begin ping-pong)**

45. **All peers receive commitments**: Each Niso gets collection and its SAR placeholder; verifies WT signatures on commits; forwards to Boomlet with block height.

46. **All Boomlets verify commitments**: Verifies signatures/"commit"/tx_id/freshness (within 10-block tolerance from start); decrypts/verifies SAR signature on placeholder, matches sent one.

47. **Digging game initialization**: All Boomlets set `counter=0`, `ping_seq_num=0`, `reached_mystery_flag=0`, `reached_boomlets_collection={}` (tracks peers who reached mystery).

**Group 6: Ping-Pong Loop (Steps 48-59: Non-deterministic delay via counter increments; random duress checks)**

This loop runs until all Boomlets reach their mystery (counter ≥ mystery). All peers participate concurrently; duress checks occur pseudo-randomly (e.g., if rand() % DURESS_CHECK_INTERVAL == 0). Loop breaks when WT detects all reached.

48-51. **Ping creation (all peers)**: Each Boomlet generates `peer_i_ping` ("ping", tx_id, last_seen_block, seq_num, flag); signs; pads with duress placeholder (from random check or prior); re-signs padded; encrypts for WT; sends to Niso; Niso forwards to WT.

52. **WT processes pings**: Decrypts/verifies outer signature; extracts/separates placeholder; verifies inner ping signature, "ping" magic, tx_id, seq_num increment, freshness; if flag=1 and not already reached, notes it; forwards placeholder to SAR.

53-54. **SAR handles ping duress (mirrors step 40)**: Decrypts; triggers rescue if duress; signs/encrypts placeholder back; WT receives.

55-57. **WT creates pong**: Waits for all pings; collects signed pings; generates `pong` ("pong", tx_id, WT height, all pings); signs; encrypts per-peer; appends per-peer SAR-signed placeholder; sends to each Niso.

58. **All peers receive pong**: Niso forwards pong and placeholder with current height to Boomlet.

59. **Boomlet processes pong**: Decrypts/verifies WT signature, "pong" magic, tx_id, freshness; verifies each included ping's signature/magic/tx_id/seq_num/flag consistency; optionally triggers duress check (updates placeholder); if conditions met (block heights within boundaries), increments counter; updates reached collection if others flagged; sets own flag if counter ≥ mystery; updates last_seen_block; generates next ping (back to step 48).

**Group 7: Transaction Finalization and Relay (Steps 60-74: Sign and broadcast once all ready)**

60. **WT breaks loop and distributes reached pings**: When all flags=1, WT sends `reached_pings_collection` (final pings from all) to every peer.

61-62. **All peers verify reached state**: Niso receives/verifies each reached ping; hydrates PSBT; forwards to Boomlet; Boomlet re-verifies pings, tx_id match; updates PSBT.

63-64. **Prepare for signing**: Niso notifies User to connect Boomlet to Iso and input network/mnemonic/passphrase.

65-69. **MuSig2 signing (all peers, concurrent)**: Iso notifies Boomlet to start; Boomlet shares PSBT, descriptor, boom pubkey share, its nonce; Iso generates/sends its nonce and partial sig; Boomlet sends its partial sig; both aggregate/save signed PSBT; Iso notifies User to reconnect to Niso.

70-72. **Export signed PSBT**: User notifies Niso; Niso requests from Boomlet; Boomlet sends `psbt_signed_i`, clears withdrawal state (except sig), regenerates mystery.

73-74. **All send to WT**: Niso forwards signed PSBT to WT; WT aggregates all 5; derives/relays final signed transaction to Bitcoin network.

### Design Decisions

These decisions are not permanent and can be changed to modify the dynamic behavior of the system or strengthen protocol's security.

1. We have not considered reusability in terms of extending the **Boomerang** period somehow yet.
2. There is a minimum and maximum range for the mystery that translate into the minimum and maximum number of steps for the digging game. These are subject to the requirements and circumstances of the peers. Given the milestones and the range for mystery, one can have an optimal limit for starting the roll over process to move funds to a new **Boomerang** setup.
3. Withdrawal will not start before `milestone_block_0` in the current design. Although it could technically, since the digging game's end could well be after the aforementioned block.
4. All checks that are done in Boomlet that can be done on Niso as well, are taking place in both entities.

## Duress Protection Mechanism

The core idea for our duress protection mechanism is as follows. A more detailed description can be found in [duress protection folder](duress_protection).

1. Boomlet has the set of all countries in memory.
2. At setup user selects 5 countries that signals their consent. All other combinations of countries are interpreted as duress. (5-dictionary model of *Jeremy Clark and Urs Hengartner. 2008. Panic passwords: authenticating under duress. In Proceedings of the 3rd conference on Hot topics in security (HOTSEC'08). USENIX Association, USA, Article 8, 1–6.*)
3. At duress check instances boomlet generates 5 random combination of numbers from 1 to 195. Those sets are exported to the device. The device will assign to each number, a country in a way that the number corresponds to the ascending alphabetical order of the country.
4. If the user is not in duress, they will search each list and find the consent countries in no particular order.
5. If the user is in duress, they will choose any other countries.
6. If the user is tortured, any combination of countries can be confessed to as the consent signal.
7. If the attacker is to choose a country at random, it has a chance of `(1/195 * 1/194 * 1/193 * 1/192 * 1/191 = 1/267,749,239,680 = 3.73e-12` to hit the consent countries.

## Economics Of Coercion

Boomerang is designed not only to function under coercion, but also to be a **deterrent** by increasing attacker uncertainty and cost.

### A Simple Game-Theoretic Model

Model a coercion attempt as a sequential decision problem:

**Players**

* Defender (D): Boomerang user(s)
* Attacker (A): coercive adversary

**Parameters**

* (V): value attacker can steal
* (T): time attacker must maintain control until funds are safely exfiltrated
* (c): attacker’s per-unit-time operational cost (guards, logistics, safehouse, coordination)
* (q(T)): probability attacker is disrupted before completion (increasing with time)
* (L): attacker loss if disrupted (legal exposure, retaliation, sunk costs)

A simple attacker expected utility:

$$
U_A = (1-q(T))\cdot V - c\cdot T - q(T)\cdot L
$$

Attacker proceeds if (U_A > 0).

### How Boomerang Shifts Incentives

Boomerang changes the attacker’s payoff by changing the *distribution* of (T) and increasing (q(T)):

1. **Increases expected completion time (E[T])**
   The secure element will not sign until its secret mystery threshold is reached.

2. **Increases variance of (T)**
   Uncertainty forces the attacker to plan for worst-case duration, increasing cost and risk.

3. **Reduces “compressibility” of time via coercion**
   Torture/pressure often aims to reduce (T). Boomerang makes (T) less compressible because the victim cannot force Boomlet to sign early.

4. **Creates a reaction window**
   Duress checks happen at commitment and then again at random intervals during the digging game. Longer duration means more chances for duress signaling and SAR response, effectively raising (q(T)).

The objective is not “coercion-proof custody.” The objective is to push (U_A) negative for a meaningful range of attackers by raising time, uncertainty, and disruption probability.

### Parameter Knobs As Economic Levers

Several design parameters are best understood as deterrence levers:

* **Mystery min/max range:** increases (E[T]) and variance.
* **Duress check interval:** increases duress opportunities (higher chance of successful signaling) at the cost of greater ceremony burden.
* **Milestone schedule:** defines when forced determinism becomes available (and what an attacker might “wait out”).
* **WT redundancy / censorship resistance:** affects whether an attacker can DoS coordination and force fallback behavior.
* **SAR selection and jurisdiction:** affects the real-world meaning of (q(T)).

## Game-Theoretic Threat Analysis Under Realistic Cost Assumptions

Boomerang’s security claims are not based on making coercion impossible.
They are based on altering the *economics* of coercion.

To evaluate whether Boomerang deters real attacks, we must examine:

1. The realistic cost of sustaining coercion.
2. The statistical duration of withdrawal under Boomerang.
3. The probability and consequence of escalation (SAR activation).
4. The rational behavior of an attacker facing uncertainty.

This section aggregates those elements into a coherent game-theoretic assessment.

### Expected Withdrawal Duration Under Boomerang

In a typical high-security configuration:

* 5 peers
* Withdrawal window: 6–9 months
* Each peer samples a hidden threshold independently

Because completion depends on the **maximum** of all peers’ hidden values, the effective withdrawal duration clusters near the upper bound.

Empirically:

* Most withdrawals complete close to 9 months.
* Early completion is rare.
* The delay is bounded but statistically concentrated near the maximum.

For an attacker, this implies:

> Coercion must likely be sustained for close to the full configured window.

This is not a short hostage scenario. It is a multi-month operation.

### Realistic Cost of Sustained Coercion

A coercive detention lasting months requires:

* Continuous staffing (multiple shifts)
* Secure holding location(s)
* Logistics and transport
* Surveillance countermeasures
* Operational risk mitigation

Even modestly organized operations imply substantial daily costs.

Conservative estimates:

* **Low-end (aggressive underestimation):** ~$5,000 per day
* **Professional sustained operation:** $20,000–$30,000 per day

Over a 6–9 month window, even at the low estimate:

* 180 days × $5,000 ≈ $900,000
* 270 days × $5,000 ≈ $1,350,000

At more realistic professional levels:

* $4.5M–$8M in coercion cost alone

This cost exists before considering escalation risk.

### Escalation Probability Under Weekly Duress Checks

Boomerang embeds recurring duress opportunities.

If:

* Duress checks occur weekly,
* The victim successfully signals with probability π per check,
* SAR responds (bites) with probability ρ given a signal,

Then over 6–9 months (approximately 26–39 checks):

The probability that *at least one escalation occurs* becomes extremely high unless π or ρ is near zero.

For example:

* If π is high (e.g., 90%)
* And ρ is moderate (e.g., 50%)

Then escalation is overwhelmingly likely long before withdrawal completion.

In that case, the relevant question becomes:

> What does escalation mean for the attacker?

### Two Escalation Regimes

The deterrence strength of Boomerang depends critically on the real-world consequence of escalation.

#### Regime A: Escalation Leads to Detention and Prosecution<!-- omit from toc -->

If SAR activation plausibly results in:

* Arrest
* Prosecution
* Asset seizure
* Severe criminal penalties

Then escalation represents catastrophic loss.

Under such conditions:

* Expected attacker loss dominates potential gain.
* Even low daily coercion costs do not salvage profitability.
* The attacker’s optimal strategy becomes “do not attempt.”

In simulations using:

* $10M target
* 6–9 month window
* Weekly duress checks
* High victim signaling reliability
* Moderate SAR effectiveness
* Significant legal consequences

No break-even point was observed even at very high target values.

In this regime:

> Boomerang produces strong economic deterrence.

#### Regime B: Escalation Has Minimal Consequence<!-- omit from toc -->

If SAR activation:

* Does not meaningfully disrupt the operation,
* Does not create significant prosecution risk,
* Imposes only minor operational cost,

Then escalation becomes a nuisance rather than a deterrent.

Under such assumptions:

* Attackers may absorb the escalation loss.
* The primary cost driver becomes the time delay.
* For sufficiently large target values, coercion may remain economically viable.

In simulations where:

* Escalation loss was small (e.g., $100k),
* Daily coercion cost was minimized ($5k/day),

Break-even attack values dropped significantly (around a few million dollars).

In this regime:

> Boomerang provides delay and detection but weaker deterrence.

### Attacker Decision Dynamics

Boomerang converts coercion into a sequential decision problem.

At each passing week:

* Withdrawal has not completed → suggests a high hidden threshold.
* The expected remaining time remains substantial.
* The probability that escalation has already occurred increases.
* The attacker’s expected payoff declines.

The attacker must continuously decide:

* Continue coercion?
* Abort?

Under strong escalation consequences, the optimal strategy converges toward early abandonment.

Under weak escalation consequences, the attacker may rationally continue.

### Deterrence Threshold

The attack becomes economically rational only if:

> Target value > (expected coercion cost + expected escalation loss)

With a 6–9 month window:

* Coercion cost alone may approach $1M–$8M.
* If escalation carries multi-million-dollar legal risk, total expected downside exceeds many typical custody sizes.

Thus:

* For $10M-class holders in jurisdictions with credible enforcement,
* With properly configured withdrawal windows,
* And reliable duress signaling,

Boomerang materially alters attacker incentives.

### Core Insight

Boomerang’s security is not purely cryptographic.

It is economic.

It leverages:

* Time uncertainty,
* Extended duration,
* Recurring hidden escalation opportunities,
* Legal consequence compounding.

Delay alone does not deter coercion.

Delay combined with credible escalation does.

### Practical Implication

Boomerang is strongest when:

* Legal enforcement is credible.
* SAR capability is real.
* Duress signaling reliability is high.
* Withdrawal window is sufficiently long relative to target value.

Boomerang is weaker when:

* Escalation has no real consequence.
* Jurisdictional enforcement is absent.
* Duress signaling is unreliable.

In other words:

> Boomerang’s deterrence strength scales with the real-world enforceability behind it.

### Final Assessment

Under realistic multi-month withdrawal windows:

* Coercion becomes a prolonged, high-cost operation.
* Escalation risk compounds with time.
* Attacker success becomes uncertain and unstable.

Boomerang does not claim to eliminate coercion.

It reframes coercion from a predictable extraction into a high-variance, escalation-prone endurance gamble whose rationality depends on real-world enforcement.

## Security Model

During the design evolution of **Boomerang**, we faced multiple threats. Some of which blocked via design, and some of them remained to be addressed by usage instructions. Here we list some of the more important of those threats.

1. **Man-in-the-middle attacks**: We have used TOR for remote communications and we have used encrypted communications for local interactions. We also have engaged user for approving critical data in setup and withdrawal. Nevertheless, we have relied on two assumptions here:
   - We have an isolated environment in which Iso is running. Iso as a software is trusted and the hardware executing it is trusted as well. Here, we assume we are not overusing current assumptions of security prevalent in Bitcoin cold storage security.
   - We are using a Secure Terminal or ST, which we assume to be tamper evident. At the current design stage, we are opting for a do-it-yourself design to minimize the supply chain attack vector. But, we may also consider a pre-built device like common physical hardware wallets.
  
2. **Peers escaping protocol and deceiving an honest user to be in Boomerang**: We have made the **Boomerang** regime to be an n-of-n multisig. So that if even only 1 user is honest, the promise of unpredictability and duress protection will stand true.

3. **Need-to-know principle**: We have tried to abide by this principle in the protocol. This has led us to minimize unnecessary exposures to WTs and SARs except for when absolutely required. For example WT does not know of the descriptor and SAR can only know about the user when the duress signal is sent and SAR's involvement is demanded. Nevertheless, this begs the question of then what? Once the duress signal is sent, SAR will know about user's engagement with bitcoin. What if the SAR turns rouge after the duress signal and turn into an attacker itself at that instance or in the future? We don't have concrete answers to those questions as of now. SAR's requirement to rescue the user stipulates real world knowledge of the bitcoiner. For now, we are counting on reputation, social trust, and multiple SARs for one user to address this class of issues.

4. **Boomlet loss**: If the java card containing Boomlet is lost or broken or damaged in any way, the funds will be unreachable during the **Boomerang** regime and the duress protection is effectively void. The protocol collapses to a multi signature cold storage with a timelock. So, we thought of having back ups for the Boomlet. But having more back ups means more attack surface and more memory on boomlet to keep track of valid backups. We decided to have only one back up which we have named **Boomletwo**. The activation of the Boomletwo is the subject of an ancillary procedure that is yet to be designed. The principle here is that only one of the Boomlet or Boomletwo will be active at a time. In case of stealing the Boomlet, the user must be able to deactivate Boomlet and Activate Boomletwo without having access to Boomlet.

5. **Tampering/stopping dynamic doxing data**: We are at least partially relying on the dynamic doxing data sent by phone to the SAR, for recovering the user under duress. If the phone is compromised, the dynamic doxing data can be tampered with and weaken the reliability of search and rescue operation. For now, we are assuming enough security is provided by the phone's operating system. On the other hand if the flow of the dynamic doxing data is lost altogether, the best we might have are the static doxing data and the latest dynamic doxing data received. For now we don't have any protection against this vector attack, but we might be able to add some sort of proximity constraint to the phone somewhere in the protocol.

6. **Jurisdiction change**: Since the search and rescue operation is physical and involves use of force, the SAR's effectiveness is jurisdiction-dependent. At the current design, the user can select multiple SARs that can be in different jurisdictions. But if the user is abducted and transferred to a jurisdiction other than those in which the selected SARs can operate, the duress signal may lose at least some of its effectiveness. The SAR still can inform trusted persons of the situation and rely on them to inform pertinent authorities with cross-border capacities.

7. **Replay attacks**: We have put nonces and recency checks in messages to make sure that replay attacks are blocked. Recency checks in withdrawal takes place in approval and commitment phases. The freshness constraints are shown in [this diagram](withdrawal/block_constraints.svg). Nevertheless, in the current design, some Boomlet messages in setup are not unique to a specific instance of **Boomerang** setup. We will address setup uniqueness down the road.

8. **Forced determinism**: Forced determinism is a serious attack on **Boomerang**. Voiding its promises on duress protection via non-deterministic withdrawal. This situation can occur by events like losing both Boomlet and Boomletwo, starting withdrawal ceremony too late (too close to the normal regime in a way that the end of the digging game will happen after `milestone_block_1`), or non-cooperation of one of the users. The practical purposes of this attack may be:
   - The adversary might have compromised our normal keys and is waiting out for those keys to come into effect.
   - Our peers are waiting us out so they can move the funds without our consent in normal period.

    We tried rather extensively to mitigate this threat. Our discussions on this topic can be found in the [history of the interactions on this topic](security_models/forced_determinism.md). For now the only practical way we found to counter this attack is to require users to roll over in time and keep their Boomlets and Boomletwos safe.

9. **Social trust assumptions (WTs and SARs)**: While the protocol avoids *custody*, it introduces **operational trust**:

   - Watchtowers see metadata and coordinate liveness.
   - SARs are jurisdiction-bound, real-world actors with coercive authority.

    This will raise concerns about:

   - Privacy leakage.
   - Legal exposure.
   - Adversarial WTs or compromised SARs.
   - Whether SAR involvement creates new attack incentives rather than deterring them.

    For now we are assuming a reputation based system of trust for WTs and SARs as they are the service providers of this protocol and can be replaced at will (this is the subject of an ancillary procedure).

10. **Reliance on specialized trusted hardware**: The security model critically depends on:

    - Java Card–class hardware behaving correctly.
    - Side-channel resistance.
    - Correct implementation of cryptography, RNGs, and counters.

    At the same time, we as Bitcoiners are deeply skeptical of:

      - Proprietary secure elements.
      - Non-auditable hardware supply chains.
      - Long-term availability of niche hardware.

    If the boomlet is compromised, cloned, or subtly biased, the entire probabilistic guarantee collapses.
    For now, we tried to use common hardware for this protocol. Java cards can be procured rather easily and the ST hardware is not particularly niche.

11. **Complexity and attack surface**: **Boomerang** is rather complex:

    - Multiple entities (Iso, Niso, Boomlet, ST, WT, SAR, Phone).
    - Long, stateful ceremonies with many cryptographic checks.
    - Tight coupling between hardware behavior, protocol liveness, and human interaction.

    In Bitcoin, complexity is often viewed as *a vulnerability itself*. Even if every component is individually sound, emergent failure modes are hard to reason about, audit, and formally verify.

The comprehensive threat model is at its final stages and will be shared soon. The work in progress can be found in [security_models folder](security_models).

## Ancillaries

There are situations in which an entity must be changed or an interruption occurs. To handle those situations, we need to design ancillary procedures. Here we list those ancillary procedures that have been identified by now:

1. Switching from an unresponsive WT to a new one.
2. Activating Boomlet's backup (Boomletwo).
3. Changing the Phone.
4. Changing Niso.
5. Changing ST.
6. Modifying (adding or removing members of) the SAR set for a User.
7. Handling timeouts caused by freshness checks.
8. Handling prolonged response by a peer (blaming and informing other peers).

## Concerns

1. **Java card's ability to handel our computational expectations**. Should it prove inviable, we have to spread some calculations to other entities and use the java card as the verifier of some final result provided by a model of distributed partial trusts. As an other option, we can collapse the java card bearing the Boomlet and the ST into one device similar to current hardware wallets like ColdCard.
2. **Java card's expected life cycle**. We must examine if java card's expected longevity can handle the expected ping pong cycles reliably and what is the maximum upper limit of those cycles.
3. **The dynamical behavior of the system** can be unpredictable if not inspected carefully. The delays caused by network, irregularities in bitcoin block mining intervals, the delays caused by users answering duress checks caused by the expected interaction with the hardware and the time of the day (user can be asleep), when exposed to the freshness tolerances put in the withdrawal ceremony, can cause unexpected increase in withdrawal time. We are planning to do dynamic simulations and get a feel of the system behavior in various delay scenarios.

## Why Boomerang Matters

### Current Designs Secures Keys. They Do Not Secure People

Bitcoin’s cryptographic guarantees end at the private key. Once a key holder is identified and physically coerced, conventional cold storage offers no meaningful protection. Multisig, hardware wallets, and timelocks all assume a cooperative signer acting freely. Under duress, they fail in predictable ways.

Physical attacks against Bitcoiners are no longer theoretical. They are documented, increasing, and rational. Today’s best practices optimize against remote attackers while leaving key holders exposed to coercion, kidnapping, and extortion.

**Boomerang** is an attempt to address this gap **without changing Bitcoin consensus**.

### The Core Insight: Predictability Is The Attacker’s Advantage

Most custody schemes fail under duress because they are **deterministic**:

- The attacker knows *who* must sign.
- The attacker knows *what* must be done.
- The attacker knows *how long it will take*.

That knowledge lets attackers plan pressure, escalation, and logistics.

**Boomerang** removes one of those pillars: **time determinism**.

### What Boomerang Does Differently

**Boomerang** introduces a withdrawal process that is:

- **Unpredictable in duration**
  Signing is gated by secret, per-device thresholds enforced by secure hardware. No participant, including the victim, can know in advance when signing becomes possible.

- **Inescapable but deniable under duress**
  Duress checks are mandatory, recurring, and embedded in normal protocol traffic. Signaling duress does not alter observable behavior.

- **Consensus-compatible**
  Funds are locked using standard Taproot, MuSig2, and timelocks. No soft forks, covenants, or new opcodes are required.

- **Fail-gracefully recoverable**
  If hardware is lost or destroyed, deterministic fallback paths unlock over time using standard keys.

The result is not “perfect safety,” but a **material increase in attacker uncertainty and cost**.

### Why This Matters Even If You Never Use Boomerang

**Boomerang** is not claiming to be a universal custody solution. Its value lies elsewhere:

1. **It formalizes a neglected threat model**
   Physical coercion is usually waved away with “no system can protect against torture.” **Boomerang** treats this as an engineering constraint, not an excuse.

2. **It explores non-determinism as a security primitive**
   Bitcoin custody today is almost entirely deterministic. **Boomerang** shows how time uncertainty can be enforced *cryptographically*, not socially.

3. **It informs future protocol design**
   Even if **Boomerang** itself is not widely adopted, its ideas are relevant to vaults, covenants, and next-generation custody schemes.

### What Boomerang Is *Not*

- It is not a claim of “duress-proof Bitcoin.”
- It is not a replacement for simple cold storage.
- It does not rely on obscurity or hidden behavior.
- It does not assume honest attackers or perfect users.

Its guarantees are explicit, bounded, and openly discussed.

### The Question Boomerang Asks

Bitcoin has spent 17 years hardening keys against math, networks, and software bugs.

**What does it look like to harden Bitcoin against humans with guns?**

**Boomerang** may be one possible answer. Even if it is not the final one, the question itself seems overdue.

## Proof-Of-Concept

We have coded a proof-of-concept implementation in rust that can be reached at [boomerang repo](https://github.com/bitryonix/boomerang).

## Roadmap

- [x] Forming the core idea of friction as defense.
- [x] Finding a practical duress protection scheme.
- [x] Anchoring design to available and cheap hardware.
- [x] Designing the duress protection mechanism.
- [x] Designing the setup ceremony.
- [x] Designing the withdrawal ceremony.
- [x] Minimizing trust in entities involved to the extent possible.
- [ ] Extensive threat modeling.
- [ ] Design optimization.
- [ ] Addressing setup uniqueness.
- [ ] Creating the dynamical model for studying the expected behavior of the protocol, subject to nonlinearity and delays.
- [ ] Designing ancillaries.
- [ ] Designing error handling.
- [ ] Designing multiple WTs voting.
- [ ] Integrating **Boomerang** in project ***Helium***.

## Who Are We

We prefer to remain anonymous for the time being.

Our vision does not stop with **Boomerang**. This is just a part of a bigger picture we hope to draw. Currently we have other ideas that we are working on in parallel, and we hope to open source them soon as well.

Our main focus is on designing and implementing a coherent semantic and procedural spectrum that makes bitcoin and the system emerging from it, more accessible to everyone.
