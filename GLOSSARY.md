# Boomerang glossary

A lookup index for Boomerang terminology. [`spec/SPEC.md`](spec/SPEC.md) is
normative for exact protocol behavior.

## Contents

- [Scope and status](#scope-and-status)
- [Actors and components](#actors-and-components)
- [Keys and on-chain policy](#keys-and-on-chain-policy)
- [Withdrawal](#withdrawal)
- [Duress and response](#duress-and-response)
- [Security and failure](#security-and-failure)

## Scope and status

| Term | Meaning |
| --- | --- |
| **Boomerang** | A Bitcoin cold-storage protocol design that makes a coerced withdrawal an uncertain operation the coerced participants cannot accelerate. The same required progress repeatedly carries covert duress checks to a prearranged responder, increasing a payout-seeking attacker's burden while creating a response interval. |
| **Design status** | Boomerang is a draft and is not production-ready. Hardware validation, parameters, operating procedures, interoperability evidence, and response arrangements remain incomplete. |
| **Coercion-aware custody** | Custody that treats physical threats and compelled participation as part of the threat model, rather than considering only key theft or remote compromise. |
| **Attacker objective** | A valid, verifiable transfer, exfiltration of the bitcoin, and escape—not merely learning credentials. |
| **Ceremony** | A stateful process involving people, devices, and services. Boomerang has setup and withdrawal ceremonies. |
| **Primary Boomerang branch** | The earliest Taproot script branch: it requires a signature under each of the five peers' Boomerang public keys and cannot satisfy Bitcoin consensus before `milestone_block_0`. Separately, each Boomlet is specified to withhold its part of a signature until the off-chain withdrawal state machine permits signing. “Primary” identifies this first branch; it does not express an operator preference. |
| **Deterministic fallback** | Later normal-key branches beginning at `milestone_block_1`, then reducing from five required normal keys to one over subsequent milestones. They preserve recoverability but restore known on-chain availability. |

## Actors and components

| Term | Meaning |
| --- | --- |
| **Peer** | One joint custodian. The specified profile has exactly five peers. |
| **User** | The human operating one peer's components. |
| **Iso** | The isolated environment that derives or reconstructs the normal key and participates in setup and final signing. |
| **Niso** | The online environment used for peer and WT coordination, Tor communication, and Bitcoin RPC access. It is not trusted to authorize a spend. |
| **Boomlet** | The active trusted device. It holds host-inaccessible private material and protocol state, evaluates ST input, generates the fresh withdrawal mystery, enforces progress, and participates in signing. |
| **Boomletwo** | The designated inactive backup device. It may receive setup state only through the authorized, authenticated, target-bound backup envelope. Activation, revocation, and anti-clone procedures remain open. |
| **ST / Secure Terminal** | The air-gapped trusted display and input device for transaction-identifier confirmation, consent enrollment, and duress challenges. |
| **WT / Watchtower** | The active coordination service for a ceremony. It verifies and relays protocol objects, obtains SAR acknowledgments, supplies a block-height view, collects progress, aggregates final fragments, and broadcasts. It does not hold a custody key. |
| **SAR / Search and Rescue** | The setup-bound responder that processes encrypted placeholders and may initiate an external response. Its legal authority and real-world effectiveness are outside the cryptographic guarantee. |
| **Phone** | The user device that registers and updates encrypted rescue information with SAR. |

## Keys and on-chain policy

| Term | Meaning |
| --- | --- |
| **Normal key** | The mnemonic-backed recoverable key held or reconstructed by Iso. It is one part of a peer's Boomerang key and is used directly by fallback branches. |
| **Boom key share** | Boomlet-held MuSig2 private material. It is unavailable to the host and never exported in plaintext; its only permitted export is inside the authenticated envelope bound to the authorized Boomletwo. |
| **Boomerang public key / `boom_pubkey`** | The aggregate of one peer's normal public key and Boomlet public share. |
| **Taproot output** | The Bitcoin output whose script tree contains the primary Boomerang branch and later normal-key fallback branches. Its internal key is unspendable. |
| **Milestone block** | An agreed absolute block height. `milestone_block_0` gates the primary branch; `milestone_block_1` through `milestone_block_5` gate fallback branches. |
| **Timelock** | A Bitcoin consensus condition that prevents a branch from being used before its milestone. Consensus enforces these timelocks and the spending policy, not Boomlet mysteries or off-chain progress. |
| **MuSig2** | The signing protocol used to combine the normal and Boomlet parts of a peer key and to produce the required signatures under the specified profile. |
| **Descriptor** | The exact, checksummed description of the Taproot spending policy agreed during setup. |

## Withdrawal

| Term | Meaning |
| --- | --- |
| **PSBT** | Partially Signed Bitcoin Transaction, used to coordinate review and signing. |
| **`tx_id`** | The identifier derived from the unsigned transaction that users independently verify on ST. |
| **Transaction review / tx-ID confirmation** | A withdrawal step in which a user independently checks that ST displays the `tx_id` of the intended unsigned transaction. ST signs the exact nonce-bound confirmation to Boomlet. This is not a Bitcoin transaction signature. |
| **Initiator / non-initiator** | Per-withdrawal roles. The peer that starts a withdrawal and supplies its transaction is the initiator; the other four peers are that ceremony's non-initiators. All five must review, approve, and commit. |
| **`TxApproval`** | A Boomlet-signed, pre-signing protocol authorization created after that peer's successful transaction review. It is bound to `withdrawal_id`; it is not a Bitcoin transaction signature and cannot spend funds. |
| **Approval set** | The ordered collection containing exactly one valid `TxApproval` from each of the five peers. Every participant verifies the set before computing `approved_withdrawal_id`. |
| **Approval-set attestation** | A non-initiator Boomlet's signed evidence that it received and verified the complete approval set and WT approval and computed the same `approved_withdrawal_id`. The four attestations are not authorization, commitments, or duress evidence. |
| **`withdrawal_id`** | The binding among the active setup, unsigned transaction `tx_id`, initiator identity, and initiator approval nonce. |
| **`approved_withdrawal_id`** | The binding over `withdrawal_id` and the exact ordered five-`TxApproval` set. It scopes every later commitment, placeholder, ping, pong, reached report, signing step, and replay check. |
| **`TxCommit` / commitment** | A Boomlet-signed binding to `approved_withdrawal_id`. Its authenticated encrypted wrapper also carries the initial duress placeholder. It is not a Bitcoin transaction signature. |
| **`DIGGING`** | The withdrawal state, also called the digging game, entered only after the complete signed `TxCommit` collection and that Boomlet's exact initial SAR acknowledgment have been verified. Entry generates the withdrawal's fresh mystery; no mystery exists during setup or `TxApproval` collection. |
| **Mystery** | A fresh secret threshold generated by each Boomlet only when the active withdrawal enters `DIGGING`, after the required `TxCommit` collection and SAR acknowledgment verification. Its bounds are protocol-profile constants, not setup choices. It is erased with the active withdrawal state after export, abort, or unrecoverable failure. |
| **Counter** | A Boomlet's successful progress-round count for the active withdrawal. It advances only when the pong, chain view, and every other peer's included ping satisfy the specified checks. |
| **Readiness share (`x`)** | An analysis quantity: the share of the allowed mystery values `{m, ..., M}` at or below a common counter value `k`. One Boomlet is ready with probability `x`; all five with `x^5`. It is neither elapsed time nor percent-complete. Defined in [DESIGN §12](DESIGN.md#12-attack-economics-and-security-argument) and [coercion economics §4](security_models/coercion_economics.md#4-protocol-derived-completion-distribution). |
| **ping / pong** | The repeated progress exchange among Boomlets and WT. Every ping carries a freshly encrypted placeholder; WT must obtain its exact SAR acknowledgment before using that ping in a pong. |
| **Reached mystery** | The one-way active-withdrawal state set when `counter >= mystery`. A reached Boomlet continues the loop until all five have current reached pings. |
| **Reached collection** | WT's collection proving that every peer's current ping reports its threshold reached. Signing is not permitted before it verifies. |
| **Hydration** | Adding allowed signing-support metadata to the PSBT bound by the verified `TxApproval` set without changing its transaction semantics, ordering, sighash policy, or `tx_id`. |
| **Rollover** | Moving funds into a fresh setup before deterministic fallback becomes an attractive predictable target. Complete operational procedures remain to be defined. |

## Duress and response

| Term | Meaning |
| --- | --- |
| **Consent set** | The unordered five-country set memorized by a user during two-round enrollment and stored by Boomlet. The versioned display vocabulary contains exactly 193 entries. |
| **Duress challenge** | A fresh nonce-bound prompt through ST that presents the 193-entry vocabulary in five independently shuffled columns; the user selects one entry per column. Selecting the enrolled set means safe; any other structurally valid five-element set means duress. |
| **Placeholder** | A SAR-encrypted payload carrying either inert zeros or the setup-bound rescue-data unlock value. It is present on every duress-bearing commitment and ping path. |
| **Freshly encrypted placeholder** | A new authenticated encryption envelope with a fresh IV. Every ping requires one even when no new challenge ran and its underlying plaintext is unchanged. |
| **SAR acknowledgment** | SAR's encrypted signature over the exact placeholder envelope it received. Each SAR deployment uses a fixed acknowledgment delay. On receipt, SAR schedules release for the receipt time plus that delay and releases the acknowledgment exactly then for valid safe and duress placeholders. If processing misses that time, SAR exposes the same failure and sends no late acknowledgment. Before release, SAR durably records processing and, for new valid duress, activates duress response. The acknowledgment proves exact delivery and activation—not timely, lawful, effective, correctly directed, or safe intervention. |
| **Plausible deniability** | Indistinguishability only on the specified protocol-observability surface. Valid safe and duress handling share response shape, routing, the same receipt-relative acknowledgment delay, durable-write path, retry behavior, and externally visible failure behavior. It does not cover physical observation, compromised devices, learned consent responses, private SAR diagnostics, metadata outside that surface, or a responder revealing the signal. |
| **`DynamicRescueUpload`** | Encrypted observation authenticated with `dynamic_update_auth_key`. SAR retains each accepted upload. |
| **`DynamicRescueReceipt`** | SAR's signed receipt for the exact accepted upload. |
| **`dynamic_update_auth_key`** | Password-derived key used to authenticate `DynamicRescueUpload` objects. SAR stores it at registration and verifies each upload's CMAC with it. |
| **Rescue information** | Static data and retained dynamic uploads, available to SAR after duress. The rescue application interprets their relevance and freshness. |

## Security and failure

| Term | Meaning |
| --- | --- |
| **Forced determinism** | Movement of the practical withdrawal path from Boomlet-enforced uncertainty to a known fallback schedule, for example through late start, stalling, device loss, or peer non-cooperation. |
| **N-of-N / five-of-five** | All five peers are required for the primary branch. This blocks a bypassing subset but lets any peer or required dependency stall progress. |
| **Fail closed** | Stop advancing an invalid or unsafe active ceremony while retaining its bindings until a valid retry, recovery input, or explicit abandonment. Failure does not authorize fallback early. |
| **Ancillary procedure** | A required operational procedure—WT switching, Boomletwo activation, Phone/Niso/ST replacement, response to prolonged setup-bound SAR unavailability, freshness-timeout handling, blame—identified but not yet specified. Their absence is an open assumption; see [DESIGN §15](DESIGN.md#15-ancillary-procedures-and-open-protocol-work). |
| **Liveness** | The ability to complete a valid withdrawal. It depends on every peer, Boomlet, WT, SAR, chain view, and other required dependency during the primary ceremony. |
| **Trust boundary** | A point where the argument depends on correct behavior or protected state, including Boomlet, ST, Iso, Niso, Phone, WT, SAR, and Bitcoin RPC. |
| **Response interval** | Time created between a durable duress activation and possible signing. It is useful only with a credible, prepared response; prolonging coercion can instead increase human harm. |
