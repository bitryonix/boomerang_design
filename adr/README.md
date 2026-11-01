# Architecture Decision Records

This directory records protocol decisions whose rationale should survive diagram
and specification revisions.

`spec/SPEC.md` is the normative protocol specification. ADRs explain why a
decision was made, its trade-offs, and its implementation consequences; they do
not override the specification.

## Accepted Decisions

| ADR | Decision |
| --- | --- |
| [0001](0001-setup-replay-and-phase-checkpoints.md) | Derive one setup instance ID and use one compact chained setup checkpoint. |
| [0002](0002-java-card-cryptographic-profile.md) | Use AES-256-CBC/PKCS#7 with AES-CMAC and an AES-CMAC KDF. |
| [0003](0003-single-sar-per-peer.md) | Use exactly one setup-bound SAR per peer. |
| [0004](0004-per-channel-directional-keys.md) | Use one four-key directional schedule per endpoint pair. |
| [0005](0005-user-chosen-doxing-password.md) | Use a user-chosen doxing password. |
| [0006](0006-per-withdrawal-mystery-generation.md) | Generate mystery per withdrawal at digging entry. |
| [0007](0007-niso-prevalidation-and-boomlet-revalidation.md) | Prevalidate protocol data on Niso and independently revalidate signing conditions on Boomlet. |
| [0008](0008-distinct-fixed-byte-encodings.md) | Give common fixed-width cryptographic byte strings distinct canonical tags. |
| [0009](0009-niso-relayed-consent-rotation.md) | Relay consent rotation through Niso and authorize it with signed ST approval, keeping the mnemonic out of the ceremony. |
| [0010](0010-consent-counter-display.md) | Display the consent counter during new set entry, authenticated completion, and Boomletwo's first activation consent query. |

## Status Values

- `Proposed`: under active review and not yet normative.
- `Accepted`: reflected in the current protocol specification.
- `Superseded`: retained for history and linked to its replacement.
