# ADR 0001: Setup Replay and Phase Checkpoints

- **Status:** Accepted
- **Recorded:** 2026-06-07

## Context

Setup needs to distinguish installation attempts, prove that all peers accepted
the same parameters, prevent phase skipping, and keep peer-local evidence out
of common agreement.

Redundant setup nonces, peer-local transcript values, duplicated signed
parameter objects, and peer-specific service receipts add hashing, parsing,
signatures, transmission, and persistent state on Boomlet without providing a
distinct replay guarantee.
Peer-specific receipts cannot be inputs to `setup_checkpoint` because honest
peers receive different receipts.

## Decision

Each Boomlet generates one fresh `peer_setup_nonce` and authenticates it
inside its signed `PeerSetupRecord`. Peers deterministically sort the signed
records by encoded Boomlet identity public key and construct
`boomerang_params_seed` from the ordered signed peer records, preference-ordered
`wt_ids_collection`, and `milestone_block_collection`. They derive:

```text
setup_instance_id =
  tagged_sha256(
    "Boomerang/setup_instance_id",
    canonical_encode(
      PROTOCOL_VERSION,
      boomerang_params_seed
    )
  )
```

`wt_ids_collection` preserves the user-approved service preference order. The
full structured context remains the preimage for `setup_instance_id`.
Boomlet sends ST a nonce-bound encrypted compact commitment during setup review:

```text
boomerang_params_seed_fingerprint =
  tagged_sha256(
    "Boomerang/boomerang_params_seed",
    canonical_encode(PROTOCOL_VERSION, boomerang_params_seed)
  )

params_seed_review_commitment =
  ParamsSeedReviewCommitment {
    setup_instance_id,
    boomerang_params_seed_fingerprint
  }
```

Niso forwards the ordered seed fields to ST beside the encrypted commitment.
ST canonical-encodes those fields, checks the resulting setup ID and
fingerprint against Boomlet's commitment, displays the matched ordered fields,
and signs the exact nonce-bound commitment after User approval.

Setup parameter agreement uses one signed 32-byte fingerprint:

```text
boomerang_params_fingerprint =
  tagged_sha256(
    "Boomerang/boomerang_params",
    canonical_encode(
      BoomerangParams {
        setup_instance_id,
        peer_ids_collection,
        wt_ids_collection,
        milestone_block_collection,
        boomerang_descriptor
      }
    )
  )

boomerang_params_fingerprint_signed_by_boomlet_i =
  sign_message(
    boomlet_i_identity_privkey,
    "Boomerang/setup/agreement",
    boomerang_params_fingerprint
  )
```

`boomerang_descriptor` is constructed from `peer_ids_collection` and
`milestone_block_collection`. Peers and WT accept agreement only after verifying
that every signed fingerprint has identical content and a valid signature under
the corresponding Boomlet identity key.

Common setup progress is the `setup_checkpoint` chain:

```text
setup_checkpoint =
  tagged_sha256(
    "Boomerang/setup_phase_checkpoint",
    canonical_encode(setup_instance_id, "parameters_agreed", zero_bytes_32)
  )

setup_checkpoint =
  tagged_sha256(
    "Boomerang/setup_phase_checkpoint",
    canonical_encode(setup_instance_id, "wt_ready", setup_checkpoint)
  )

setup_checkpoint =
  tagged_sha256(
    "Boomerang/setup_phase_checkpoint",
    canonical_encode(setup_instance_id, "sar_ready", setup_checkpoint)
  )

setup_checkpoint =
  tagged_sha256(
    "Boomerang/setup_phase_checkpoint",
    canonical_encode(setup_instance_id, "backup_ready", setup_checkpoint)
  )
```

A Boomlet signs `setup_checkpoint` only after successfully verifying its local
prerequisite for that phase. WT, SAR, and backup receipts remain local evidence
for signing. Boomlet persists `setup_checkpoint` as its local setup-completion
state.

Withdrawal binds the stored `setup_instance_id`; the final checkpoint is only
a local precondition that setup completed.

## Rationale

The setup ID gives every later message a compact binding to the exact setup
context. The phase chain proves a common order of locally verified milestones
without claiming that peer-local evidence is identical. Fresh challenge nonces
already provide replay protection for ST request-response exchanges, so adding
the setup nonce there is redundant.

## Security Effect

- Replayed setup material either derives an old setup ID or fails exact record
  inclusion, state, challenge, or replay-memory checks.
- Peers cannot reorder or skip common setup phases without deriving a different
  checkpoint.
- A checkpoint means every signer accepted its own prerequisite; it does not
  prove that all peers received byte-identical service receipts.
- Receipt validity still depends on local signature, setup ID, the
  `boomerang_params_fingerprint`, and object-specific checks.

## Boomlet Impact

- Keeps one 32-byte setup nonce in the signed setup record.
- Requires one setup-context hash and one small hash per setup phase.
- Reuses one signed `boomerang_params_fingerprint` for peer and WT agreement.
- Removes repeated setup-nonce parsing and comparison from ST exchanges.
- Removes rolling local transcript hashes and large receipt hashes.
- Removes redundant setup-ID and final-checkpoint fields from messages where
  the values are already authenticated or not yet available.

## Rejected Alternatives

- **Hash every setup message:** rejected because it adds secure-element work and
  makes peer-local transcripts unsuitable for common agreement.
- **Hash service receipts into checkpoints:** rejected because honest receipts
  differ by peer.
- **Use independent magic-value fingerprints:** rejected because they do not
  commit to phase order.
- **Transmit the final `setup_checkpoint` during withdrawal:** rejected because the
  setup ID supplies message binding and the checkpoint is only a local readiness
  condition.
