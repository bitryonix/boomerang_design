# Setup no-prose guards

The following setup guard abbreviations expand to the conditions below.

## Duress input

`GS-DURESS-SELECTION(selection, columns)` requires:

```text
count(selection) == count(columns)
count(selection) == 5
selection contains exactly one displayed choice from each columns column
each selection value exists in its corresponding columns column
```

`GS-DURESS-INDEXES(indices)` requires:

```text
count(indices) == 5
indices are distinct
indices are in range [1..193]
```

`GS-DURESS-RESPONSE(response, nonce)` requires:

```text
response.nonce == nonce
GS-DURESS-INDEXES(response.content.indices)
```

`GS-DURESS-CONFIRM(response, nonce, space, consent)` requires:

```text
GS-DURESS-RESPONSE(response, nonce)
space.space[response.content.indices] == consent
```

## Peer agreement

`GS-SIGNED-CONTENT(collection, expected, range)` requires every signed value
selected by `range` from `collection` to have content equal to `expected`.

`GS-SETUP-CHECKPOINTS(collection, checkpoint)` expands to:

```text
GS-SIGNED-CONTENT(collection, checkpoint, [1 <= i <= 4])
```

The diagram's following collection assignment remains part of the operation;
this guard only abbreviates its repeated equality checks.

## External receipts

`GS-WT-RECEIPT(receipt, setup_id, fingerprint)` requires:

```text
receipt.content.setup_instance_id == setup_id
receipt.content.boomerang_params_fingerprint == fingerprint
```

`GS-SAR-PAYLOAD(payload, setup_id, identifier)` requires:

```text
payload.setup_instance_id == setup_id
payload.doxing_data_identifier == identifier
```

## SAR registration

`GS-SAR-REGISTRATION(payment_receipts, identifier, static_envelope, protocol_version, dynamic_update_auth_key, first_dynamic_doxing_data_upload)`
applies the SAR acceptance and atomic commit conditions of
`G-SAR-REGISTRATION` in `setup_development_contracts.md` and SPEC Section 13.1
to the combined `SetupPhoneSarMessage2`.

`GS-DYNAMIC-RECEIPT(receipt, upload)` applies the Phone verification in SPEC
Section 10: selected SAR signer, receipt domain, registered `PROTOCOL_VERSION`,
`doxing_data_identifier`, device ID, upload ID, sequence number, and encrypted
payload hash must match the submitted upload.
