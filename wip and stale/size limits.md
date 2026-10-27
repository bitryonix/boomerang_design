# Size limits

`OPEN` means that the limit has not been defined. Values marked `exact` follow
directly from the current canonical wire structure.

`<context>` and `<domain>` denote limit families, not one shared wildcard
limit. Finalizing the registry requires one concrete entry for every registered
context or signature domain. They remain written as families here because the
wire catalog still marks some context type registries as incomplete.

## Collection item counts

| Limit | Definition | Value |
| --- | --- | ---: |
| `DuressCheckSpace.space.item_count` | Required number of encoded country indices | 193 exact |
| `BoomerangParamsSeed.ordered_peer_setup_records.item_count` | Required number of signed peer setup records | 5 exact |
| `BoomerangParamsSeed.wt_ids.item_count` | Maximum number of WT identities in the seed | 5 exact |
| `BoomerangParams.peer_ids.item_count` | Required number of peer identities | 5 exact |
| `BoomerangParams.wt_ids.item_count` | Maximum number of WT identities in the finalized parameters | 5 exact |
| `BoomletBackupState.duress_consent_set.item_count` | Required number of selected country indices | 5 exact |
| `Pong.prev_pings.item_count` | Required number of signed pings from peers other than the recipient | 4 exact |
| `DuressSignalIndex.indices.item_count` | Required number of returned original-list indices | 5 exact |
| `list.<context>.item_count` | Required or maximum item count for each protocol list that is selected by context rather than declared as a schema field | `OPEN` |

## Variable field payload sizes

Each value is the maximum number of payload bytes, excluding the type tag and
four-byte payload-length field.

| Limit | Definition | Value |
| --- | --- | ---: |
| `CbcCmacEnvelope.ciphertext.max_payload_bytes` | Maximum ciphertext bytes accepted across all registered encryption contexts | `OPEN` |
| `CbcCmacEnvelope.<context>.ciphertext.max_payload_bytes` | Maximum ciphertext payload bytes accepted for each registered encryption context | Per-context table below |
| `SignedMessage.domain.max_payload_bytes` | Maximum ASCII bytes in a registered signature domain | 45 exact |
| `SarServiceFeePaymentInfo.service_fee_invoice.max_payload_bytes` | Maximum UTF-8 bytes in a SAR service-fee invoice | `OPEN` |
| `WtServiceFeePaymentInfo.service_fee_invoice.max_payload_bytes` | Maximum UTF-8 bytes in a WT service-fee invoice | `OPEN` |
| `WtId.wt_tor_address.max_payload_bytes` | Maximum UTF-8 bytes in a canonical WT Tor address | `OPEN` |
| `SarId.sar_tor_address.max_payload_bytes` | Maximum UTF-8 bytes in a canonical SAR Tor address | `OPEN` |
| `StaticDoxingData.name.max_payload_bytes` | Maximum UTF-8 bytes in the subject's normalized name | `OPEN` |
| `StaticDoxingData.national_id.max_payload_bytes` | Maximum UTF-8 bytes in the subject's normalized national ID | `OPEN` |
| `StaticDoxingData.address_home.max_payload_bytes` | Maximum UTF-8 bytes in the subject's normalized home address | `OPEN` |
| `StaticDoxingData.address_work.max_payload_bytes` | Maximum UTF-8 bytes in the subject's normalized work address | `OPEN` |
| `StaticDoxingData.phone_number_mobile.max_payload_bytes` | Maximum UTF-8 bytes in the subject's canonical mobile number | `OPEN` |
| `StaticDoxingData.phone_number_home.max_payload_bytes` | Maximum UTF-8 bytes in the subject's canonical home number | `OPEN` |
| `StaticDoxingData.phone_number_work.max_payload_bytes` | Maximum UTF-8 bytes in the subject's canonical work number | `OPEN` |
| `StaticDoxingData.trusted_person_name.max_payload_bytes` | Maximum UTF-8 bytes in the trusted person's normalized name | `OPEN` |
| `StaticDoxingData.trusted_person_address.max_payload_bytes` | Maximum UTF-8 bytes in the trusted person's normalized address | `OPEN` |
| `StaticDoxingData.trusted_person_phone_number.max_payload_bytes` | Maximum UTF-8 bytes in the trusted person's canonical phone number | `OPEN` |
| `DynamicDoxingData.payload.max_payload_bytes` | Maximum bytes in a registered dynamic doxing-data payload | `OPEN` |
| `DynamicDoxingData.<schema_id>.payload.max_payload_bytes` | Maximum payload bytes for each registered dynamic doxing-data schema | `OPEN` |
| `ServicePaymentReceipt.service_id.max_payload_bytes` | Maximum UTF-8 bytes in a canonical service identifier | `OPEN` |
| `ServicePaymentReceipt.invoice_reference.max_payload_bytes` | Maximum UTF-8 bytes in an accepted invoice reference | `OPEN` |
| `ServicePaymentReceipt.payment_proof.max_payload_bytes` | Maximum bytes in an accepted payment proof | `OPEN` |
| `PeerSetupRecord.tor_address.max_payload_bytes` | Maximum UTF-8 bytes in a canonical peer Tor address | `OPEN` |
| `BoomerangParams.boomerang_descriptor.max_payload_bytes` | Maximum bytes in a canonical Boomerang descriptor | `OPEN` |
| `WtSarSetupResponse.wt_suffix.max_payload_bytes` | Maximum UTF-8 bytes in a canonical WT suffix | `OPEN` |
| `BoomletBackupState.tor_secret_key.max_payload_bytes` | Maximum bytes in the selected Tor secret-key encoding | `OPEN` |

## Collection encoded sizes

Each value includes the list tag, four-byte count, and all encoded items.

| Limit | Definition | Value |
| --- | --- | ---: |
| `DuressCheckSpace.space.max_encoded_bytes` | Maximum canonical size of the complete country-index list | 584 exact |
| `BoomerangParamsSeed.ordered_peer_setup_records.max_encoded_bytes` | Maximum canonical size of five signed peer setup records | `OPEN` |
| `BoomerangParamsSeed.wt_ids.max_encoded_bytes` | Maximum canonical size of the seed's WT identity list | `OPEN` |
| `BoomerangParams.peer_ids.max_encoded_bytes` | Maximum canonical size of the five peer identities | 570 exact |
| `BoomerangParams.wt_ids.max_encoded_bytes` | Maximum canonical size of the finalized WT identity list | `OPEN` |
| `BoomletBackupState.duress_consent_set.max_encoded_bytes` | Maximum canonical size of the five selected country indices | 20 exact |
| `Pong.prev_pings.max_encoded_bytes` | Maximum canonical size of four signed pings | 833 exact |
| `DuressSignalIndex.indices.max_encoded_bytes` | Complete canonical size of five encoded indices | 20 exact |
| `list.<context>.max_encoded_bytes` | Maximum canonical size of each protocol list that is selected by context rather than declared as a schema field | `OPEN` |

## Schema encoded sizes

Each value is the maximum complete canonical encoding, including the struct
header, field IDs, and encoded field values.

| Packet ID | Limit | Definition | Value |
| ---: | --- | --- | ---: |
| 1 | `CbcCmacEnvelope.max_encoded_bytes` | Maximum canonical size of a CBC-CMAC envelope | `OPEN` |
| 2 | `SignedMessage.max_encoded_bytes` | Maximum canonical size across registered signature domains and content types | `OPEN` |
| 3 | `MessageWithNonce.max_encoded_bytes` | Maximum canonical size across registered nonce-wrapper contexts | 637 exact |
| 4 | `PaddedMessage.max_encoded_bytes` | Maximum canonical size across registered signed content and padding contexts | 322 exact |
| 5 | `SarServiceFeePaymentInfo.max_encoded_bytes` | Maximum canonical size of SAR fee-payment information | `OPEN` |
| 6 | `WtServiceFeePaymentInfo.max_encoded_bytes` | Maximum canonical size of WT fee-payment information | `OPEN` |
| 7 | `DuressCheckSpace.max_encoded_bytes` | Complete canonical size of the duress check space | 593 exact |
| 8 | `WtId.max_encoded_bytes` | Maximum canonical size of a WT identity | `OPEN` |
| 9 | `SarId.max_encoded_bytes` | Maximum canonical size of a SAR identity | `OPEN` |
| 10 | `StaticDoxingData.max_encoded_bytes` | Maximum canonical size of static doxing data | `OPEN` |
| 11 | `DynamicDoxingData.max_encoded_bytes` | Maximum canonical size of dynamic doxing data | `OPEN` |
| 12 | `ServicePaymentReceipt.max_encoded_bytes` | Maximum canonical size of a service payment receipt | `OPEN` |
| 13 | `PeerId.max_encoded_bytes` | Complete canonical size of a peer identity | 113 exact |
| 14 | `PeerSetupRecord.max_encoded_bytes` | Maximum canonical size of a peer setup record | `OPEN` |
| 15 | `MilestoneBlocks.max_encoded_bytes` | Complete canonical size of milestone block heights | 49 exact |
| 16 | `BoomerangParamsSeed.max_encoded_bytes` | Maximum canonical size of a parameter seed | `OPEN` |
| 17 | `BoomerangParams.max_encoded_bytes` | Maximum canonical size of finalized parameters | `OPEN` |
| 18 | `WtSetupReceipt.max_encoded_bytes` | Complete canonical size of a WT setup receipt | 77 exact |
| 19 | `SarSetupResponse.max_encoded_bytes` | Complete canonical size of a SAR setup response | 131 exact |
| 20 | `WtSarSetupResponse.max_encoded_bytes` | Maximum canonical size of a combined WT and SAR setup response | `OPEN` |
| 21 | `BoomletBackupRequest.max_encoded_bytes` | Complete canonical size of a Boomlet backup request | 78 exact |
| 22 | `BackupDone.max_encoded_bytes` | Complete canonical size of a backup completion message | 114 exact |
| 23 | `BoomletBackupState.max_encoded_bytes` | Maximum canonical size of Boomlet backup state | `OPEN` |
| 24 | `TxApproval.max_encoded_bytes` | Complete canonical size of a transaction approval | 84 exact |
| 25 | `WtTxApproval.max_encoded_bytes` | Complete canonical size of a WT transaction approval | 49 exact |
| 26 | `TxCommit.max_encoded_bytes` | Complete canonical size of a transaction commitment | 49 exact |
| 27 | `Ping.max_encoded_bytes` | Complete canonical size of a ping | 63 exact |
| 28 | `Pong.max_encoded_bytes` | Maximum canonical size of a pong | 884 exact |
| 29 | `DuressSignalIndex.max_encoded_bytes` | Complete canonical size of a returned duress selection | 29 exact |

## Context-selected field encoded sizes

The non-contextual entry for each `value` field is the maximum across all of
its registered contexts. Each contextual entry is the tighter limit selected
before that field is decoded.

| Packet ID | Limit | Definition | Value |
| ---: | --- | --- | ---: |
| 2 | `SignedMessage.content.max_encoded_bytes` | Maximum signed content size across every registered signature domain | `OPEN` |
| 2 | `SignedMessage.<domain>.content.max_encoded_bytes` | Maximum signed content size for each registered signature domain and exact content type | Per-domain table below |
| 3 | `MessageWithNonce.content.max_encoded_bytes` | Maximum nonce-wrapper content size across every registered context | 593 exact |
| 3 | `MessageWithNonce.<context>.content.max_encoded_bytes` | Maximum nonce-wrapper content size for each registered context and exact content type | Per-context table below |
| 4 | `PaddedMessage.content.max_encoded_bytes` | Maximum padded-message content size across every registered context | 207 exact |
| 4 | `PaddedMessage.<context>.content.max_encoded_bytes` | Maximum padded-message content size for each registered envelope-signature context | Per-context table below |
| 4 | `PaddedMessage.padding.max_encoded_bytes` | Maximum padding-field size across every registered context | 104 exact |
| 4 | `PaddedMessage.<context>.padding.max_encoded_bytes` | Maximum padding-field size for each registered envelope-signature context; this is the `duress_placeholder` envelope in the withdrawal contexts | 104 exact in both registered contexts |
| 23 | `BoomletBackupState.replay_state.max_encoded_bytes` | Maximum replay-state size across every registered active protocol state | `OPEN` |
| 23 | `BoomletBackupState.<context>.replay_state.max_encoded_bytes` | Maximum replay-state size for each registered active protocol state and exact value type | `OPEN` |

## Contextual wrapper encoded sizes

| Packet ID | Limit | Definition | Value |
| ---: | --- | --- | ---: |
| 1 | `CbcCmacEnvelope.<context>.max_plaintext_bytes` | Maximum canonical plaintext bytes for each registered encryption context | Per-context table below |
| 1 | `CbcCmacEnvelope.<context>.max_encoded_bytes` | Maximum complete envelope size for each registered encryption context | Per-context table below |
| 2 | `SignedMessage.<domain>.max_encoded_bytes` | Maximum signed-message size for each registered domain and exact content type | Per-domain table below |
| 3 | `MessageWithNonce.<context>.max_encoded_bytes` | Maximum nonce-wrapper size for each registered content context | Per-context table below |
| 4 | `PaddedMessage.<context>.max_encoded_bytes` | Maximum padded-message size for each registered content and padding context | Per-context table below |

### Registered signature domains

The signature is a 64-byte Schnorr value and its canonical `bytes64` encoding
occupies 65 bytes. A complete signed wrapper is therefore `119 + ASCII domain
bytes + encoded content bytes`. `OPEN` below is caused only by a variable-size
content type, not by the signature representation.

| Domain | Content type | Content bytes | Signed bytes | Depth |
| --- | --- | ---: | ---: | ---: |
| `Boomerang/setup/peer_setup_record` | `PeerSetupRecord` | `OPEN` | `OPEN` | 3 exact |
| `Boomerang/setup/params_review_approval` | `MessageWithNonce<bytes32>` | 77 exact | 234 exact | 2 exact |
| `Boomerang/setup/agreement` | `bytes32` | 33 exact | 177 exact | 1 exact |
| `Boomerang/setup/wt_receipt` | `WtSetupReceipt` | 77 exact | 222 exact | 2 exact |
| `Boomerang/setup/phase_checkpoint` | `bytes32` | 33 exact | 184 exact | 1 exact |
| `Boomerang/setup/sar_id` | `tuple<bytes32, SarId>` | `OPEN` | `OPEN` | 3 exact |
| `Boomerang/setup/sar_response` | `SarSetupResponse` | 131 exact | 278 exact | 2 exact |
| `Boomerang/setup/wt_sar_response` | `WtSarSetupResponse` | `OPEN` | `OPEN` | 3 exact |
| `Boomerang/setup/backup_request` | `BoomletBackupRequest` | 78 exact | 227 exact | 2 exact |
| `Boomerang/setup/backup_done` | `BackupDone` | 114 exact | 260 exact | 2 exact |
| `Boomerang/withdrawal/tx_review_approval` | `MessageWithNonce<bytes32>` | 77 exact | 235 exact | 2 exact |
| `Boomerang/withdrawal/tx_approval` | `TxApproval` | 84 exact | 235 exact | 2 exact |
| `Boomerang/withdrawal/wt_tx_approval` | `WtTxApproval` | 49 exact | 203 exact | 2 exact |
| `Boomerang/withdrawal/approval_set_attestation` | `bytes32` | 33 exact | 197 exact | 1 exact |
| `Boomerang/withdrawal/tx_commit` | `TxCommit` | 49 exact | 198 exact | 2 exact |
| `Boomerang/withdrawal/tx_commit_envelope` | `PaddedMessage<SignedMessage<TxCommit>, CbcCmacEnvelope<bytes32>>` | 313 exact | 471 exact | 4 exact |
| `Boomerang/withdrawal/wt_tx_commit_ack` | `SignedMessage<TxCommit>` | 198 exact | 354 exact | 3 exact |
| `Boomerang/withdrawal/sar_placeholder_response` | `CbcCmacEnvelope<bytes32>` | 104 exact | 268 exact | 2 exact |
| `Boomerang/withdrawal/ping` | `Ping` | 63 exact | 207 exact | 2 exact |
| `Boomerang/withdrawal/ping_envelope` | `PaddedMessage<SignedMessage<Ping>, CbcCmacEnvelope<bytes32>>` | 322 exact | 475 exact | 4 exact |
| `Boomerang/withdrawal/pong` | `Pong` | 884 exact | 1,028 exact | 5 exact |

### Registered padded-message contexts

| Signature domain selecting the context | Content bytes | Duress-placeholder bytes | Padded bytes | Depth |
| --- | ---: | ---: | ---: | ---: |
| `Boomerang/withdrawal/tx_commit_envelope` | 198 exact | 104 exact | 313 exact | 3 exact |
| `Boomerang/withdrawal/ping_envelope` | 207 exact | 104 exact | 322 exact | 3 exact |

### Registered nonce-wrapper contexts

| Protocol context | Content type | Content bytes | Wrapper bytes | Depth |
| --- | --- | ---: | ---: | ---: |
| `setup/duress_challenge` | `DuressCheckSpace` | 593 exact | 637 exact | 3 exact |
| `setup/duress_response` | `DuressSignalIndex` | 29 exact | 73 exact | 3 exact |
| `setup/params_review` | `bytes32` | 33 exact | 77 exact | 1 exact |
| `withdrawal/tx_review` | `bytes32` | 33 exact | 77 exact | 1 exact |
| `withdrawal/duress_challenge` | `DuressCheckSpace` | 593 exact | 637 exact | 3 exact |
| `withdrawal/duress_response` | `DuressSignalIndex` | 29 exact | 73 exact | 3 exact |

### Registered CBC-CMAC encryption contexts

All canonical CBC-CMAC context labels used by the protocol are listed here.
`OPEN` rows have a variable-size plaintext and therefore require a policy bound;
their presence in the registry is not unresolved.

| Context label | Plaintext type | Plaintext bytes | Ciphertext bytes | Envelope bytes | Plaintext depth |
| --- | --- | ---: | ---: | ---: | ---: |
| `backup_state` | `BoomletBackupState` | `OPEN` | `OPEN` | `OPEN` | 4, replay-state context unresolved |
| `duress_challenge` | `MessageWithNonce<DuressCheckSpace>` | 637 exact | 640 exact | 696 exact | 3 exact |
| `duress_response` | `MessageWithNonce<DuressSignalIndex>` | 73 exact | 80 exact | 136 exact | 3 exact |
| `params_review` | `MessageWithNonce<bytes32>` | 77 exact | 80 exact | 136 exact | 1 exact |
| `params_review_approval` | `SignedMessage<MessageWithNonce<bytes32>>` | 234 exact | 240 exact | 296 exact | 2 exact |
| `sar_dynamic_data` | `DynamicDoxingData` | `OPEN` | `OPEN` | `OPEN` | 1 exact |
| `sar_id` | `SignedMessage<tuple<bytes32, SarId>>` | `OPEN` | `OPEN` | `OPEN` | 3 exact |
| `sar_identifier` | `tuple<bytes32, bytes32>` | 69 exact | 80 exact | 136 exact | 1 exact |
| `sar_response` | `SignedMessage<SarSetupResponse>` | 278 exact | 288 exact | 344 exact | 2 exact |
| `sar_static_data` | `StaticDoxingData` | `OPEN` | `OPEN` | `OPEN` | 1 exact |
| `withdrawal_duress_challenge` | `MessageWithNonce<DuressCheckSpace>` | 637 exact | 640 exact | 696 exact | 3 exact |
| `withdrawal_duress_placeholder` | `bytes32` | 33 exact | 48 exact | 104 exact | 0 exact |
| `withdrawal_duress_response` | `MessageWithNonce<DuressSignalIndex>` | 73 exact | 80 exact | 136 exact | 3 exact |
| `withdrawal_ping` | `SignedMessage<PaddedMessage<SignedMessage<Ping>, CbcCmacEnvelope<bytes32>>>` | 475 exact | 480 exact | 536 exact | 4 exact |
| `withdrawal_pong` | `SignedMessage<Pong>` | 1,028 exact | 1,040 exact | 1,096 exact | 5 exact |
| `withdrawal_psbt` | `bytes` | `OPEN` | `OPEN` | `OPEN` | 0 exact |
| `withdrawal_sar_placeholder_response` | `SignedMessage<CbcCmacEnvelope<bytes32>>` | 268 exact | 272 exact | 328 exact | 2 exact |
| `withdrawal_tx_approval` | `SignedMessage<TxApproval>` | 235 exact | 240 exact | 296 exact | 2 exact |
| `withdrawal_tx_commit` | `SignedMessage<PaddedMessage<SignedMessage<TxCommit>, CbcCmacEnvelope<bytes32>>>` | 471 exact | 480 exact | 536 exact | 4 exact |
| `withdrawal_tx_review` | `MessageWithNonce<bytes32>` | 77 exact | 80 exact | 136 exact | 1 exact |
| `withdrawal_tx_review_approval` | `SignedMessage<MessageWithNonce<bytes32>>` | 235 exact | 240 exact | 296 exact | 2 exact |

## Schema nesting limits

Depth counts simultaneously open structs, lists, and tuples. Primitive values
do not add depth.

| Packet ID | Limit | Definition | Value |
| ---: | --- | --- | ---: |
| 1 | `CbcCmacEnvelope.max_nesting_depth` | Deepest valid path from this schema | 1 exact |
| 2 | `SignedMessage.max_nesting_depth` | Deepest valid path across registered content types | 5 exact |
| 3 | `MessageWithNonce.max_nesting_depth` | Deepest valid path across registered content types | 3 exact |
| 4 | `PaddedMessage.max_nesting_depth` | Deepest valid path from this schema | 3 exact |
| 5 | `SarServiceFeePaymentInfo.max_nesting_depth` | Deepest valid path from this schema | 2 exact |
| 6 | `WtServiceFeePaymentInfo.max_nesting_depth` | Deepest valid path from this schema | 2 exact |
| 7 | `DuressCheckSpace.max_nesting_depth` | Deepest valid path from this schema | 2 exact |
| 8 | `WtId.max_nesting_depth` | Deepest valid path from this schema | 1 exact |
| 9 | `SarId.max_nesting_depth` | Deepest valid path from this schema | 1 exact |
| 10 | `StaticDoxingData.max_nesting_depth` | Deepest valid path from this schema | 1 exact |
| 11 | `DynamicDoxingData.max_nesting_depth` | Deepest valid path from this schema | 1 exact |
| 12 | `ServicePaymentReceipt.max_nesting_depth` | Deepest valid path from this schema | 1 exact |
| 13 | `PeerId.max_nesting_depth` | Deepest valid path from this schema | 1 exact |
| 14 | `PeerSetupRecord.max_nesting_depth` | Deepest valid path from this schema | 2 exact |
| 15 | `MilestoneBlocks.max_nesting_depth` | Deepest valid path from this schema | 1 exact |
| 16 | `BoomerangParamsSeed.max_nesting_depth` | Deepest valid path across registered signed setup records | 5 exact |
| 17 | `BoomerangParams.max_nesting_depth` | Deepest valid path from this schema | 3 exact |
| 18 | `WtSetupReceipt.max_nesting_depth` | Deepest valid path from this schema | 1 exact |
| 19 | `SarSetupResponse.max_nesting_depth` | Deepest valid path from this schema | 1 exact |
| 20 | `WtSarSetupResponse.max_nesting_depth` | Deepest valid path from this schema | 2 exact |
| 21 | `BoomletBackupRequest.max_nesting_depth` | Deepest valid path from this schema | 1 exact |
| 22 | `BackupDone.max_nesting_depth` | Deepest valid path from this schema | 1 exact |
| 23 | `BoomletBackupState.max_nesting_depth` | Deepest valid path across its contextual nested values | `OPEN`, at least 4 |
| 24 | `TxApproval.max_nesting_depth` | Deepest valid path from this schema | 1 exact |
| 25 | `WtTxApproval.max_nesting_depth` | Deepest valid path from this schema | 1 exact |
| 26 | `TxCommit.max_nesting_depth` | Deepest valid path from this schema | 1 exact |
| 27 | `Ping.max_nesting_depth` | Deepest valid path from this schema | 1 exact |
| 28 | `Pong.max_nesting_depth` | Deepest valid path across registered signed pings | 4 exact |
| 29 | `DuressSignalIndex.max_nesting_depth` | Deepest valid path through the five-index list | 2 exact |
| N/A | `global.max_nesting_depth` | Deepest permitted path across every v1 canonical object and context | `OPEN`, at least 5 |

## Contextual nesting limits

These limits cover decoded context-selected values and decrypted plaintexts;
the envelope schema's own depth does not include the plaintext hidden in its
`ciphertext` field.

| Packet ID | Limit | Definition | Value |
| ---: | --- | --- | ---: |
| 1 | `CbcCmacEnvelope.<context>.max_plaintext_nesting_depth` | Deepest canonical plaintext permitted for each registered encryption context | Per-context table above |
| 2 | `SignedMessage.<domain>.max_nesting_depth` | Deepest signed-message path for each registered signature domain and exact content type | Per-domain table above |
| 3 | `MessageWithNonce.<context>.max_nesting_depth` | Deepest nonce-wrapper path for each registered content context | Per-context table above |
| 4 | `PaddedMessage.<context>.max_nesting_depth` | Deepest padded-message path for each registered content and padding context | 3 exact in both registered contexts |
| 23 | `BoomletBackupState.<context>.replay_state.max_nesting_depth` | Deepest replay-state value for each registered active protocol state | `OPEN` |

## Top-level object limits

| Limit | Definition | Value |
| --- | --- | ---: |
| `top_level.<context>.max_encoded_bytes` | Maximum canonical size accepted at each permitted top-level protocol boundary | `OPEN` |
| `top_level.<context>.max_nesting_depth` | Deepest canonical value accepted at each permitted top-level protocol boundary | `OPEN` |
| `global.max_canonical_object_bytes` | Largest maximum among all permitted top-level v1 contexts | `OPEN` |

## PSBT and Bitcoin transaction limits

| Limit | Definition | Value |
| --- | --- | ---: |
| `psbt.max_bytes` | Maximum complete PSBT bytes accepted by every component | `OPEN` |
| `psbt.max_unsigned_transaction_bytes` | Maximum serialized unsigned-transaction bytes in an accepted PSBT | `OPEN` |
| `psbt.max_inputs` | Maximum number of transaction inputs in an accepted PSBT | `OPEN` |
| `psbt.max_outputs` | Maximum number of transaction outputs in an accepted PSBT | `OPEN` |
| `psbt.<map>.max_key_value_pairs` | Maximum key-value pairs in each supported global, input, or output map | `OPEN` |
| `psbt.<map>.<key_type>.max_entries` | Maximum entries of each supported key type in its permitted map | `OPEN` |
| `psbt.<map>.<key_type>.max_key_bytes` | Maximum key bytes for each supported key type, including key data | `OPEN` |
| `psbt.<map>.<key_type>.max_value_bytes` | Maximum value bytes for each supported key type | `OPEN` |
| `psbt.<map>.max_proprietary_entries` | Maximum non-semantic proprietary entries retained in each supported map | `OPEN` |
| `psbt.<map>.max_proprietary_bytes` | Maximum aggregate key and value bytes retained for proprietary entries in each supported map | `OPEN` |
| `psbt.max_non_witness_utxo_bytes` | Maximum serialized previous-transaction bytes in one non-witness UTXO field | `OPEN` |
| `psbt.max_derivation_path_steps` | Maximum child-number steps in one accepted BIP32 or Taproot derivation path | `OPEN` |
| `psbt.max_tap_leaf_hashes_per_derivation` | Maximum Taproot leaf hashes associated with one derivation entry | `OPEN` |
| `psbt.max_tap_leaf_scripts_per_input` | Maximum Taproot leaf-script entries accepted for one input | `OPEN` |
| `psbt.max_partial_signatures_per_input` | Maximum partial-signature entries accepted for one input | `OPEN` |
| `psbt.max_script_bytes.<position>` | Maximum script bytes for each supported PSBT script position | `OPEN` |
| `psbt.max_witness_items_per_input` | Maximum number of witness stack items for one input | `OPEN` |
| `psbt.max_witness_item_bytes` | Maximum bytes in one witness stack item | `OPEN` |
| `psbt.max_witness_bytes_per_input` | Maximum aggregate witness bytes represented for one input | `OPEN` |
| `psbt.max_witness_bytes` | Maximum aggregate witness bytes represented by the complete PSBT | `OPEN` |
| `transaction.max_bytes` | Maximum complete serialized transaction bytes accepted by every component | `OPEN` |
| `transaction.max_inputs` | Maximum number of inputs in an accepted transaction | `OPEN` |
| `transaction.max_outputs` | Maximum number of outputs in an accepted transaction | `OPEN` |
| `transaction.max_script_bytes.<position>` | Maximum script bytes for each supported transaction script position | `OPEN` |
| `transaction.max_witness_items_per_input` | Maximum witness stack items accepted for one transaction input | `OPEN` |
| `transaction.max_witness_item_bytes` | Maximum bytes accepted in one transaction witness item | `OPEN` |
| `transaction.max_witness_bytes_per_input` | Maximum aggregate serialized witness bytes for one transaction input | `OPEN` |
| `transaction.max_witness_bytes` | Maximum aggregate serialized transaction witness bytes | `OPEN` |
| `transaction.max_weight` | Maximum transaction weight accepted by every component | `OPEN` |
