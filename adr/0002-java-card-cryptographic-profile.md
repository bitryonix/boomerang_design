# ADR 0002: Java Card Cryptographic Profile

- **Status:** Accepted
- **Recorded:** 2026-06-07

## Context

Boomlet is intended to run on constrained Java Card devices. The symmetric
profile should support as many suitable card models as practical and minimize
code size, transient memory, persistent writes, primitive initialization, and
data transferred through the card interface.

An AEAD-only profile, especially AES-GCM, would exclude cards that provide AES
and CBC but lack a usable GCM API. Plain AES-CBC is broadly available but is
malleable and exposes padding-oracle risks unless separately authenticated.
Adding HMAC/HKDF would require another primitive family and additional key and
state handling.

## Decision

Authenticated encryption uses encrypt-then-MAC:

- AES-256-CBC;
- PKCS#7 padding;
- a fresh unpredictable 16-byte IV for each new encryption;
- AES-256-CMAC with the full 16-byte tag;
- independent 256-bit encryption and MAC keys;
- authentication of protocol context, IV, and ciphertext;
- constant-time CMAC comparison before any CBC decryption or padding check;
- one externally indistinguishable failure class for authentication, padding,
  and decoding errors.

ECDH output is not used directly. Distinct channel keys are derived with NIST
SP 800-108 counter mode using AES-256-CMAC. Domain-separated labels and
canonical context bind protocol version, message type, roles, identities,
`setup_instance_id`, and `tx_id` where available.

Java Card implementations may use `Signature.ALG_AES_CMAC_128`; the name's
`128` denotes the AES block and CMAC output size, not the key size. The profile
requires a 256-bit AES key. Cards without a native CMAC API may implement NIST
SP 800-38B using their AES engine, subject to side-channel and conformance
testing.

## Rationale

This profile concentrates Boomlet's symmetric work in AES:

- CBC provides encryption on a broad range of Java Card products.
- CMAC provides integrity and authentication using the same block cipher.
- SP 800-108 CMAC mode provides key separation without adding a hash-MAC
  implementation.
- A full tag avoids introducing a reduced forgery bound to save a small number
  of transmitted bytes.

The profile therefore favors card compatibility and implementation simplicity
while retaining authenticated-encryption properties through composition.

## Security Requirements

- Unauthenticated CBC is forbidden.
- MAC-then-encrypt and encrypt-and-MAC are forbidden.
- IV reuse with a different plaintext under the same encryption key is
  forbidden.
- Encryption and MAC keys must be distinct.
- Raw ECDH output must be erased after key derivation.
- Derived keys may be cached only in transient memory for the active exchange
  and must be erased when that exchange completes, stalls, or is explicitly
  aborted.
- Parsing and allocation limits must be checked before processing attacker-
  controlled lengths.
- Test vectors must cover context, IV, ciphertext, and tag modification and
  verify that padding is never examined before successful CMAC verification.

## Boomlet Impact

- Uses an AES engine for CBC, CMAC, and the CMAC-based KDF.
- A 64-byte key schedule requires four CMAC outputs.
- Each envelope requires one CMAC generation or verification in addition to CBC
  processing.
- The full tag adds 16 bytes per envelope.
- Streaming CBC and CMAC avoids buffering whole messages where the Java Card API
  permits it.
- Transient key caching can avoid repeating ECDH and KDF work within one active
  exchange without adding persistent secret state.

## Rejected Alternatives

- **AES-GCM or another mandatory AEAD:** rejected because support is less
  consistent across target Java Card generations and models.
- **Plain AES-CBC:** rejected because it provides no integrity and enables
  ciphertext malleability and padding-oracle attacks.
- **AES-CBC with truncated CMAC:** rejected because the small transmission saving
  is not worth reducing the forgery bound.
- **HMAC/HKDF plus AES-CBC:** rejected because it adds a second primitive family,
  more code paths, and more constrained-device state without a necessary
  protocol benefit.
- **Raw ECDH as a traffic-encryption or envelope-MAC key:** rejected because it
  lacks key separation and protocol-context binding. Raw ECDH is used only as
  the SP 800-108 CMAC KDF input key.
