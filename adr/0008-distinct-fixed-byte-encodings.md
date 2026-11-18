# ADR 0008. Distinct Tags for Fixed-Width Byte Strings

- **Status:** Accepted
- **Recorded:** 2026-08-31

## Context

The canonical data model distinguishes variable `bytes` from the fixed types
`bytes16`, `bytes32`, `bytes33`, and `bytes64`. Encoding a fixed value as generic
bytes adds a four-byte length field and makes the enclosing schema responsible
for rejecting a length already inherent in the field type.

These fixed widths recur in IVs, authentication tags, hashes, identifiers,
private scalars, compressed public keys, and signatures. Boomlet is intended
for constrained Java Card hardware, and fixed-width parsing should avoid
unnecessary length processing.

## Decision

The canonical tag assignments follow.

| Type | Tag | Payload width | Total encoded size |
| --- | --- | ---: | ---: |
| `bytes16` | `0x22` | 16 | 17 |
| `bytes32` | `0x23` | 32 | 33 |
| `bytes33` | `0x24` | 33 | 34 |
| `bytes64` | `0x25` | 64 | 65 |

The tag is followed directly by the fixed-width payload. There is no length
field. Tag `0x20` remains the canonical representation of genuinely variable
`bytes` and retains its four-byte unsigned length field.

The expected schema type determines which encoding is valid. A fixed-width
field cannot use generic `bytes`, and a variable field cannot use a fixed-width
tag merely because its current value has one of the fixed lengths.

## Consequences

- Every fixed-width value saves four encoded bytes relative to generic
  length-prefixed bytes.
- Decoders determine the payload width from the tag and reject truncation,
  extension, or type substitution.
- Encoders and decoders need four additional primitive cases.
- Adding another fixed byte-string width requires a new canonical tag or use of
  genuinely variable `bytes` in a later schema.
- All conformance vectors and golden transcripts must use these tags.
