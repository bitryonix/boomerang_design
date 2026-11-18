# Boomletwo recovery from self-contained pings

A self-contained ping can carry the state needed to interpret and resume a
DIGGING checkpoint. Boomletwo can remain offline while peers, WT and Niso retain
the packet. ST participates in its ordinary user interactions and an eventual
recovery review, with no continuous recording duty.

This proposal extends [SPEC.md](../../../spec/SPEC.md) and the [WT failover
profile](README.md). Their requirements apply unless explicitly changed below.
Source exclusion and later-state recovery remain unresolved, so the checkpoint
does not authorize activation. Recovery assumes no source cooperation.

## High-level flow

1. Encrypt a bounded recovery checkpoint to the designated Boomletwo and bind it
   into an ordinary signed ping.
2. Have Niso, WT and peers retain the ping bundle while normal DIGGING continues.
3. After source loss, verify the checkpoint, setup binding, SAR duty and recorded WT head.
4. Establish source exclusion and account for later security state before enabling recovery.
5. Restore progress under a conservative threshold and resume ordinary protocol gates;
   remain inactive if the activation safeguards are incomplete.

## 1. Packet and setup binding

The SPEC Section 13.10 backup fixes the only eligible Boomletwo and immutable
setup policy. Recovery cannot select another target or change that policy.

A recovery-capable ping bundle contains:

- the ordinary ping header and exact encrypted SAR placeholder;
- a bounded checkpoint encrypted and authenticated directly to the designated
  Boomletwo under the SPEC cryptographic profile;
- a commitment to that ciphertext inside the ping's existing signed content.

The ciphertext is a detached part of the same bundle. The commitment lets peers
forward the signed ping without copying a large private checkpoint into every
collection. The bundle itself includes the bytes, not only a URL or hash. Any
holder can supply it, but no holder gains recovery authority.

Bind the encryption to the profile, setup, target, approved withdrawal ID, ping
sequence and checkpoint format. The header, checkpoint and placeholder must
agree. Apply SPEC IV and retry rules, and authenticate retained ciphertext before
decryption.

The checkpoint contains the actual counter, initialized status, original mystery
if the selected delay policy uses it, reached flag, block height and spacing floors,
peer sequence and reached-state checks, active WT head and index, exact current
SAR duty, accepted duress state and required challenge state. It also binds the
approved transaction and the evidence needed for its current phase. All private
fields stay encrypted. A ping sequence is not a counter: valid no-advance pongs
still generate pings in the base protocol.

Construct the new placeholder and the checkpoint before signing the ping. The
checkpoint represents the state associated with that output, with explicit next
sequence and pending acknowledgment. It must not embed the signature, ciphertext
or digest of the complete packet containing itself. Required header fields and
placeholder bytes provide the binding without a recursive hash definition.
Store only bounded state and references to supplied evidence, not nested earlier
checkpoint packets. Exact field layout and size limits remain conformance work.

The ping signature covers the added commitment. Its WT-neutral recovery context
survives a WT change; current authority still requires the head-bound WT checks.

## 2. Including the recipient's own ping in pong

If “self-inclusive” also means returning a peer's own ping to it, the pong can
contain one ordered collection of all five signed pings. WT can sign that common
pong content once, while recipient encryption and each recipient's exact SAR
acknowledgment remain separate. This is a proposed change from the base's five
recipient-specific four-ping collections.

Require exactly one ping from each setup peer in canonical peer order, including
the recipient in its own slot. Missing, repeated or substituted identities fail.

Boomlet compares its own returned ping with retained bytes. Boomletwo instead
verifies the source signature and checkpoint binding. SPEC validation applies to
the other four pings.

Counter advancement still tests only the other four peers. An old self ping and
historical pong earn no new increment. Exact SAR discharge remains mandatory.

The change saves common WT signing work but adds an own-ping header to each
recipient's collection. Detached checkpoints avoid multiplying private-state
payloads across those collections. These are proposed wire changes, requiring
consistent version-selected handlers; they cannot be mixed with the base shape.

## 3. An independently generated mystery

Let `c` be the authenticated saved counter, `M` the original mystery, and `R`
a new independent draw by Boomletwo. For an unreached checkpoint, `c < M`.

Installing `R` as the threshold while restoring `c` is unsafe. With a draw from
1 through 100 and `c = 80`, 80 percent of replacements would immediately become
reached. Even excluding draws at or below `c` can shorten the actual remaining
work and gives a second completion opportunity if the source still exists.

There are two useful policies:

| Policy | Effective recovery threshold | Consequence |
| --- | --- | --- |
| Retain an encrypted original floor | `T = max(M, R)` | The new draw is independent, but recovery never lowers this withdrawal's original threshold |
| Do not carry the original mystery | `T = U + R`, where `U` is the setup-fixed upper bound on `M` | Recovery has an independent threshold above every possible original draw, at the cost of additional delay |

Both restore the authenticated `c` and require subsequent valid progress until
that counter reaches `T`. For the first policy,

```text
remaining_recovery = max(0, max(M, R) - c)
remaining_original = max(0, M - c)
remaining_recovery >= remaining_original
```

For the second, `U + R >= M`. Checked arithmetic and a setup-selected extended
counter range are required. The extra delay must fit the fallback planning
budget; making deterministic fallback more likely would undermine the purpose.
The first policy is the smaller change. The second meets a strict requirement
that the effective recovery threshold be independent of the original draw.

Generate `R` once after the exact checkpoint and recovery attempt have been
irrevocably bound. Persist it before any observable result. Retries, power loss,
a different peer response or a later snapshot cannot trigger another draw for
that attempt. No preview, diagnostic or trial-activation interface exposes its
value or whether a candidate draw would finish early.

An authenticated reached checkpoint proves that the old threshold was met at
that point. It does not prove that later rescue duties are absent or authorize
signing by itself. A new delay must not silently regress an already published
reached flag in the same ping stream; any such policy needs explicit incarnation
semantics. The formulas above apply to an unreached checkpoint.

Starting a fresh counter at zero neither preserves the realized delay nor solves
authority or obligation recovery. No threshold policy completes activation.

## 4. What a saved ping proves

| Claim | Supported by a valid packet? |
| --- | --- |
| The designated source authenticated these bytes for this setup and withdrawal | Yes, under its signature and encryption checks |
| The source had accepted at least the checkpoint's recorded progress | Yes, under the base honest-device assumption |
| This packet names the WT head at that checkpoint | Yes |
| This is the last ping or the currently installed WT head | No |
| No later duress challenge, placeholder, vote or signing output exists | No |
| The source is destroyed or unable to continue | No |

A signed packet authenticates one snapshot. It cannot turn missing duties,
tombstones or votes into harmless lost progress.

For example, the host may retain a safe checkpoint, suppress a later ping with
an undelivered duress placeholder, and present the safe checkpoint at recovery.
The source could also have emitted a WT COMMIT after its last ping; four peers
can conceal that vote and its final certificate. Recovering “no pending vote”
from the old packet could then create a conflicting decision.

More checkpoint fields do not cover later events, and an untrusted holder can
conceal the newest checkpoint.

## 5. Peer WT discovery

Peers may supply the `WtReady` record matching the checkpoint's WT head. Verify
its canonical setup, scope and roster index, then use the existing commitments:

```text
ready_commitment = H("Boomerang/wt/candidate_ready", ready_record)
lookup_head = H("Boomerang/wt/activation",
  ready_record.scope.setup_instance_id,
  ready_record.scope.previous_head,
  ready_record.scope.switch_id,
  ready_commitment)
require lookup_head == checkpoint.active_wt_head
```

`H` uses the setup's existing WT profile. One matching record suffices without
fresh peer signatures. It proves the WT identity at that checkpoint. A newer
head requires the ordinary authenticated decision history and recovery rules;
choose neither the greatest claimed index nor the majority's assertion.

A fresh challenge to all four survivors authenticates their present responses,
not their completeness or truthfulness. When the lost device belonged to the
only honest peer, every survivor can lie. Discovery cannot supply the missing
source vote or establish exclusive activation.

## 6. Activation requirements beyond the ping

Source-loss handling must remain safe when the original is isolated rather than
destroyed. A new incarnation or peer promise cannot fence it. The threshold
floor prevents delay reduction but not concurrent authority or conflicting WT
histories.

Before enabling signing or WT voting on Boomletwo, a complete design must supply:

- an enforceable exclusion mechanism for the old device's new authority;
- a rule for later or concealed rescue duties, decisions and replay state;
- authenticated reconstruction of the selected transaction and required evidence;
- one-time activation, replay protection and fresh local signing nonces;
- all inherited protocol gates, chain assumptions and a conservative progress floor.

Historical pongs cannot be replayed to earn fresh progress. Recovery must not
convert offline elapsed time, ping count, checkpoint selection or speculative
block-height catch-up into extra counter increments. The exact fresh-progress and
incarnation rules require specification before using the numerical delay bound
as a claim about elapsed time. The base authenticated-chain issue remains open.

The first two guarantees remain unsolved under the one-honest-peer model. No
activation command is authorized solely by a checkpoint or peer reports.

## 7. Cost and verification

The candidate adds one bounded private-state encryption and signed commitment
to a ping, with no extra source signature. A full five-ping pong can share its
WT signature across recipients. Normal Boomlet self-ping validation can use
exact retained-byte comparison; recovery performs source verification and one
checkpoint decryption. ST has no recurring recording role, and Boomletwo has no
normal-operation power or connectivity requirement.

The snapshot format, limits, retention, incarnation rules and activation
mechanism remain unfinished. Checkpoints cannot alter private policy or restore
secret signing nonces; SPEC confidentiality and duress-shape rules apply.

The [bounded checks](boomlet_rollover_model.md) demonstrate threshold bounds,
redraw counterexamples and stale-checkpoint indistinguishability. They do not
prove a complete activation protocol. The [sequence](09_boomlet_rollover.puml)
shows packet preparation and guarded recovery, with unresolved activation
requirements explicit.
