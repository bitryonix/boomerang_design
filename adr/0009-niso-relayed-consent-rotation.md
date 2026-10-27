# ADR 0009. Niso-Relayed Consent Rotation With ST Approval

- **Status:** Accepted
- **Recorded:** 2026-10-07

## Context

Boomlet and its designated inactive backup Boomletwo store the five-country
consent set. Their paired trusted ST device supplies user input and approval.
Reconstructing the normal key for consent rotation adds a mnemonic-handling
occasion and brings spending components together for maintenance. Iso receives
mnemonic, passphrase, and normal-key material
([SPEC §7.2](../spec/SPEC.md#72-iso-state)).
The normal key participates in final signing
([SPEC §15.13](../spec/SPEC.md#1513-musig2-signing)) and authorizes fallback branches
([SPEC §11.2](../spec/SPEC.md#112-taproot-policy)).

The mnemonic and passphrase determine the seed, from which the private key tree
can be reconstructed ([BIP39](https://bips.dev/39/), [BIP32](https://bips.dev/32/)).
Handling them during rotation creates another opportunity for capture or
accidental retention, with consequences beyond consent maintenance.

## Decision

Use Niso as an untrusted relay and authorize rotation with explicit signed ST
approval of the exact nonce-bound current consent freshness token. Rotation
requires no mnemonic, passphrase, or normal private key.

Existing setup establishes the fixed pair through normal-key backup authorization,
authenticated state transfer, and signed `BackupDone`
([SPEC §13.10](../spec/SPEC.md#1310-boomletwo-backup)). Devices retain the setup and
paired identities. ST authenticates review through its existing paired Boomlet
channel.

Both devices independently verify the fixed pair and fresh ST approval.
Niso transports encrypted messages.

ST approval permits consent rotation only within that fixed pair. Device
replacement, rebinding, signing, and Boomletwo activation retain their separate
requirements.

## Rationale

Consent maintenance can preserve the separation between the normal key and
Boomlet until an operation requiring both occurs. The existing trusted ST
provides explicit review and authenticated approval without exposing spending
credentials.

## Consequences

ST approval establishes trusted intent but provides less independent ownership
authentication than possession of the normal key. Someone controlling the paired
trusted devices and ST can perform unauthorized maintenance, including forcing
duress or blocking maintenance. Approval also cannot prove freedom from
coercion.

Trusted devices enforce freshness, withdrawal binding, resource limits, durable
updates and acknowledgments, and SAR delivery despite a malicious Niso. Backup
provisioning copies Boomlet's identity private key, so trusted applet roles must
prevent the inactive backup from originating source messages. The previous-set
check accepts valid wrong answers, transferring the duress flag to the
backup without observable classification differences. Niso can interrupt
delivery; uncertain participants stay held.

## Rejected Alternatives

- **Iso plus fresh normal-key authorization:** adds an independent ownership
  credential but requires additional mnemonic access and spending-key handling
  for this maintenance operation.
- **Consent-set correctness as authorization:** the set may already be observed,
  and valid wrong answers must still complete rotation with withdrawal-scoped duress.
