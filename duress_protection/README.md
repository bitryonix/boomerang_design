# Duress Mechanism

Duress checks and signaling initiates and takes place in the boomlet as the trusted hardware component in our protocol. User interface to boomlet is the Secure Terminal or ST that is designed to minimize side channel attacks surface against the duress mechanism in a non-isolated environment.

## Duress actors

1. **The User**: Is the person of interest that creates a doxing data package and chooses a doxing password and signs up with the SAR privately.
2. **The Search And Rescue entity or SAR**: Is the trusted entity that once received the duress signal, can decrypt doxing data and start the search and rescue operation for the user in duress.
3. **The Adversary**: Is an entity with abilities described later in this document.

## Doxing Data

Doxing data is used to retrieve the user in case of an attack. User retrieval is done by the SAR.

Doxing data consists of two parts:

1. **Static doxing data**:

    ```rust
    StaticDoxingData {
        name: String,
        national_id: String, 
        address_home: String,
        address_work: String,
        phone_number_mobile: String,
        phone_number_home: String,
        phone_number_work: String,
        trusted_person_name: String,
        trusted_person_address: String,
        trusted_person_phone_number: String,
    }
    ```

2. **Dynamic doxing data**: This a live feed of choice that helps indicating user's last known state. Like a location feed provided by user's mobile phone.

All doxing data are encrypted by `doxing_key` which is the SHA256 digest of the `doxing_password`. Static doxing data is indicated at setup stage and sent to SAR by user as part of the registration process with the SAR. Dynamic doxing data is encrypted at source by the `doxing_key` and continuously sent to SAR via the originating device.

Most communications with the SAR is done through the user's phone.

Once the positive duress signal is sent, boomlet sends a payload containing `doxing_key` encrypted for SAR. This is sent to WT via the next packet. WT sends it to SAR to receive an acknowledgement of reception.

In other cases (no positive duress signal issued), boomlet encrypts an all zeros data in the same manner and puts it in the duress signal bearing part of each message. SAR always decrypts the received data with its keys. If the decrypted data turns out to be all zeros, there is no duress. If the decrypted data turns out to be anything else, SAR will hash the resulting data (`doxing_key`) to create `doxing_identifier`. It then searches its database for a registered identifier as the `doxing_identifier` and decrypts the doxing data with the `doxing_key` to use them in the search and rescue operation.

## Duress coverage

Duress mechanism is to cover the following adverse scenarios:

1. Being forced to initiate and commit to a transaction.
2. Being forced to continue the signing process when informed that the target address has been compromised.

Duress mechanism does not cover setup stage.

## Expectations and evaluation criteria

1. The duress payload should be easily producible within the confines of the java card, like a 256bit number.
2. The duress payload must lend itself to a particular pattern that we name duress pattern (plausibly deniable).
3. The duress signal/pattern should be easily remembered.
4. Given any pattern, it is required that in any duress check instance, the user can signal being in duress.
5. Given any pattern, it is required that in any duress check instance, the user can signal not being in duress.
6. The duress check must be resistant to replay attacks.
7. The duress check must be resistant to typos or miscalculations by the user.
8. If the attacker is to guess at random, let's say it has a chance of `P(ag)` to pass the duress check successfully.
9. If the attacker is to torture the user, ideally, the attacker must know that the user has a big enough set to choose from and every choice will be consistent with user's  and system's behavior in non-duress condition, in a way that adhering to those may cause duress signal with a probability of `P(ac)`. This should be high enough to prevent the attacker from initiating an attack under our assumptions.

## Attacker characteristics

1. Attacker **can** capture the user and does not leave.
2. Attacker **can** observe the user in captive.
3. Attacker **can** read plain messages sent by the user.
4. Attacker **can** act instead of the user.
5. Attacker **can** commit vandalism (physically break the card, destroy the computer containing niso, etc.).
6. Attacker **cannot** break the java card security promises.
7. Attacker **cannot** break cryptography.

## Duress signal consequences

After emitting duress signal the following happens:

1. No change in observable messages will happen. (this may cause an issue of assurance over sending the duress signal and surprise by SAR after some error made by user in the mechanism).
2. A duress bearing payload that exist in every message omitted by boomlet, will be replaced by the doxing key encrypted by boomlet and SAR keys (DH / AES).
3. The pre-determined, unknown duration of withdrawal ceremony that is bound by the largest mystery generated in boomlets, will not change.
4. SAR will be informed and will act on information at hand to search for the user and rescue them.

## Solution

### Assumptions

1. Duress checks are shown on a secure device with a qr code scanner, a small monitor for establishing visual channel with user and showing a qr code and a joystick for input.The monitor is sufficiently small for user to see and be able to protect its content in a non-isolated environment easily. We call this ***ST*** for Secure Terminal.
2. The attacker cannot observe user's hand when using the device (ST) or cannot confer meaningful information without seeing the hand and the monitor at the same time.

### Proposal

#### Core

1. Boomlet has the set of all countries in memory.
2. At setup user selects 5 countries (without order and without repetition) that signals their consent. All other combinations of countries are interpreted as duress. (5-dictionary model of *Jeremy Clark and Urs Hengartner. 2008. Panic passwords: authenticating under duress. In Proceedings of the 3rd conference on Hot topics in security (HOTSEC'08). USENIX Association, USA, Article 8, 1–6.*)
3. At duress check instances boomlet generates 5 random combination of numbers from 1 to 195. Those sets are exported to the ST. The ST will assign to each number, a country in a way that the number corresponds to the ascending alphabetical order of the country.
4. If the user is not in duress, they will search each list and find the consent countries in no particular order.
5. If the user is in duress, they will choose any other countries.
6. If the user is tortured, any combination of countries can be confessed to as the consent signal.
7. If the attacker is to choose a country at random, it has a chance of `(1/195 * 1/194 * 1/193 * 1/192 * 1/191 = 1/267,749,239,680 = 3.73e-12` to hit the consent countries.

#### Ceremony

Duress check is initiated by the boomlet at the commitment to the withdrawal transaction step and at random instances.

If we use the latest bitcoin block hash as a Verifiable Random Function for this purpose, we may fall victim to false block hashes attack, since the boomlet is not in the know about the real bitcoin blockchain. Nevertheless, if we choose this setting, duress checks will occur simultaneously for all peers and can alleviate the requirements for addressing the asynchronous delay caused by inputting duress signal by peers.

On the other hand we can use consequent hashing as the VRF. This can be done based on an agreed upon and shared piece of data. Like a hash of the descriptor or a hash of all `boomlet_identity_pubkey`s. This makes duress check intervals predictable.

A third option will be the PRNG inside boomlet, which makes the duress check instances unpredictable for any entity outside boomlet. This will cause issues though if we do the duress checks too frequently as people may not be present 24/7 to answer the checks and their geographic distance stipulates different time zones and prolonged delays as a result. Of course this is a thing for rather all methods stated here.

We chose the PRNG inside the boomlet for duress check intervals. Users will choose a number representing how many blocks apart they want their duress checks at average. Boomlet will use the PRNG output's mod with the said number as its internal timer for duress check. Whenever the mod is zero, boomlet will start a duress check during the ping pong or digging game period.

##### Setup

1. Boomlet and the ST will perform a key exchange via Iso and QR code in the setup. Hence, the boomlet will know the ST and vise versa.

    We may have one or more of these devices (ST) introduced to boomlet for backup purposes. For now we assume 1 ST.

2. Boomlet will send an encrypted randomly ordered list of numbers from 1 to 195 inclusive, via a QR code showed on Iso to ST.
3. ST will scan the QR code on Iso's monitor, decrypt the data and show the list of countries assigned to the list in an ascending alphabetical order.
4. User will choose 5 countries at will. Order is not important and repetition is not allowed.
5. ST will record the selection and encrypt the 5 word byte array with its shared key with boomlet.
6. The ST will show the resulting encrypted data in QR code format on its monitor.
7. Iso will scan the QR code and send it too boomlet.
8. Boomlet will decode and decrypt the QR code and record the set of numbers as `duress_consent_set`.

##### Withdrawal

1. Duress check is initiated by the boomlet, either at commitment to the withdrawal PSBT or at a random interval.
2. Boomlet will create 5 random combination of numbers of the complete `[1:195]` set. That is 5 sets consisting of numbers between 1 and 195 (inclusive) that are ordered randomly. Boomlet will save these sets in its memory.
3. Boomlet encrypts the set of these sets with the shared key with ST.
4. Boomlet will export this encrypted data to niso via QR code.
5. Niso will encode it into a QR code and displays it on its monitor, prompting the user to scan the QR code with ST.
6. User will scan the QR code with ST.
7. ST will decrypt the code and extract the 5 sets from the decrypted data.
8. ST will show to the user 5 columns on its monitor, each a list of countries assigned to the number in those initial random lists based on their ascending alphabetical order.
9. The user will use the D-pad to select a country for each column.
10. ST will translate the selection into an array of 5 bytes, each byte representing the index of the selected country in the pertinent set.
11. ST will encrypt the array with its shared key with boomlet.
12. ST will encode the encrypted data into a QR code.
13. Niso will scan this QR code with its camera.
14. Niso will decode the QR code into a byte array and send it to boomlet.
15. Boomlet will decrypt the message with its shared key with ST and extract the numbers that were present in the communicated indices.
16. Boomlet will check if the extracted numbers form a set equal to the `duress_consent_set`.
17. If:

    ##### 17.1 the user has entered a duress signal by entering any set other that `duress_consent_set`

    17.1.1. Boomlet will encrypt `doxing_key` with SAR keys.
    17.1.2. Boomlet will put the encrypted result in the duress bearing part of the next message.
    17.1.3. Boomlet will send the message to WTs.
    17.1.4. WTs will decompose the message and send a message to SARs containing the duress bearing part of the message.
    17.1.5. SARs will receive the message and try to decrypt it with their private keys.
    17.1.6. SARs will hash the resulted data and search if any of their customers have the resulting digest as their identifier.
    17.1.7. If there is a match, the SAR knows its user is in duress and goes into the operational procedure of retrieval by decrypting static and dynamic doxing data by the `doxing_key`.

    ##### 17.2. the user has not input a duress

    17.2.1. Boomlet encrypts an all zeros data with with SAR keys (there is iv, hence the result will not be the same at each instance).
    17.2.2. Boomlet will put the encrypted result in the duress bearing part of the next message.
    17.2.3. Boomlet will send the message to WTs via Niso.
    17.2.4. WTs will decompose the message and send a message to SARs containing the duress bearing part of the message.
    17.2.5. SARs will receive the message and try to decrypt it with their private keys.
    17.2.6. SARs will see that the data is all zeros and ignore the message.
