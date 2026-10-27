# Dynamic rescue-data lifecycle proposal

| Item | Value |
| --- | --- |
| Status | Adopted into the design specification, 2026-09-18; deployment evidence pending |
| Phase 1 priority | 1 |
| Proposal baseline | [`SPEC.md`](../../spec/SPEC.md) at `8ad77efabe03385d60733827fe55b3f988519c65` |
| Related gaps | DG-11, DG-40 |
| Normative integration | Current [`SPEC.md`](../../spec/SPEC.md) |

## Integration change map

Recorded 2026-09-18 before applying the integration; paths reflect the current
repository layout. The user requested adoption
of this proposal, a running changelog, and an incremental Phase 1 report.

| File | Places to change | Intended result |
| --- | --- | --- |
| `spec/SPEC.md` | Sections 7.5, 8.3, 9.4, 9.6, 10, 13.1, 16.4, 18.1–18.4, 19.4, 19.9, 20–22 | `dynamic_update_auth_key` and `PROTOCOL_VERSION` state; device KDF; authenticated append; exact receipts; bounded opaque payloads; registration, replacement, retry, and complete-history rescue access. |
| `wip/wire_catalog.json`, `wip/wire_catalog.txt` | Schema registry, signature domains, encryption contexts, registration tuple, generated layouts | Assign IDs 11 and 30 to upload and receipt; register exact types and regenerate the text catalog. |
| `outside/test_vectors/` | Focused vectors | Cryptographic bytes and lifecycle evidence with explicit boundaries. |
| `setup/derived/setup_development_contracts.md`, `setup/derived/setup_no_prose_guards.md`, `setup/README.md` | SAR registration, stored objects, receipt verification, retry | Protected SAR registration and signed first-upload receipt. |
| `setup/**/*.puml`, `withdrawal/**/*.puml`, `duress_protection/*.puml` | SAR upload and rescue-data retrieval fragments | Device-key encryption and asynchronous access to all retained uploads; source edits only. |
| `DESIGN.md`, `GLOSSARY.md`, `withdrawal/README.md`, `duress_protection/README.md` | Rescue data, registration, replacement, activation | Terminology and explanatory behavior consistent with the specification. |
| `security_models/security_model_update_note.md` | DG-11, DG-40, T-DATA-02, related risks and assumptions | Hand off affected security-model entries for independent reassessment. |
| `outside/Cannon_roadmap.md`, `outside/Cannon_phase_1_report.md`, `outside/CHANGELOG.md` | Dynamic-data milestone, progress, decisions, evidence, chronological entries | Adopt history retention and record progress and remaining gates as work proceeds. |
| This proposal | Status, current-status baseline, roadmap divergence, object limits, integration checklist | Record adoption and exact implementation choices without claiming unrun deployment tests. |

The independent review should also revisit the rescue-data attack-tree branch
and SAR data-flow labels. The [security review note](../../security_models/security_model_update_note.md)
lists the affected entries.

The catalog validation covers the schema registry and registered dynamic
exchange types.

## Current status

The specification defines authenticated append, uploads keyed by
`doxing_data_identifier`, device ID, and upload ID, exact signed receipts,
protected first registration, replacement, non-erasure, retry, and conflict rejection.
`DynamicRescueUpload` uses schema ID
11; `DynamicRescueReceipt` uses ID 30.

The [changelog](../../outside/CHANGELOG.md) records integration and validation. The
[Phase 1 report](../../outside/Cannon_phase_1_report.md) records evidence and open
gates. The existing DG-40 entry awaits independent reassessment. Protected
provisioning, real storage restoration, and duress-observability
validation remain open.

## Roadmap goal

Define authenticated append, complete-history retention, exact retries, and
conflict rejection. Corrections append without suppressing accepted evidence.
Payload meaning and freshness belong to the rescue application. Complete SAR
restoration through supported storage failures is a deployment assumption.

## Reason for divergence

The baseline milestone assumed one current record. Dynamic payloads can be a
stream, independent observations, or another collection in which late and
conflicting entries remain useful. Selecting one current record would add
payload semantics and let a malicious Phone make earlier evidence obsolete.

| Milestone objective | Proposal treatment |
| --- | --- |
| Prevent an older valid upload from replacing a newer one | Satisfied because no upload replaces another |
| Accept corrections | Satisfied by appending corrected data while retaining prior data |
| Reject rollback | Phone cannot erase accepted entries; SAR storage recovery is a deployment assumption |
| Handle duplicates and conflicts | Exact retries are idempotent; distinct uploads are retained; conflicting reuse of an upload ID is rejected |
| Identify one current update | Replaced by complete-history delivery to SAR implementation |
| Define a winning version | Unneeded because the protocol retains every upload |
| Define expiry and Phone clock skew | Delegated with freshness and relevance to SAR implementation |

A later payload has no protocol authority to suppress an earlier one. The
protocol remains independent of payload meaning while satisfying the
milestone's non-replacement, correction, rollback, duplicate, and conflict
goals.

The adopted roadmap uses authenticated append, complete-history retention,
and rollback detection. Deployment evidence remains a completion gate.

## Adopted changes

| Area | Adopted behavior |
| --- | --- |
| Payload | Bounded bytes interpreted by SAR's rescue application |
| Storage | Retain every accepted encrypted upload |
| Identity | Identify uploads by `doxing_data_identifier`, device ID, and random upload ID; use a per-device sequence number for ordering |
| Encryption | Derive a device-data key from `doxing_key_for_sar` |
| Authentication | Authenticate uploads with `dynamic_update_auth_key` |
| Acknowledgment | Return a signed receipt for the exact encrypted envelope |
| Registration | Extend the existing Phone and SAR setup exchange for the first device |
| Replacement | Reconstruct `dynamic_update_auth_key` and choose a fresh device ID |

The identifier derivation, static-data encryption, and Boomlet release of
`doxing_key_for_sar` remain as specified. Dynamic uploads use the existing
cryptographic primitives and canonical encoding.

## Requirements

### DRD-SR-01 Phone non-erasure

No Phone message, credential, secret, or procedure for restoring Phone
credentials can make SAR hide, delete, truncate, reset, or forget an accepted
upload. Later uploads cannot revoke or supersede earlier uploads. SAR makes
every retained upload available to its rescue application.

Retention continues while a related Boomerang setup may remain active. Later
deletion requires independent SAR authority; Phone cannot authorize it.

### DRD-SR-02 Payload agnosticism

The payload is a bounded `bytes` value. The protocol assigns no payload format,
source timestamp, cross-device ordering, freshness, or semantic validation rule.
Payload meaning cannot affect acceptance, retention, receipts, retries, or withdrawal.

### DRD-SR-03 Durable append

SAR keeps separate histories for each `(doxing_data_identifier, device_id)` and
appends each accepted authenticated upload. Exact retries return
the original receipt. No accepted upload replaces another.

### DRD-SR-04 Device IDs and Phone replacement

Phone generates `device_id = random_bytes(32)` once per device upload stream
and a fresh random `upload_id` for each new upload using SPEC Section 9.2.
It numbers new submissions consecutively from `upload_seq_num = 0` for each
`device_id`.
Retries retain both values and the complete original upload.
If Phone cannot continue a device upload stream without reusing a number, it
chooses a new `device_id`. SAR does not reject a valid upload solely for
arriving out of sequence or repeating a sequence number under a different
upload ID. A gap alone does not prove that an upload was sent or lost.

A replacement Phone reconstructs `dynamic_update_auth_key` from the doxing
password, selected SAR, and registration protocol version, then chooses a
fresh device ID. Its first valid upload starts that device ID's history;
no additional device enrollment exchange is required.

Device IDs separate histories and encryption keys. Any holder of
`dynamic_update_auth_key` can submit under any device ID, including an existing
one. SAR does not attest which physical Phone sent an upload. An upload naming
one device ID does not modify entries associated with another ID. Old Phones
retain append authority, but cannot erase or replace accepted entries.

## Objects

```text
DynamicRescueUpload {
  doxing_data_identifier: bytes32,
  device_id: bytes32,
  upload_id: bytes32,
  upload_seq_num: u64,
  dynamic_doxing_data_encrypted_by_device_data_key: CbcCmacEnvelope,
  dynamic_upload_cmac: bytes16
}

DynamicRescueReceipt {
  doxing_data_identifier: bytes32,
  device_id: bytes32,
  upload_id: bytes32,
  upload_seq_num: u64,
  hash_of_dynamic_doxing_data_encrypted_by_device_data_key: bytes32
}
```

Receipts use the existing `SignedMessage` wrapper with content type
`DynamicRescueReceipt`. All objects follow SPEC Sections 8.2 and 8.3 for
canonical encoding, fixed-width types, and encoded-size limits. SPEC Section
8.3 leaves the numeric v1 limits open. Focused vectors use a provisional test
bound. Both new schemas use version 1 and field IDs starting at 1 in declared
order.

Final bounds must limit per-upload work on Phone and SAR. SAR storage is modeled
with unbounded capacity; finite-capacity behavior remains open.

## Cryptographic definitions

### Keys

Use `kdf_counter_cmac_aes256` from SPEC Section 9.4; output lengths are in bytes.
SAR stores `PROTOCOL_VERSION` with each `doxing_data_identifier` at registration.

```text
device_data_key = kdf_counter_cmac_aes256(
  doxing_key_for_sar,
  "Boomerang/sar_dynamic_device_data_key/v1",
  canonical_encode(device_id),
  32
)

dynamic_update_auth_key = kdf_counter_cmac_aes256(
  doxing_key_for_sar,
  "Boomerang/sar_dynamic_update_auth_key/v1",
  canonical_encode(PROTOCOL_VERSION, doxing_data_identifier),
  32
)
```

### Encrypted dynamic doxing data

Encrypt one canonical variable-length `bytes` value with `cbc_cmac_encrypt`
and the following keys and context:

```text
keys = derive_cbc_cmac_keys(device_data_key, "Boomerang/sar_stored_data")

context = canonical_encode(
  "Boomerang", PROTOCOL_VERSION, "sar_dynamic_data",
  doxing_data_identifier, device_id, upload_id, upload_seq_num
)
```

The shared key schedule retains `Boomerang/cbc_cmac/key_schedule/v1`.
Decryption uses `cbc_cmac_decrypt` with the same inputs. The existing fresh-IV,
full-tag, authentication-before-decryption, and rejection rules apply. Dynamic
stored data require no setup or withdrawal identifier.

### Upload authentication

```text
dynamic_upload_cmac = aes256_cmac(
  dynamic_update_auth_key,
  canonical_encode(
    "Boomerang/sar_dynamic_upload/v1", PROTOCOL_VERSION,
    doxing_data_identifier, device_id, upload_id, upload_seq_num,
    dynamic_doxing_data_encrypted_by_device_data_key
  )
)
```

`dynamic_upload_cmac` is the full 16-byte AES-CMAC result and is verified in
constant time. The separate upload key lets SAR authenticate submissions
before duress releases the data key.
It authorizes upload and receipt retrieval only. SAR cannot verify the inner
data-key tag before duress release; accepted undecryptable entries remain
retained, and their failures cannot hide other entries from rescue processing.

### Receipt

```text
dynamic_doxing_data_upload_receipt = DynamicRescueReceipt {
  doxing_data_identifier,
  device_id,
  upload_id,
  upload_seq_num,
  hash_of_dynamic_doxing_data_encrypted_by_device_data_key =
    sha256(canonical_encode(dynamic_doxing_data_encrypted_by_device_data_key))
}

dynamic_doxing_data_upload_receipt_signed_by_sar = sign_message(
  sar_private_key,
  "Boomerang/setup/sar_dynamic_receipt",
  dynamic_doxing_data_upload_receipt
)
```

Phone calls `verify_signature` with the selected SAR public key and exact
domain, then matches `doxing_data_identifier`, device ID, upload ID, sequence
number, and envelope hash against its upload. The existing signature helper
provides BIP340 signing and protocol-version binding.

## Processing

### Initial dynamic rescue data submission

Phone includes the initial dynamic rescue data submission in the SAR
registration exchange of SPEC Section 13.1. The exchange provides
confidentiality and authenticates SAR as the selected `SarId`.

1. Phone derives `dynamic_update_auth_key` and constructs
   `first_dynamic_doxing_data_upload`, a `DynamicRescueUpload` authenticated
   with that key.
2. Phone sends `SetupPhoneSarMessage2` containing payment receipts,
   `doxing_data_identifier`, the static envelope, and the canonical encoding
   of `dynamic_update_auth_key` followed by
   `first_dynamic_doxing_data_upload`.
3. SAR verifies payment against its invoice associated with
   `doxing_data_identifier`, selected `SarId`, and `PROTOCOL_VERSION`. It
   requires the `doxing_data_identifier` in
   `first_dynamic_doxing_data_upload` to match the submitted identifier.
   If SAR already stores `dynamic_update_auth_key` under
   `doxing_data_identifier`, it compares the supplied and stored keys in
   constant time and rejects a mismatch.
4. SAR requires `upload_seq_num = 0` in `first_dynamic_doxing_data_upload`,
   authenticates it using the supplied key, and applies the ordinary upload
   rules. SAR commits the static envelope, `PROTOCOL_VERSION`,
   `dynamic_update_auth_key`, `first_dynamic_doxing_data_upload`, and
   `dynamic_doxing_data_upload_receipt_signed_by_sar` together durably before
   returning the receipt in `SetupSarPhoneMessage2`. A rejected registration
   cannot establish or replace these records.
5. Phone verifies `dynamic_doxing_data_upload_receipt_signed_by_sar` before
   reporting successful SAR registration.

A lost reply is retried with the complete original `SetupPhoneSarMessage2`.
If the first upload was accepted, SAR returns its original receipt. SAR must
not receive `doxing_key_for_sar` during registration.

SPEC Section 13.1 binds the invoice to the selected `SarId`, `PROTOCOL_VERSION`,
and recorded `doxing_data_identifier`. Its first-upload CMAC proves possession
of the supplied `dynamic_update_auth_key`. Phone reaches the selected SAR at
the v3 onion address in `SarId` through Tor; it verifies SAR-signed receipts
under the separate `sar_pubkey`.

### Later uploads and replacement Phones

After initial registration, Phone sends each new `DynamicRescueUpload` as
`dynamic_doxing_data` arrives, using the stored `dynamic_update_auth_key`.
A replacement Phone uses the same procedure with its newly generated device
ID. Payment and static-data enrollment are not repeated for device
replacement.

The SAR registration remains bound to its `PROTOCOL_VERSION`. Replacement
Phones and SAR use that version for authentication, receipt verification, and
historical decoding. A software update must preserve access to accepted data;
a different `PROTOCOL_VERSION` cannot silently replace `dynamic_update_auth_key`
or reinterpret stored uploads.

### Upload outcomes

Upload identity is `(doxing_data_identifier, device_id, upload_id)`. SAR
validates authentication before comparing the canonical bytes of
the complete upload. Transport framing is excluded from the comparison.

| Upload | Result |
| --- | --- |
| New upload identity accepted by SAR | Append once and return a signed receipt |
| Accepted identity with identical canonical bytes | Return the original signed receipt without another append |
| Accepted identity with different canonical bytes | Reject the conflict and preserve the accepted upload and receipt |
| Same sequence number with a different upload ID | Retain both uploads; the claimed order is ambiguous |
| Same upload ID under another `doxing_data_identifier` or device ID | Treat as a separate upload; authenticate with the `dynamic_update_auth_key` stored for the stated `doxing_data_identifier` |

Phone retries the complete original upload after a missing reply. A missing
reply alone does not establish whether acceptance occurred. Re-encrypting the
same plaintext with a new IV creates different bytes and conflicts with an
accepted upload under the same identity. Concurrent delivery follows the same
rules; conflicting submissions under one identity cannot both be accepted.
Exact retry also provides receipt retrieval.

### Durability and activation

SAR accepts an upload durably before releasing its receipt. Retention applies
from acceptance, including when the reply is lost.

The design assumes SAR deployments preserve accepted uploads and their
original receipts through supported storage failures. Crash recovery and
restore validation belong to the deployment. Phone replacement and
reconstruction of `dynamic_update_auth_key` are separate operations described
above.

After valid duress activation, SAR uses the Boomlet-provided
`doxing_key_for_sar` and each stored device ID to derive the required data keys.
It makes every retained upload available to its rescue application.
Retrieval, decryption, interpretation, and SAR crash recovery preserve the fixed
acknowledgment and observability requirements of SPEC Sections 16.4–16.6.

## Security considerations

The protocol cannot establish whether payload data are true, fresh, or useful.
Sequence numbers express uploader-assigned first-submission order within a
device history, not actual send time or cross-device order. Duplicate numbers
leave their relative order unresolved. The rescue application interprets the
payloads. Signed receipts attest acceptance of encrypted bytes. Device IDs
identify histories; they do not authenticate
physical devices or restrict `dynamic_update_auth_key` to one history.

Durability across SAR crashes relies on deployment storage and recovery
procedures. A malicious SAR can discard accepted uploads or refuse to perform
rescue.

Rescue-data confidentiality retains the password assumptions of ADR 0005.
Disclosure of `doxing_key_for_sar` allows derivation of all device-data keys
under that root. Traffic timing and ciphertext sizes retain the metadata risks
already described in the main specification.

## Conformance requirements

| Property | Required result |
| --- | --- |
| First registration | Payment receipts, `doxing_data_identifier`, static envelope, `dynamic_update_auth_key`, and `first_dynamic_doxing_data_upload` share `SetupPhoneSarMessage2`. SAR checks the invoice associated with `doxing_data_identifier` and `PROTOCOL_VERSION`, then commits the static envelope, `PROTOCOL_VERSION`, `dynamic_update_auth_key`, `first_dynamic_doxing_data_upload`, and `dynamic_doxing_data_upload_receipt_signed_by_sar` before replying in `SetupSarPhoneMessage2`. |
| Registration retry | Phone retries the complete original request after a lost reply. SAR returns the original receipt and preserves the static envelope, `dynamic_update_auth_key`, `PROTOCOL_VERSION`, and accepted history. A different `dynamic_update_auth_key` or static envelope is rejected. |
| Authentication | Wrong keys, changed authenticated fields, and context mismatches are rejected. |
| Exact retry | Repeated delivery returns the original signed receipt without another append, including after SAR crash recovery and under concurrent delivery. |
| Sequence order | New uploads use consecutive numbers from zero per device history; SAR retains valid delayed or duplicate-number uploads under distinct IDs without treating gaps as proof of loss. |
| Conflict | Different bytes under the same `doxing_data_identifier`, device ID, and upload ID cannot both be accepted. Include changed-IV cases. |
| Device separation | The same upload ID under different device IDs identifies separate entries. Replacement needs no new device enrollment. `dynamic_update_auth_key` authenticates uploads under any device ID. |
| Non-erasure | Later uploads, replacement, and registration retries preserve accepted entries and receipts. |
| Payload handling | The encrypted plaintext is canonical `bytes`; payload meaning does not affect protocol processing. All retained entries remain available to rescue. |
| Cryptographic encoding | Shared vectors agree on KDF inputs, envelope context, upload CMAC, envelope hash, and receipt signature. Wire-size boundaries follow SPEC Section 8.3. |
| Duress behavior | Registration and upload changes preserve SPEC Sections 16.4–16.6 and Boomlet's existing key-release behavior. |

## Integration checklist

### Completed design integration

- [x] Adopt history retention in the roadmap and record DG-11 and DG-40 for
  independent security review.
- [x] Add `dynamic_update_auth_key`, recorded `PROTOCOL_VERSION`, device histories,
  and receipts to SPEC Section 7.5.
- [x] Allocate upload schema ID 11 and receipt schema ID 30, version 1,
  sequential field IDs; register plaintext `bytes`, registration tuple, and
  receipt domain; regenerate the wire catalog.
- [x] Add exact keys, context, CMAC, receipt rules, and dynamic-context numeric
  limits to SPEC Sections 8–10; publish focused shared vectors.
- [x] Integrate protected first registration, identifier binding, receipt checks,
  replacement, and retries in SPEC Section 13.1 and setup contracts, preserving
  static verification and Section 13.9 finalization.
- [x] Integrate retry and complete-history rescue
  access with the existing placeholder and fixed acknowledgment contract.
- [x] Consolidate conformance requirements in SPEC Section 21; add focused
  vector and bounded acceptance-model checks.
- [x] Update crypto notation and affected setup, withdrawal, and duress source
  diagrams, including `C-SAR-STORE-SEAL` and `C-SAR-STORE-OPEN` with device keys.
- [x] Align DESIGN, GLOSSARY, and supporting documentation; record adoption and
  compatibility here. Prepare the security review handoff.
- [x] Map dynamic failures to SPEC Section 18.1 classes.

### Evidence still required

- [ ] Independently reassess the security model using the
  [review note](../../security_models/security_model_update_note.md).
- [ ] Validate authenticated Phone–SAR transport for registration and later
  uploads with a real implementation, including channel substitution,
  endpoint failover, and payment-replay attacks.
- [ ] Pin the production `PROTOCOL_VERSION` value and canonical type, then
  independently reproduce vectors with the implementation revision recorded.
  Focused vectors explicitly use the test profile `u16(1)`.
- [ ] Define finite-capacity behavior and validate independent deletion authority
  after setup retirement,
  and SAR storage and restoration assumptions under documented deployment
  failure conditions.
- [ ] Test parallel acceptance and acknowledgment timing for safe and duress
  cases with large, unavailable, and partly undecryptable histories.
