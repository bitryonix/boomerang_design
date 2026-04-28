# Boomerang Glossary

This glossary translates Boomerang’s internal names into plain language. It is intentionally short. The protocol specification remains the authority for exact behavior.

---

## Table of Contents

- [Big-picture terms](#big-picture-terms)
- [Actors and components](#actors-and-components)
- [Keys and on-chain structure](#keys-and-on-chain-structure)
- [Withdrawal terms](#withdrawal-terms)
- [Emergency-signal terms](#emergency-signal-terms)
- [Risk terms](#risk-terms)
- [Acronym quick list](#acronym-quick-list)
- [One-sentence summary](#one-sentence-summary)

## Big-picture terms

| Term | Plain meaning |
| --- | --- |
| **Boomerang** | A research-stage Bitcoin custody protocol that makes the primary withdrawal path bounded but unpredictable in time and includes hidden emergency signaling. |
| **Coercion-aware custody** | Custody design that considers a person being pressured, threatened, or forced to participate, not only remote key theft. |
| **Bounded unpredictability** | The withdrawal cannot take forever, but nobody knows the exact moment when it will become signable. |
| **Time determinism** | The property that a withdrawal has a known or easy-to-estimate completion time. Boomerang treats this as an attack surface. |
| **Ceremony** | A multi-step operational process involving humans, devices, and services. Setup and withdrawal are both ceremonies. |
| **Primary path** | The Boomerang withdrawal path: stricter, slower, less predictable, and dependent on Boomlets. |
| **Fallback path** | Later timelocked normal-key spending paths that preserve recoverability if the primary path cannot complete. |
| **Research-stage** | The design is being explored and reviewed. It is not ready for production funds. |

---

## Actors and components

| Term | Plain meaning |
| --- | --- |
| **Peer** | One member of the custody group. In the current design profile, Boomerang assumes five peers. |
| **User** | The human operating one peer’s devices. |
| **Boomlet** | The active secure element or smartcard-like device. It holds a non-exportable key share, tracks withdrawal progress, enforces the mystery threshold, and evaluates emergency-signal challenges. |
| **Boomletwo** | A backup secure element. Its activation and lifecycle details are still design work. |
| **Iso** | The isolated/offline environment used for sensitive operations such as normal-key derivation and final signing. |
| **Niso** | The networked environment used for coordination, peer communication, Bitcoin node access, and Watchtower interaction. |
| **ST / Secure Terminal** | An air-gapped human interface for checking transaction details and answering emergency-signal challenges. Its job is to reduce trust in networked computers. |
| **WT / Watchtower** | A coordination service. It relays messages, helps track liveness and block height, and aggregates final signed fragments. It is not supposed to custody funds. |
| **SAR / Search and Rescue** | A real-world response service. It receives encrypted placeholders and interprets them only if they contain a valid duress signal. Its legal and operational model is jurisdiction-dependent. |
| **Phone** | The user device used to register encrypted rescue information with SAR and update dynamic rescue information. |

---

## Keys and on-chain structure

| Term | Plain meaning |
| --- | --- |
| **Taproot output** | The Bitcoin output that holds the funds and contains the Boomerang and fallback spending branches. |
| **Descriptor** | A precise description of the spending policy for the Bitcoin output. |
| **Normal key** | A recoverable key backed by mnemonic/passphrase material. It is used in fallback paths and as one part of a Boomerang key. |
| **Boom key share** | A non-exportable key share generated and held inside the Boomlet. |
| **Boom pubkey** | The combined public key for a peer’s Boomerang signing role, produced from the normal key and Boomlet-held share. |
| **MuSig2** | A multisignature-style signing method used to aggregate signing keys. The exact implementation profile belongs in the spec and implementation docs. |
| **Timelock** | A Bitcoin rule that prevents a branch from being spendable until a specified block height or time condition. |
| **Milestone block** | A block height at which a new spending branch becomes available or a protocol stage becomes valid. |
| **Waterfall fallback** | A sequence of later fallback branches whose required signer threshold gradually decreases over time. |

---

## Withdrawal terms

| Term | Plain meaning |
| --- | --- |
| **Withdrawal** | The process of moving funds out of the Boomerang output. |
| **PSBT** | Partially Signed Bitcoin Transaction. A standard way to coordinate transaction review and signing before final broadcast. |
| **Commitment** | The point where peers bind themselves to a specific transaction identity so later steps cannot silently become a different withdrawal. |
| **Hydration** | Filling in the final transaction/signing details after the waiting process is complete. Boomlet must verify continuity with the committed transaction. |
| **Ping / pong** | Repeated progress messages between Boomlets, peers, and Watchtower during the waiting loop. |
| **Freshness** | Evidence that a message belongs to the current withdrawal and is not a stale replay from an earlier state. |
| **Counter** | A Boomlet’s internal progress value during withdrawal. |
| **Mystery** | The Boomlet’s secret threshold. When the internal counter reaches this value, the Boomlet becomes ready to sign. |
| **Reached mystery flag** | The device state saying its mystery threshold has been reached. Once true, it should not revert to false for that ceremony. |
| **Fail closed** | Refuse to advance the current Boomerang ceremony when checks fail. This prevents unsafe progress but may require recovery or fallback planning. |

---

## Emergency-signal terms

| Term | Plain meaning |
| --- | --- |
| **Duress** | A situation where a user is not acting freely. The design tries to let the user signal this without making the signal visible to the attacker or other protocol observers. |
| **Consent pattern** | The memorized answer pattern that means “I am safe; continue.” In the current design, this is represented with a set of countries. |
| **Challenge** | A prompt shown by the Secure Terminal during withdrawal. The user’s answer is evaluated by the Boomlet. |
| **Duress placeholder** | An encrypted payload included in ordinary protocol messages. It should look structurally normal whether the user is safe or signaling duress. |
| **Safe placeholder** | A placeholder that decrypts to inert material, such as zeros or padding. |
| **Duress placeholder with unlock material** | A placeholder that gives SAR enough information to identify and decrypt the user’s encrypted rescue information. |
| **Rescue information** | The information SAR would need to identify and help the user in an emergency. Some older project docs call this `doxing data`; this glossary uses `rescue information` because the intended purpose is emergency response, not public exposure. |
| **Plausible deniability** | The property that the visible protocol flow does not obviously reveal whether the user signaled safe or duress. |

---

## Risk terms

| Term | Plain meaning |
| --- | --- |
| **Forced determinism** | A failure mode where the system is pushed out of the unpredictable Boomerang path and into the predictable fallback path. This can happen through lost hardware, non-cooperation, or starting too late. |
| **Liveness** | The ability of the system to eventually complete a valid withdrawal or recovery. |
| **Recoverability** | The ability to regain access to funds after loss, failure, or non-cooperation. |
| **Metadata exposure** | Information leaked by timing, communication patterns, service usage, peer identities, or operational behavior, even when keys remain safe. |
| **Trust boundary** | A place where the system relies on a component or actor to behave correctly. Boomlet, Secure Terminal, Watchtower, SAR, Iso, Niso, and Phone each create different boundaries. |
| **Assumption register** | The project document that lists important assumptions and design gaps. It is useful for reviewers who want to separate claims from dependencies. |

---

## Acronym quick list

| Acronym | Expands to | Role |
| --- | --- | --- |
| **ST** | Secure Terminal | Air-gapped user interface |
| **WT** | Watchtower | Coordination and liveness service |
| **SAR** | Search and Rescue | Emergency-response service |
| **PSBT** | Partially Signed Bitcoin Transaction | Transaction coordination format |
| **PoC** | Proof of Concept | Experimental implementation, not production code |

---

## One-sentence summary

Boomerang is a research design where Bitcoin funds are recoverable, but the preferred withdrawal path cannot be made predictably fast merely because a human is pressured.
