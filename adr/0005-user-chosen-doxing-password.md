# ADR 0005: Use a User-Chosen Doxing Password

- **Status:** Accepted
- **Recorded:** 2026-07-09

## Context

Boomerang uses `doxing_password` to derive the key that protects SAR rescue
data. SAR stores rescue envelopes under a derived identifier, and rescue-data
confidentiality depends on the strength of the password the user supplied
during setup.

The strongest cryptographic option would be to require protocol-generated
high-entropy secret material, such as another mnemonic-length recovery phrase,
and derive `doxing_key` only from that material. That improves resistance to
offline guessing if SAR-held encrypted rescue data or related metadata is
exposed.

It also creates another secret the user must understand, record, protect, and
retrieve during backup or recovery ceremonies. Boomerang users already manage a
wallet mnemonic, passphrase, physical Boomlets, ST consent memory, SAR
selection, and withdrawal coordination. Adding another 12-word secret for
doxing data makes the ceremony harder to complete correctly and raises the
chance that a user loses or mishandles the rescue path.

## Decision

`doxing_password` is user-chosen. The protocol does not dictate a fixed entropy
target, require generated 12-word material, or reject passwords solely because
they fail a protocol-level strength threshold.

Implementations may help the user choose a better password with local UI
guidance, warnings, or optional generation, but those checks are advisory. The
normative protocol accepts the user's chosen `doxing_password` and derives:

```text
doxing_key =
  tagged_sha256(
    "Boomerang/doxing_key",
    utf8(doxing_password)
  )
```

Boomlet receives `doxing_key`, never `doxing_password`, and does not enforce
password policy.

## Rationale

This decision favors UX over the stronger security posture of mandatory
high-entropy doxing secrets.

SAR rescue data is valuable only if users can reliably keep the rescue path
usable. Forcing every user to preserve another mnemonic-length secret creates a
large operational burden for a secret that is separate from the wallet recovery
mnemonic. A user who loses or confuses that extra secret may make SAR-held
dynamic rescue data unrecoverable or unverifiable when it is most needed.

A user-chosen password keeps setup and backup closer to the mental model users
already have for account protection. It lets careful users choose or generate a
strong password, but it does not make the protocol dependent on every user
safely storing another 12 words.

## Security Effect

- Rescue-data confidentiality is bounded by the entropy and uniqueness of
  `doxing_password`.
- If SAR-stored encrypted rescue data is exposed, weak or reused passwords are
  more susceptible to offline guessing than generated high-entropy material.
- The derived `doxing_data_identifier` remains a lookup value, not a secret.
- This decision does not weaken Boomlet private keys, wallet mnemonic
  derivation, duress challenge validation, SAR routing, or withdrawal message
  authentication.
- Product profiles should make the risk visible to users and should encourage,
  but not require, a high-quality password.

## Consequences

The specification must continue to state that rescue-data confidentiality
depends on `doxing_password` entropy. Security reviews must treat weak
user-chosen passwords as an accepted product risk, not as a protocol violation.

Implementations should avoid sending `doxing_password` to Boomlet and should
avoid persisting it after deriving the needed setup and backup values. Password
quality meters, generated-password options, and warnings belong in Phone or Iso
UX, not in Boomlet protocol validation.

## Rejected Alternatives

- **Require another 12-word secret:** rejected because it overloads users with
  another mnemonic-length item to store, distinguish, and retrieve.
- **Mandate a protocol entropy threshold:** rejected because it converts a UX
  recommendation into a hard ceremony failure and makes recovery depend on a
  policy that Boomlet should not enforce.
