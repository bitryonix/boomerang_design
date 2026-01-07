# Secure Terminal (ST)

## Inception

The idea of a Secure Terminal (ST) was formed during duress design, to omit the need for user to go into the isolated environment to enter their duress answer and to eliminate the risk of Niso messing up with the duress signaling as we deemed Niso untrusted in withdrawal process.

Later, given the possibilities provided by such direct line on communication between the boomlet as the trusted entity and the user, the idea morphed into Secure Terminal (ST) that can be used to minimize trust in other entities specially when and if they act as mediators between boomlet and the user.

This document is to clarify the expectations and design requirements from ST.

### Operational expectations from the ST in duress

1. Must have a persistent memory to keep the `boomlet_identity_pubkey` and its own `st_identity_privkey`
2. Must have a monitor large enough for user to see clearly and small enough to be covered easily to show the duress question.
3. Must have a camera to scan QR codes as its means of communication to stay air gapped.
4. Must have a joystick for easy maneuvering and selection by user.

#### Trust assumptions in duress

1. ST can generate secure keys.
2. ST can securely store its keys.
3. ST can securely execute cryptographic operations.
4. ST's monitor keeps the integrity of the input data intact.
5. ST's joystick reflects user's true intentions.
6. ST is air-gapped and does not communicate its data with outside world except via the QR code.
7. ST's keys are not extractable.
8. ST does not accept new code to be run on itself.
9. ST does not run arbitrary code that is able to highjack decrypted data or manipulate the answers by user.

#### Capabilities

Based on the aforementioned expectations, one can assume these capabilities being built in the ST:

1. Key management capability (creating its own keypair and a shared key with Boomlet).
2. Encryption/decryption capabilities.
3. Air-gapped communication with Boomlet via sending and receiving QR codes through its monitor and camera (This means no direct communication with Boomlet which necessitate an intermediary. The intermediary must be trusted in key exchange and can be untrusted afterwards. ).
4. Visual channel to user.
5. Selection input from user.

#### Tamper resistance

Why are we using the ST?

Niso is not trusted in withdrawal and as so, can hijack the consent set --> No duress protection.
We need a trusted communication device then between the user and boomlet to make sure that the integrity of the duress check remains intact. I don't see a trustless solution to this, hence the trusted ST.

What does trusted mean?

* Does not give out keys.
* Does not accept new keys.
* Does not alter user choices.
* Does not alter messages received from Boomlet.

In other words, can securely generate and store keys and only executes the program we have put in it.

Now, what is tamper?

An intentional but unauthorized act resulting in the modification of a system, components of systems, its intended behavior, or data.

What are the results of tampering ST?

* Inject/Extract the private key into/from it --> Attacker can read the communications between ST and Boomlet --> Attacker will obtain the consent set.
* Extract the private shared key between ST and Boomlet --> Same as above.
* Meddle the program running on ST --> Accidental duress signal / [coupled with key extraction ] No duress signal no matter the user trying.

Hence, it seems to me that the device must be tamper proof to some extent.

Regarding tamper evidency, I think it depends on what aspect of tampering is to be evident. For example changing an SD card? Or trying to compromise the key store?

#### Hardware requirements

1. CPU for cryptographic operations.
2. A persistent memory for key management.
3. A Monitor to communicate with user.
4. A joystick to get selection inputs from user.
5. A camera to get inputs in an air-gapped manner.
6. A battery to supply power.
7. Must be tamper resistant.

#### Package

##### Common

1. Case with opening for power.
2. Tamper evident tape with serial number.

##### Raspberry Pi

1. Raspberry Pi Zero.
2. Raspberry Pi Zero battery.
3. Zimkey.
4. Internal Battery for Zimkey.
5. Waveshare Shield (Monitor and Joystick).
6. Camera module.
7. SD Card.

##### Arduino

1. Portenta H7 Lite (ABX00045) with secure element and without wireless connectivity.
2. Portenta Breakout with battery slot, camera and display ports.
3. OV2640 camera module.
4. 2.0 Inch 240x320 Full Color TFT LCD Display Module with ST7789 Controller, SPI/Parallel Interface for arduino. [https://www.aliexpress.us/item/3256808604687621.html?spm=a2g0o.productlist.main.18.61645876jv7OB0&algo_pvid=69f13fc9-05bc-490d-a6c3-783fb2ceb805&algo_exp_id=69f13fc9-05bc-490d-a6c3-783fb2ceb805-17&pdp_ext_f=%7B%22order%22%3A%22145%22%2C%22eval%22%3A%221%22%2C%22fromPage%22%3A%22search%22%7D&pdp_npi=6%40dis%21USD%213.90%210.99%21%21%213.90%210.99%21%40211b655217632056866594899ed64a%2112000051614741164%21sea%21US%210%21ABX%211%210%21n_tag%3A-29910%3Bd%3Abe8dfe73%3Bm03_new_user%3A-29895%3BpisId%3A5000000187461913&curPageLogUid=zy82F5O1A9ja&utparam-url=scene%3Asearch%7Cquery_from%3A%7Cx_object_id%3A1005008791002373%7C_p_origin_prod%3A]
5. Adafruit analog joystick breakout.


https://store.arduino.cc/collections/portenta-family/products/portenta-h7-lite

https://store.arduino.cc/collections/portenta-family/products/portenta-breakout

https://www.waveshare.com/ov2640-camera-board.htm

https://www.amazon.com/240x320-Display-Controller-Parallel-Interface/dp/B0FBGH21S9

https://www.adafruit.com/product/512