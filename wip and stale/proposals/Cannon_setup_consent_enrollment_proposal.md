# Setup consent enrollment on ST

[Enrollment sequence](Cannon_setup_consent_enrollment_sequence.puml).
Proposed and unadopted. Applies to initial consent enrollment in SPEC Sections
12, 13.3, and 16.1.

## Procedure

Iso establishes the Boomlet and ST identities through the existing trusted
pairing procedure. Enrollment is available only in `INSTALLED`, before any
consent set has been enrolled.

1. Boomlet generates one fresh `consent_enrollment_nonce` and one permutation of
   1–193. It sends `consent_set_query`, a `MessageWithNonce<DuressCheckSpace>`,
   encrypted for its paired ST through Iso.
2. ST authenticates the query and displays five independently shuffled country
   columns. The user selects five distinct countries.
3. ST independently reshuffles the columns for confirmation, using the same
   underlying permutation. It compares the unordered country sets and immediately
   repeats confirmation on mismatch. Both entries stay on ST.
4. After a match, ST sends one encrypted `consent_set_answer`, a
   `MessageWithNonce<DuressSignalIndex>`. Its indices refer to the original
   permutation, and its nonce equals `consent_enrollment_nonce`.
5. Boomlet verifies the paired channel, enrollment phase, exact outstanding nonce,
   and five distinct indices in 1–193. It resolves the countries as an unordered set,
   installs `duress_consent_set`, and enters `ST_ENROLLED` before returning
   `magic: "setup_duress_finished"` to Iso. Iso continues its existing setup flow.

ST is responsible for confirming that both entries match. Boomlet receives only
the confirmed answer and stores no provisional consent set.

## Channels and recovery

Use setup's CBC-CMAC directional channels and existing wrapper types. The exact
contexts are `canonical_encode("setup_consent_set_query")` and
`canonical_encode("setup_consent_set_answer")`. These contexts identify initial
enrollment; withdrawal and rotation messages cannot satisfy this exchange.

Keep one immutable pending query. Losing it before acceptance requires a fresh
nonce and permutation. An accepted answer installs the set; retain its
digest and completion reply for recovery under SPEC Section 18.3. Duplicate
answers recover that reply before fresh-phase checks. Enrollment cannot overwrite
an enrolled set or restart later setup phases.

Before ST emits its final answer, lost local input may be entered again. ST
records its exact final answer before output and freezes it thereafter. If that
record cannot be recovered and emission cannot be ruled out, explicitly retire
the pending query before issuing a fresh one. This is allowed only while Boomlet
remains `INSTALLED`; an enrolled device recovers completion instead. A status
snapshot alone cannot rule out an answer still in transit.

## Workload and security

Boomlet generates one permutation and nonce and performs one encryption and one
decryption. This saves one of each operation, four relay messages, the provisional
set, and the separate confirmation stage.

Matching and encoding rely on ST's display and input integrity, already assumed
by SPEC Section 5.3. Boomlet loses the independent comparison of two separately
encoded entries, so an otherwise valid ST encoding fault can escape detection.
A compromised ST could already fabricate two matching responses. Independent
confirmation shuffling preserves the need to repeat country names.

If paired consent rotation is also adopted, initialize `consent_counter := 0`
and `under_duress := false`, and copy their current values in the authenticated
backup transfer. Enrollment retries never reset them.

Adoption requires a matching protocol profile and replaces the `INSTALLED` to
`ST_ENROLLED` prerequisite with one authenticated, locally confirmed ST answer.
