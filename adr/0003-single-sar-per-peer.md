# ADR 0003: Use One SAR Per Peer

- **Status:** Accepted
- **Recorded:** 2026-06-16

## Context

Boomerang uses Search and Rescue (SAR) to hold encrypted rescue data and to
acknowledge duress placeholders during setup and withdrawal. Earlier setup
text still described a collection of SAR identities and WT-side SAR selection,
even though the security model already assumed exactly one SAR per peer.

Multi-SAR setup leaves important policy undefined: selection, substitution,
quorum, failover timing, and blame when one service is unavailable. Those
choices affect user safety and physical-response modelling, not only message
routing.

## Decision

Each peer has exactly one setup-bound SAR identity.

Phone, Iso, and Boomlet receive a single `SarId`. Boomlet derives one
`doxing_key_for_sar`, one `doxing_data_identifier`, one SAR identifier envelope,
and one signed `{setup_instance_id, sar_id}` binding for WT. WT verifies that
binding and forwards the SAR envelope only to that SAR. WT does not select
among SAR candidates and does not substitute a different SAR during the setup
or withdrawal path.

Withdrawal routes each peer's duress placeholder to the SAR identity bound
during setup. SAR unavailability is an explicit service failure; it is not
hidden behind an underspecified in-protocol fallback.

## Rationale

One SAR per peer gives the security model a single source of rescue for that
peer. That makes physical-security reasoning clearer: the protocol has one
service that can identify the user, store that user's rescue data, and begin an
external rescue response.

The decision avoids adding more single points of failure through multi-SAR
policy. A protocol that requires selection, quorum, or failover can be blocked
or confused by each additional required service or by disagreement about which
service is authoritative. It also excludes in-place SAR replacement. A new SAR
changes the key derivation, identifier, WT binding, and service receipts, so it
requires a fresh setup.

The decision also limits Boomlet load. Boomlet performs one SAR-key derivation,
one identifier derivation, one SAR envelope construction, one WT-bound SAR
identity signature, and one SAR response verification path. It does not loop
over candidate SARs, retain a SAR list for selection, or hash multi-SAR state
into setup checkpoints.

## Consequences

`SarId` remains the SAR identity type. No SAR collection, SAR quorum object, or
SAR selection policy is part of the active protocol.

`BoomletBackupState.selected_sar_id` remains the canonical stored SAR field.
The stored value identifies the one SAR that can acknowledge safe or duress
placeholders for that peer.

The SAR cannot be replaced within an existing setup. A peer that chooses a
different SAR must complete a fresh setup and move the funds to it. SAR
unavailability stalls the Boomerang path under the existing setup. Operator
response and rollover timing for prolonged unavailability remain open work.
