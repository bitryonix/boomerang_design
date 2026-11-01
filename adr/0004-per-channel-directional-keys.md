# ADR 0004: Use Per-Channel Directional Keys

- **Status:** Accepted
- **Recorded:** 2026-06-20

## Context

Boomerang previously described `channel_keys` as deriving one encryption/MAC
key pair from ECDH plus a message-specific context string such as
`"Boomerang/setup/duress_challenge"`. That gave strong message separation, but
it made Boomlet repeat ECDH/KDF work for every distinct message label even when
the same two endpoints were communicating over the same setup relationship.

Boomlet is intended to run on constrained Java Card hardware. Public-key
operations and repeated CMAC-KDF blocks are material load. The protocol already
authenticates message type, protocol version, sender, receiver, setup or
transaction binding, nonces, and phase information in the AES-CBC/CMAC envelope
context. That context can continue to provide message separation without using
the message label as a channel-KDF input.

The main security requirement is that simplifying the key schedule must not
reintroduce direction confusion, reflection, cross-protocol key reuse, or
host-controlled endpoint ordering.

## Decision

`channel_keys` derives one four-key schedule per ordered endpoint pair:

- endpoint 0 to endpoint 1 encryption key;
- endpoint 0 to endpoint 1 MAC key;
- endpoint 1 to endpoint 0 encryption key;
- endpoint 1 to endpoint 0 MAC key.

`channel_keys` takes the local entity name and peer entity name directly,
alongside the sender and receiver public keys used to select direction. Each
`entity_name` is a hardcoded Boomerang protocol enum. It is not accepted as an
unvalidated host string. Distinct endpoint names are ordered by bytewise ASCII
order of the fixed enum labels, and endpoints with the same name are ordered by
canonical public-key bytes.

The KDF label is `"Boomerang/channel_keys/v1"`. Its context binds
`PROTOCOL_VERSION` and the two canonically ordered endpoint names and identity
public keys. The message label is no longer a KDF input.

For each envelope, the sender and receiver select the directional key pair from
the channel schedule. Message type, protocol version, sender, receiver,
setup/transaction binding, nonce, and phase remain in the authenticated
`cbc_cmac_encrypt` / `cbc_cmac_decrypt` context.

This ADR refines ADR 0002's key-derivation discussion. ADR 0002 remains the
accepted decision for AES-256-CBC, PKCS#7 padding, AES-CMAC, encrypt-then-MAC,
and SP 800-108 AES-CMAC as the KDF construction.

## Rationale

The new schedule keeps independent encryption and MAC keys and keeps the two
directions cryptographically separate. A reflected envelope uses the wrong
directional MAC key and fails authentication. A replayed envelope under another
message type, phase, setup ID, or transaction ID fails because those values are
still authenticated by the envelope CMAC context.

Moving message labels out of the channel KDF reduces repeated KDF work and
allows one ECDH result to support all messages on that endpoint pair during the
active exchange. The KDF expands to 128 bytes once per channel schedule instead
of deriving a separate 64-byte schedule per message label and direction.

Ordering endpoints by hardcoded entity name makes derivation deterministic
without trusting Niso or another host to provide roles. Public-key-byte ordering
is retained only as the same-entity tiebreaker; backup uses the distinct
`boomlet` and `boomletwo` entity names shown in the setup diagram.

## Consequences

Implementations must validate the entity names and public keys before key
derivation. One endpoint must match the local entity name and identity public
key, and the other must match the peer entity name and public key used for
ECDH.

Envelope contexts become the mandatory location for message separation. Tests
must reject wrong-direction CMACs, reflected envelopes, message-type replay,
setup-bound replay under another setup ID, and derivation using a wrong entity
name.

Boomlet may cache the active channel schedule only in transient memory and must
erase it when the active exchange completes, stalls, or is explicitly aborted
unless a later ADR explicitly approves persistent channel-key state.

Historical diagrams or ADR text that refers to message-specific channel keys
must be read through this ADR and the current specification.

## Rejected Alternatives

- **One encryption key and one MAC key for both directions:** rejected because
  it weakens direction separation and makes reflection mistakes harder to
  detect.
- **One shared MAC key plus directional encryption keys:** rejected because the
  MAC is the envelope authenticator and should carry direction separation
  independently of encryption.
- **Host-provided endpoint ordering:** rejected because a malicious Niso or
  relay could try to cause peers to derive incompatible or role-confused keys.
- **Removing message labels from envelope context as well:** rejected because
  the channel schedule is intentionally shared across message types; envelope
  context is what prevents cross-message replay.
