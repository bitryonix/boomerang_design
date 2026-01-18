# Forced determinism

## Vector attacks review

### 1. Stealing at least 1 boomlet card

#### Description

If for any reason 1 card is lost, all the protocol is forced into the normal deterministic period for withdrawal ceremony.

#### Potential guards

Have a backup boomlet.

#### Vulnerability

The backup may get faulty or lost or broken as well.

#### Justification

Loosing two cards is on average less likely compared to loosing two cards, given the holder comprehends that the backup needs a more secure safe keeping procedure.

### 2. Starting the withdrawal ceremony too late

#### Description

Since the peers don't know about the mystery in their cards exactly (as it is generated randomly within limits indicated by boomerang params) and consequently don't know till when is a safe time for them to start the withdrawal ceremony, it is generally a risk.
Limiting the range in protocol, leads to adversary knowing the effective starting time of the normal period. Users can choose this on their own, but this does not alleviate the risk.

#### Potential guards

- A flush mechanism.
- Setting the start of normal period long enough and rely on the user to start the roll-over process in a timely manner. There can be notifications by Niso as well.

#### Vulnerability

- Flush mechanisms cannot be static, for they will lead to leaving the problem of the last resort unsolved.
- People can be sloppy and inattentive to the requirement or the notification. 

#### Justification

- 

### 3. Non-cooperation of one of the peers

#### Description

If one of the peers stop cooperating in the withdrawal ceremony, the whole setting must wait for normal period.

#### Guard

- We are assuming users to be in accord with each other, like a 5 of 5 multisig assumptions. If we are to add a 4 of 5 to the boomerang period, we are loosing commitment to the protocol, as 4 users can deceive the other one of their commitment to the protocol and honesty of a single user will not guarantee the promises of the protocol to be delivered.

#### Vulnerability

- Remains.

#### Justification

-

## On solution

### Rough ideas

1. Flushing the funds into another boomerang instantly, via a pre signed transaction. [problematic, what happens to the last boomerang?]
    Does not work, for it just kicks the can down the road and does not solve the problem.
2. Involving a third party for fast transfer in boomerang era.
    if we involve a third party, the transaction cannot be presigned, for the third party may publish it at its whim. On the other side, where will be the destination? Another boomerang setup maybe. How that second boomlet is going to be protected without the possibility of being sidestepped? Maybe a state requirement. Nevertheless, involving a third party without a cryptographic proof that it will behave in a certain way, will be problematic.
3. Setting the boomerang era long enough and setting the random range for ceremony generation tight enough to ensure that in at least x% of the boomerang period, withdrawal ceremony will not overlap with normal period.
     Seems like the only viable option at the time.

### Solution for now

- Setting the boomerang period long enough compared to the range of uncertainty period.
- Notifying the users of the best time to do a roll-over transaction into another boomerang setup.

#### Thing to be considered

- Involving a third party that is cryptographically committed to perform as expected, once an expected function is designed to do a transition into another setup without posing a threat in and of itself.

---

## Comments and Discussions

### Comment 1

As of now, peers with money in a Boomerang address need to renew their setup before Boomerang becomes deterministic, which is a usability issue and if left unattended, an attack vector (item 2 of FD attacks). Using a mechanism similar to Ajolote, we can eliminate the need for renewal when withdrawal is not triggered. A simplified view of this mechanism is as follows:

user's UTXO -----(Setup TX)-----> Boomerang Vaulted UTXO -----(Withdrawal TX)-----> Boomerang Unvaulted UTXO -----(either nondeterministic TX using Boomlet keys or relatively-time-locked deterministic TX using normal keys)-----> Boomerang Spend UTXO

The drawback of this setup is that there would not be a single static Boomerang address: Each time we want to deposit money to Boomerang, we need to generate corresponding Boomerang TXs all over again. This effectively renders Boomerang ineffective as a fallback option to Ajolote. A partial mitigation to this problem is to mark Setup TX as ANYONECANPAY, and pre-make it with a predetermined output amount (e.g. 1 BTC). This gives us a Boomerang address that acts as a Boomerang storage with exactly 1 BTC capacity, which can then be used as a fallback address to Ajolote.

Note: This mechanism has no effect on determinism attacks during withdrawal process (e.g. an attacker breaking a peer's Boomlet device).

### Comment 2

>
>
> As of now, peers with money in a Boomerang address need to renew their setup before Boomerang becomes deterministic, which is a usability issue and if left unattended, an attack vector (item 2 of FD attacks). Using a mechanism similar to Ajolote, we can eliminate the need for renewal when withdrawal is not triggered. A simplified view of this mechanism is as follows:
>
> user's UTXO -----(Setup TX)-----> Boomerang Vaulted UTXO -----(Withdrawal TX)-----> Boomerang Unvaulted UTXO -----(either nondeterministic TX using Boomlet keys or relatively-time-locked deterministic TX using normal keys)-----> Boomerang Spend UTXO
>
> The drawback of this setup is that there would not be a single static Boomerang address: Each time we want to deposit money to Boomerang, we need to generate corresponding Boomerang TXs all over again. This effectively renders Boomerang ineffective as a fallback option to Ajolote. A partial mitigation to this problem is to mark Setup TX as ANYONECANPAY, and pre-make it with a predetermined output amount (e.g. 1 BTC). This gives us a Boomerang address that acts as a Boomerang storage with exactly 1 BTC capacity, which can then be used as a fallback address to Ajolote.
>
> Note: This mechanism has no effect on determinism attacks during withdrawal process (e.g. an attacker breaking a peer's Boomlet device).
>

Using a deleted will be like this:

1. We create a descriptor in which the boom leaf is constant. We can keep the normal leafs derived from some master normal key. The vault leaf will be generated at the time of disposal.
2. Having a gate address seems logical as it reliefs us from dealing with outsiders and gives us more control,
3. We can put the vault leaf up in the tree, so that it won't affect inclusion proofs of the other leafs down the tree.
4. Each disposal will generate a new address for the enforcement key (those that are to be deleted) must be regenerated.
5. Hence, for withdrawal, boomlet needs to sign a different inclusion proof for each address. That makes boomlet either cognizant or reliant on another entity to say what the control blocks are.
These can be resolved one way or another, I think.

But the thing is, we logically must put another boomerang address at the only receiving end of the unvault transaction. The point with vault structure is that we have a presigned transaction that once released can move bitcoins to another address. But:

1. What if the adversary obtains those presigned transactions as well?
2. In whom are we trusting those?
3. How many nesting would be suitable? How deep?
4. What happens to the last boomerang?

My intuition is on the side of it not working as we expected.

### Comment 3

On enabling Boomlet to accelerate the counter when the boomerang period is close to be finished:

This shall not happen as it will create an attack surface for others to pretend the current block is close to the end of the boomerang period and make Boomlet into sign rather prematurely.

### Comment 4

To document my thoughts:

1. What are we trying to fend off? And why?

- Being forced by an external entity into waiting till the normal period begins.

- Because:

    1. The adversary might have compromised our normal keys and is waiting out for those keys to come into effect.
    2. Our peers are waiting us out so they can move the funds without our consent in normal period.

### Comment 5

What if we get rid of the normal period?

The problem will change face into backing up the Boomlet data in a way that does not betray its purpose as an enforcement device. Even after solving that issue, anything digital will be susceptible to an EMP and relying solely on digital data would be problematic.

### Comment 6

Using attestation services

Downsides:

1. Attestation server can be subject to a DoS attack, preventing prompt response, if the reaction time frame is limited.

### Comment 7

One is on the side of relying solely on digital data.

Downsides:

1. Digital data can be deleted.
2. Digital data is prone to bit rot or malfunctions that make retrieval impossible.
3. Relying solely on digital data severs users connection with bitcoin protocol. That's why lightning is not used for significant sums of bitcoin or why nobody uses Ajolote or deleted key covenants. We are designing a cold storage protocol that must not be disconnected from bitcoin's base layer security on the basic level.

### Comment 8

cryptographic puzzle may prove to be useful here. However I'm fairly sure there are limitations on card processing power that make it impossible to create such puzzle.
cryptographic puzzle make use of un-parallelizable computations. we can have card export a backup file that's a cryptographic puzzle that takes as long as mystery number of blocks to extract. this backup file can either be the boom key or the withdrawal tx(with changes to withdrawal procedure).

### Comment 9

> Using attestation services
>
> Downsides:
>
> 1. Attestation server can be subject to a DoS attack, preventing prompt response, if the reaction time frame is limited.

more details: there's attestation that the code run in secure element in the server is correct and the pubkey is generated there. It can then be included in the spending conditions such that it only signs transactions that send from one boomerang setup to another only with the same keys involved.

maybe boomlet can do the same if it's able to look at the transaction outputs.

### Comment 10

It seems undoable. How should that attestation thingy know beforehand that what boomlet key is going to be generated? If you are referring to the same exact keys, this would be a reuse. Boomlet must be in the know of this possibility and the requirements, otherwise such utxo will be unspendable in the boomerang regime.
