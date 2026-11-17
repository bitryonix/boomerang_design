# Boomlet nonce challenge for backup authorization

Draft, not adopted. 2026-11-09.

Optional freshness hardening for [SPEC Section 13.10](../../spec/SPEC.md#1310-boomletwo-backup)
under the existing trusted setup assumptions.

1. When connected to Iso for backup, Boomlet generates
   `backup_request_nonce := random_bytes(32)`, retains it for the current setup,
   and sends it to Iso.
2. Iso adds `backup_request_nonce: bytes32` to the existing
   `BoomletBackupRequest`, alongside the Boomletwo key, and signs it with the
   normal private key under `"Boomerang/setup/backup_request"`.
3. Boomlet verifies the signature and requires the nonce to match its outstanding
   challenge for the current setup. It consumes the challenge when accepting
   the request, before exporting state.

On reconnection, Boomlet resends a pending nonce; if lost, it generates a fresh
one. A setup change, setup abandonment, or backup skip invalidates the nonce.

This adds one message, the nonce from Boomlet to Iso.
