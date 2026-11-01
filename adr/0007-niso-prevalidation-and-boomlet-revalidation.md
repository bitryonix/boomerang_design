# ADR 0007: Niso Prevalidation and Boomlet Revalidation

- **Status:** Accepted
- **Recorded:** 2026-08-25

## Context

Niso is not trusted to authorize spending, but it is the online component that
receives local and network inputs, maintains relay state, obtains the Bitcoin
chain view, presents transactions to the user, hydrates PSBTs, and transports
objects to Boomlet. Those operations require Niso to parse and interpret the
objects it handles.

Boomlet runs on a constrained secure element. Signature verification,
authenticated decryption, ECDH and KDF operations, transaction parsing, user
interaction, and persistent state transitions consume limited computation,
memory, write endurance, and interface bandwidth. Forwarding every object to
Boomlet before applying checks would let traffic with an invalid signer,
identity, ceremony, sequence, block height, schema, or size consume those resources.
An attacker without protocol authorization could then force work on Boomlet
through Niso.

The trust model therefore has two separate questions. Niso must decide whether
an object is valid enough for Niso to process, present, transform, or forward.
Boomlet must decide whether the object satisfies the conditions for changing
trusted state or releasing its signing share.

## Decision

Niso validates every applicable condition that it can evaluate from the object
and its locally available protocol state before it acts on that object or sends
it to Boomlet. The applicable checks depend on the message and include the
following.

- canonical schema, length, and collection bounds;
- expected message type and ceremony phase;
- setup and withdrawal identifiers;
- sender identity, membership, and visible signatures;
- ordering, uniqueness, sequence, and freshness;
- Bitcoin transaction syntax, inputs, outputs, amounts, fees, sighash policy,
  descriptor membership, and milestone eligibility; and
- the signatures, identities, sequences, and reached flags in
  `reached_pings_collection`.

Niso rejects an object that fails an applicable check before invoking the next
Boomlet operation. When Boomlet must first decrypt data that Niso cannot read,
Niso performs its remaining checks after Boomlet returns the plaintext and
before Niso presents or transforms it. For example, a non-initiator Niso checks
the reconstructed `withdrawal_id` before presenting the decrypted PSBT to the
user.

Boomlet does not accept a Niso validation result as proof that a condition
holds. Boomlet independently validates every bound, identity, context, state
transition, transaction property, and collection property required for the
Boomlet operation being requested. Only Boomlet's own validation can change
trusted Boomlet state or release its signing share. Niso does not send a
trusted-validity flag that replaces any Boomlet check.

The specification demonstrates this division during withdrawal.

- initiator Niso validates the PSBT before asking Boomlet to derive `tx_id` and
  begin ST review ([SPEC §15.2](../spec/SPEC.md#152-initiator-review));
- non-initiator Niso checks the visible WT and initiator approval state before
  forwarding the encrypted PSBT, while Boomlet repeats those checks and verifies
  the encrypted transaction binding
  ([SPEC §15.4](../spec/SPEC.md#154-non-initiator-review)); and
- Niso and Boomlet independently validate `reached_pings_collection`, with
  Boomlet revalidating the transaction and collection before signing
  ([SPEC §15.11](../spec/SPEC.md#1511-reached-collection),
  [§15.12](../spec/SPEC.md#1512-psbt-hydration)).

## Rationale

An untrusted component can still perform useful validation. Niso's result is
not evidence of user consent and cannot authorize signing, but an honest Niso
can reject corrupt, stale, unauthenticated, out-of-context, or oversized data
before it reaches a user-facing operation or the constrained device.

This ordering prevents any input that fails Niso's admission checks from
forcing the corresponding computation on Boomlet. In particular, a remote
attacker that lacks a valid current peer or WT message cannot use arbitrary
traffic to trigger all available Boomlet signature checks, decryptions, KDF
operations, transaction parsing, or state-transition work. Niso performs the
available host-side checks before the secure-element operation whenever the
protocol makes the required information visible to Niso.

Boomlet repeats security-critical checks because Niso may be compromised or may
contain a validation bug. Resource protection on an honest Niso and
authorization enforcement on Boomlet have different purposes. Neither purpose
eliminates the other.

Niso also needs verified inputs for its own behavior. It should not display an
invalid transaction, hydrate a PSBT for an unapproved ceremony, update local
relay state from a malformed collection, or forward an object under the wrong
identity or context merely because Boomlet will later reject signing.

## Security Effect

- Traffic rejected by Niso cannot trigger the corresponding Boomlet operation.
- Invalid remote peer or WT traffic is rejected before it can consume the full
  set of constrained-device validations when Niso has enough visible information
  to identify the failure.
- Objects that violate Niso-enforceable schema or size bounds are rejected
  before Boomlet allocation or cryptographic processing.
- A compromised Niso can bypass its own filters but cannot authorize signing or
  bypass Boomlet's independent validation.
- A valid authenticated peer, WT, or local ceremony can still cause the
  protocol's expected Boomlet work. Rate limiting, timeout handling, transport
  access control, and service-level denial-of-service defenses remain necessary.
- Niso checks must preserve the protocol's permitted failure behavior and must
  not expose the safe or duress classification.

## Consequences

Niso and Boomlet implement some of the same validation rules. Shared test
vectors must confirm that both interpret canonical encodings, identities,
ordering, freshness, transaction bindings, and collection contents
consistently. A Niso acceptance followed by a Boomlet rejection is a protocol
failure that must not be silently ignored.

Boomlet retains its own input-size bounds, parser defenses, authentication,
replay checks, state checks, and transaction validation. Niso prevalidation is
additional resource protection and does not replace any secure-element check.

Implementations should order Niso checks so inexpensive schema, size, identity,
state, and freshness failures are rejected before expensive cryptographic or
Bitcoin operations. This ordering is a resource-protection rule; it does not
change message meaning or signing authority.

## Rejected Alternatives

- **Validate only on Boomlet.** This preserves the authorization boundary but
  lets malformed, stale, unauthenticated, and oversized traffic consume scarce
  secure-element resources before rejection.
- **Validate only on Niso.** A compromised host could then bypass transaction,
  ceremony, and reached-state checks and request an unauthorized signature.
- **Have Niso attest that it validated the object.** A flag or Niso signature
  does not make an untrusted host authoritative and would add another object
  without removing Boomlet's validation work.
- **Forward all encrypted network traffic directly to Boomlet.** This gives
  remote senders unnecessary access to Boomlet decryption, authentication, and
  parser work even when visible outer fields already prove that the message
  should be rejected.
