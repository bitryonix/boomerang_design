# Boomerang: A Bitcoin Cold Storage with Duress Protection

Boomerang is a bitcoin cold storage protocol that provides duress protection via a non-deterministic withdrawal mechanism, interweaved with duress checks.

Boomerang is suitable for enterprises and individuals with large sums of bitcoin in treasury mode or as a secure fallback for deleted key covenant structures like [Ajolote](https://arxiv.org/abs/2310.11911) or [Revault](https://wizardsardine.com/revault/).

We use secure execution environments to make signing a non-deterministic procedure in time. This non-deterministic duration provides a shield for user to signal duress, without changing anything in the procedure to signal off the attacker.

We don't take refuge in security through obscurity. On the contrary, our goal is to make Boomerang a deterrence factor for an average attacker.

## What we need

We need your feedbacks on idea and implementation in design to get a feel if we are on right track or not.

## Table of Contents

- [Boomerang: A Bitcoin Cold Storage with Duress Protection](#boomerang-a-bitcoin-cold-storage-with-duress-protection)
  - [What we need](#what-we-need)
  - [Table of Contents](#table-of-contents)
  - [The problem](#the-problem)
  - [Our solution](#our-solution)
  - [Protocol overview](#protocol-overview)
    - [Entities](#entities)
    - [Descriptor](#descriptor)
    - [Setup overview](#setup-overview)
    - [Withdrawal overview](#withdrawal-overview)
    - [Design decisions](#design-decisions)
  - [Duress protection mechanism](#duress-protection-mechanism)
  - [Security model](#security-model)
  - [Proof-of-concept](#proof-of-concept)
  - [Concerns](#concerns)
  - [Roadmap](#roadmap)
  - [Call for collaboration](#call-for-collaboration)
  - [Financial support](#financial-support)
  - [Updates](#updates)

## The problem

A classic cold storage setup is mostly about keeping the bitcoins safe. The bitcoiner, on the other hand, is rather out of the enclave of safety and left with their own devices to keep themselves safe.

The ever-growing list of [physical attacks on bitcoiners](https://github.com/jlopp/physical-bitcoin-attacks), begs addressing effective duress protection for the bitcoiner.

## Our solution

Given the problem statement, we have tried our best to address the issue to the extend possible, without any changes in bitcoin protocol.

How can one be truly protected in a duress situation? By partially, yet effectively, abandoning what an attacker comes for in the first place. Total control over the asset.

An average attacker counts of predictability of the withdrawal procedure and the ability of the victim to run the aforementioned procedure in a rather pre-determined and bounded time period. This period can be 10 minutes or 2 days if they can take hostage the owner's loved ones. Being sure of such deterministic factors, the attacker will be able to plan for resources and intensify pressure to expedite the withdrawal process for a prompt escape.

The core idea of our solution is to create a taproot with 2 regimes of spending. In boomerang regime, all keys are MuSig2 keys constructed from a normal key which the operatives create during setup and keep back ups of in form of mnemonics and passphrase, and one other key that is generated inside a java card applet (never backed up or revealed to anyone) and is programmed not to sign anything unless a randomly selected number (drawn from a user specified range but not known to anyone other that the applet inside the java card itself) of steps are passed via a rather lengthy ceremony that may take a few months, between the peers and a watchtower. The other regime solely uses the same normal keys mentioned earlier and is named normal regime.

That lengthy withdrawal ceremony is put there to guarantee a degree of unpredictability, as well as a suitable reaction window to the duress signal. In withdrawal ceremony, duress checks happen at initial transaction approval and later at random intervals. Nevertheless, an encrypted payload is part and parcel of every message in the ceremony after initial transaction approval. Those payloads then get directed to an entity named SAR (Search And Rescue). SAR can unpack that payload and should in contain a positive duress check, use that same signal to decrypt the doxing data shared by user and go ahead with the search and rescue operation.

## Protocol overview

### Entities

1. **User / Peers**: 5 entities that collaborate as the main key holders of the protocol. We name the peer from which POV we explain the protocol, the user.
2. **Iso**: A stateless piece of software that is run on an isolated computing environment.
3. **Niso**: A stateful piece of software that runs on a non-isolated computing environment with access to a full bitcoin node and TOR network.
4. **Boomlet**: A java card applet that is installed and run on a java card.
5. **Boomletwo**: A java card containing the back up of Boomlet.
6. **ST**: Short for Secure Terminal. A tamper evident, air-gapped hardware used for trust minimization in communications between user and boomlet. More on ST can be found in [secure terminal folder](secure_terminal)
7. **WT**: Short for Watchtower. An online entity that manages coordination of the peers and act as one of the roots of trust to provide the heartbeat of the protocol which is the bitcoin block height.
8. **SAR**: Short for Search And Rescue. Is an online entity that receives encrypted doxing data from user and checks for the duress signal. Should it detect a positive duress signal, the SAR uses that signal to decrypt the doxing data and perform a search and rescue operation based on the aforementioned data. Since, SAR's operations are mostly focused on physical security, this entity is jurisdiction dependent in order to comply with the pertinent use-of-force rules and regulations.
9. **Phone**: User's phone that registers with the SAR and provides encrypted dynamic and static data to the SAR.

### Descriptor

Boomerang descriptor is a taproot descriptor with two regimes:

```rust
                 unspendable key path
                /                    \
               /                      \
probabilistic boomerang regime       waterfall deterministic regime with time locks
```

1. Boomerang regime: In which only boomerang keys work.
2. Normal regime: In which boomerang keys work, as well as normal keys. But since normal keys work in this regime, there is no obligation to the non-deterministic withdrawals. This must have a waterfall structure to allow defense against keys being lost.

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

Hence, the only essential requirement on the descriptor is that it contains scripts with probabilistic keys (boomerang scripts), that are executable before scripts with normal keys (normal scripts).

We have chosen the script in boomerang regime to be a 5-of-5 to guarantee that the boomerang protocol is in effect, even if only the self is committed to the protocol.

### Setup overview

The core idea for setup procedure is as follows. A more detailed description, alongside detailed sequence diagrams can be found in [setup folder](setup).

1. *Peer*s agree on a set of *WT*s as well as the time structure of the descriptor, which is a collection of block heights to be put in place of `milestone_block_0` ... `milestone_block_5`.
2. The *user* registers with a private *SAR* of their own choosing, pertinent to the jurisdiction they live in, via their *phone*. They provide a `doxing_password` and a set of `static_doxing_data` to their *phone* and the *SAR* in the process.
3. The *user* initiates *iso* with information needed for creating a private key, the `doxing_password` and the identity of the *SAR* with which the *user* had previously registered.
4. *Iso* creates the master keypair and a `normal_pubkey` from a predefined derivation path, that is used in the normal regime of the protocol.
5. *Iso* installs the *boomlet* applet on an empty java card. The installation payload consists of the `normal_pubkey`, the `doxing_key` (which is the sha256 digest of the `doxing_password`) and the identity of the *SAR*.
6. Upon installation, *boomlet* creates an internal identity keypair that never leaves the java card.
7. *Boomlet* then creates an aggregated MuSig2 public key with its own `identity_pubkey` and the `normal_key` received. This new key is named `boom_key`. *Boomlet* also creates a `tor_secret_key` that is used for secure communication between entities over TOR.
8. Now we enter into duress setup. *Boomlet* and *ST* perform a key exchange via *iso*. From now on, any data exchange between *boomlet* and *ST* will be encrypted by this shared key. This is to ensure *user* is seeing exactly what *boomlet* meant for it to see and there is no "man in the middle" attack vector specifically when the coordinator is *niso* during the withdrawal phase, in which it is connected to the web and TOR.
9. User selects 5 words from a pre-defined set (we have chosen the names of all countries as our dictionary) as their `duress_consent_set`. The order does not matter. The exact mechanism is laid out in [duress design folder](duress).
10. The `duress_consent_set` is saved inside *boomlet* and the *user* must remember these 5 words.
11. Now is the time to install *niso* and start communication among *users*. Each *user* installs *niso* on a non-isolated computing environment that is TOR-enabled and runs a bitcoin full node.
12. Each *user* connects the card containing *boomlet* to the *niso* so that *niso* can fetch data about the identity of *boomlet* and the tor secret key that *boomlet* had previously made.
13. *Users* exchange their `peer_id` and their `tor_secret_key` with each other, via a secure, previously established, out of band channel. Say Signal messaging app.
14. Now, each user has what we call `boomerang_params` that consists of all `peer_id`s, a collection of pre-selected `wt_id`s and the `boomerang_descriptor`. This `boomerang_param` that uniquely defines the setup, then is transferred to each *user*'s *boomlet*.
15. *Boomlet* checks the `boomerang_params` with the *user* via the *ST* for the last time to make sure everything is integral.
16. Now, each *boomlet* signs the `boomerang_params` and shares it via TOR with other *peer*s. All *peers* then collect and check if every other *peer* has arrived at the same `boomerang_params` and the signature is valid.
17. After making sure that everyone is on the same page regarding the `boomerang_params`, each *boomlet* uniformly draws a random number between `0` and `milestone_block_1 - milestone_block_0` and saves it internally as `mystery`. This will never be revealed to anyone, unless reached in the digging game in withdrawal ceremony. Each *boomlet* then sets a variable named `counter` to `0`. The point is, not until the `counter` reaches the `mystery`, can *boomlet* sign a transaction. Here, because the draw happens uniformly random, there is a significant probability that reaching an total readiness state for signing, happens after `milestone_block_1`. This poses no threat in our opinion, but can be optimized when we study the dynamical model of our protocol and consider various kinds of delay that affect the increments to be out of sync with bitcoin blocks.
18. *User*s collectively start registering with the first *WT* on their agreed upon list. Once the registration is complete, *boomlet*s sync to make sure every *user* had registered with the aforementioned *WT*. The syncing is performed by passing *boomlet* signatures over *TOR* between *niso*s that will be checked inside other *boomlet*s for integrity.
19. Now each *boomlet* sends its pertinent *SAR*'s id to the *WT* to make sure the *WT* knows where to send the duress data and also make sure there is a connection between the two parties. *SAR* signs and encrypts a piece of data for *boomlet* to verify it has received the package from *WT*. And in this way the registration with *SAR* completes.
20. *Boomlet*s sync again with each other to make sure everyone has the protection of its pertinent *SAR*.
21. Now each *user* creates a back up for their *boomlet* named *boomletwo*.
22. Once the back up is done, *boomlet*s sync for the last time to conclude the setup procedure.

### Withdrawal overview

The core idea for withdrawal procedure is as follows. A more detailed description, alongside detailed sequence diagrams can be found in [withdrawal folder](withdrawal).

1. A psbt is created by the initiator *user* via their *niso*.
2. The psbt will be sent to the *boomlet* for it to get *user*'s approval via the *ST*. This is to minimize trust in *niso* which is not trusted in withdrawal ceremony.
3. Once the initiator *user* approves the transaction, *boomlet* encrypts the *psbt* for each *peer*'s *boomlet* via pertinent symmetric ECDH shared keys.*Boomlet* also signs an approval message and encrypts it for the *WT*.Then *boomlet* sends sends this data package to *WT* via *niso*.
4. *WT* checks the content of the approval message and if ok, sends the encrypted psbt to non-initiator *peer*s.
5. Each non-initiator *peer* receives the encrypted psbt and checks the content. Each *peer* (person) also checks the psbt via their *ST* and if they approve such transaction, they sign an approval message encrypted for the *WT* and for each *peer and send it back to the *WT*.
6. The *WT* receives these messages and checks their integrity and checks if the *peer*s have approved the psbt. If so, *WT* will send a collection of all other *peer*s approvals to each *peer*, alongside its own approval.
7. Each *peer* verifies the received package containing all other *peers* and the *WT*s approval in its *boomlet*. Also each *boomlet* checks if the distance between the block height this approval process began and the current block height reported by its connected *niso* is less than 1000, as a check for recency.
8. Now the *boomlet*s enter into a ceremony for committing to the transaction. The recency tolerance between initiation of this ceremony and each *boomlet* receiving commitment of all other *boomlet*s is set to be 10 blocks. Within this ceremony, the each *boomlet* performs a duress check as well to make sure none of the *user*s were in duress approving this transaction.
9. Before the *boomlet* enter into a committed state to the psbt, it will initiate a duress check.
10. Once all *peer*s commit to the transaction without duress, a digging game starts to increase `counter` in each *boomlet* to meet the `mystery`. Once all *boomlet*s reach the state of readiness, they can sign the committed transaction.
11. Each *peer*'s *boomlet* will sign a data structure named ping, containing the string "ping", the `tx_id` of the committed psbt, last seen block as reported by their *niso* and an increasing ping sequence number. Then the *boomlet* adds a duress placeholder to the end of this signed data structure, and signs the resulting data. Then the whole data will be encrypted for the *WT*. The duress placeholder itself is encrypted for the *SAR* and is not readable for the *WT*.
12. The resulting data is sent to the *WT*. *WT* will decrypt and decompose the message to send the encrypted duress placeholder to the *SAR*. *SAR* will check the contents and regardless of the duress result, signs the decrypted duress placeholder and encrypts the signature by the pertinent *boomlet*'s shared key and sends it back to the *WT*. Meanwhile, the *WT* verifies the first part of the message and checks if the signature is right and if the block height in the message is within the last 5 blocks as the *WT* sees the network.
13. *WT* waits to receive the data discussed in the last step from all *peer*s. Then it will create pong. Pong consists of all other *peer*s' pings, their pertinent signature on it, the block height as observed by the *WT* and a string "pong". *WT* will sign this pong data structure and encrypt it for each *boomlet*. Then the *WT* will add the duress placeholder that was signed by the *SAR* and encrypted for *boomlet* to the end of this data and send the whole thing to the *boomlet*.
14. Then *Boomlet* checks the integrity of the pong and if verified, it will increase its internal counter by 1 and produce another ping for the *WT*.
15. Whenever, based on an internal random number generator, *Boomlet* indicates that it's time for a duress check, it will create randomly sorted duress checks that will be presented to the *User*. Based on *User*'s answer to the duress check, the duress placeholder will be constructed and added to the pong.
16. The ping pong will continue until all *Boomlets*'counters reach to their pertinent mystery.
17. Once all *Boomlets* have reached the mystery, *WT* is informed and all peers can sign their PSBT via a collaboration between *Boomlet* and *Iso*.

### Design decisions

These decisions are not permanent and can be changed to modify the dynamic behavior of the system or strengthen protocol's security.

1. We have not considered reusability in terms of extending the boomerang period somehow yet.
2. There is a minimum and maximum range for the mystery that translate into the minimum and maximum number of steps for the digging game. These are subject to the requirements and circumstances of the peers. Given the milestones and the range for mystery, one can have an optimal limit for starting the roll over process to move funds to a new boomerang setup.
3. Withdrawal will not start before `milestone_block_0` in the current design. Although it could technically, since the digging game's end could well be after the aforementioned block.

## Duress protection mechanism

The core idea for our duress protection mechanism is as follows. A more detailed description can be found in [duress protection folder](duress_protection).

1. Boomlet has the set of all countries in memory.
2. At setup user selects 5 countries that signals their consent. All other combinations of countries are interpreted as duress. (5-dictionary model of *Jeremy Clark and Urs Hengartner. 2008. Panic passwords: authenticating under duress. In Proceedings of the 3rd conference on Hot topics in security (HOTSEC'08). USENIX Association, USA, Article 8, 1–6.*)
3. At duress check instances boomlet generates 5 random combination of numbers from 1 to 198. Those sets are exported to the device. The device will assign to each number, a country in a way that the number corresponds to the ascending alphabetical order of the country.
4. If the user is not in duress, they will search each list and find the consent countries in no particular order.
5. If the user is in duress, they will choose any other countries.
6. If the user is tortured, any combination of countries can be confessed to as the consent signal.
7. If the attacker is to choose a country at random, it has a chance of `(1/195 * 1/194 * 1/193 * 1/192 * 1/191 = 1/267,749,239,680 = 3.73e-12` to hit the consent countries.

## Security model

The threat model is at its final stages and will be shared soon.

## Proof-of-concept

We have coded a proof-of-concept implementation in rust that can be reached at [boomerang repo](https://github.com/bitryonix/boomerang).

## Concerns

1. Java card's ability to handel our expectations. Should it prove inviable, we have to spread some calculations to other entities and use the java card as the verifier of some final result made by a model of distributed partial trusts.
2. We have come to the conclusion that we cannot close the forced determinism attack vector completely.

## Roadmap

- [ ] Creating the dynamical model for studying the expected behavior of the protocol, subject to nonlinearity and delays.
- [ ] Designing ancillaries.
- [ ] Designing error handling.
- [ ] Designing multiple WTs voting.

## Call for collaboration

We welcome any comments or contributions on design and implementation.

## Financial support

We need financial support to expand our team to further th design. We are planning to apply for grants, should the idea worth it's dime.

If you want to support us, you can email us at <bitryonix@proton.me>.

## Updates

None for now.
