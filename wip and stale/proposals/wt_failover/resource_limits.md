# WT failover resource limits

`WT_FAILOVER_V1` fixes these admission limits. Raising one selects a different
profile and requires a fresh setup. These are protocol ceilings, not measured
claims about a particular secure element. An implementation must reserve the
required storage before setup and reject the profile if it cannot do so.

| Resource | Limit |
| --- | --- |
| Peers and roster entries | Exactly 5 peers; 1 to 5 WT entries |
| Active withdrawal and pending switch | One of each per Boomlet |
| Admitted withdrawal IDs per setup | 4, including abandoned and completed attempts |
| Retained local signed fragment | 8,192 canonical bytes per withdrawal; 4 slots |
| Local public resume package | 16,384 canonical bytes |
| All five resume packages | 81,920 bytes, streamed at the host and candidate |
| One encoded control object, including certificate sidecars | 16,384 bytes |
| Encoded intent plus manifest | 2,048 bytes per peer |
| Prepared certificate | 1,024 bytes; exactly five signatures |
| Recovery sidecars | At most 5 distinct prepared certificates |
| Canonical object nesting | At most 12, also constrained by the exact schema |
| Host streaming chunk | At most 1,024 bytes; total checked before first chunk |
| Outstanding placeholder | One exact envelope, at most 512 bytes, per peer |
| Accepted or required unanswered duress challenge | One per peer |
| Locally admitted switch intents per setup | 64, including withdrawn intents |
| Admitted decision instances per setup | 64, including closures of withdrawn-intent combinations |
| Ballots admitted per switch instance | 16, numbered 0 through 15 |
| Successor ballot admission | Exactly local promised ballot plus one |
| Automatic successors per trusted switch admission | 2 |
| Further recovery allowance | Trusted local approval, at most 2 successors each |
| Signature verifications per trusted processing allowance | 128 |
| Durable control transactions per admitted switch | 80 |
| Reserved switch journal | 16,384 bytes, excluding active withdrawal state |
| Reserved replay ledger | 8,192 bytes for 64 intents and 4 withdrawal IDs |

Fragment and package sizes include all wrappers. Oversized PSBTs, fragments,
control proofs, and snapshots are rejected before review or admission, while
ordinary state is still resumable. In-memory parsing uses a bounded streaming
cursor; the aggregate five-package limit is not a RAM reservation. The profile
therefore requires at least 73,728 bytes for fragment slots, one public resume
package, the switch journal, and replay ledger, plus key slots, ordinary state,
atomic-write redundancy and working memory. A hardware implementation reports
those additional allocations separately.

A local placeholder obligation stores the envelope, approved ID, originating
object digest, and gate commitment. The padded object remains in the bounded
resume package. Persist the obligation before exposing its bytes. Deleting or
replacing that slot requires verification of the exact SAR acknowledgment.
A newer safe envelope cannot overwrite an unresolved envelope.

A trusted review reserves an intent slot, journal space, replay entries, and
initial verification and recovery allowances. Lengths, type, counts, scope,
head, candidate, ballot range, local tombstones, and duplicate digest are checked
before allocation, signature verification, or a persistent update. Each admitted
signature check consumes the allowance before cryptographic work; failed checks
also consume it. Exhaustion requires trusted replenishment. Packet receipt and
host timers never replenish an allowance or reset a persistent setup limit.
Protected duplicate caches return exact saved results without another signature
or write. Reboot either recovers the remaining allowance or requires trusted
admission; it never silently restores an allowance already consumed.

A device can debit a complete verification allowance durably at trusted
admission, then consume its checks from volatile memory. Reboot discards any
unused portion of that debited allowance. This avoids a persistent write for
each verification while preventing host-driven replenishment. Further trusted
admission must reserve its own control transaction and remaining endurance.

A host proposal cannot persist a higher promise by itself. With a trusted
allowance, Boomlet can admit its next successor ballot; same-ballot retries
return the stored report. Reports for an ahead ballot are buffered externally
until local admission reaches that ballot. No numeric jump, recursive history,
or peer assertion about a missing journal is accepted. A valid final certificate
for the local unresolved instance can always be learned irrespective of its
ballot, subject to one bounded validation allowance. It does not require a new
vote or restore permission to emit an old one.

The journal reserves one current report, own current votes, the highest complete
prepared certificate, original prerequisites, and one final certificate. Higher
valid certificates replace the retained certificate. Older reports
and votes can be discarded only after the higher promise makes reissuance
impossible. Retain the exact signed intent in the live journal. Each replay-ledger entry is
at most 112 internal bytes: intent digest, first admitted switch ID, predecessor
head, disposition and allocation flags. Four 128-byte withdrawal records retain
IDs, initialized status and fragment commitments. These fixed records occupy
7,680 bytes within the 8,192-byte ledger reservation. Exact archived public
intents and final certificates can be retrieved and checked against those
commitments; the highest prepared certificate itself stays on Boomlet because
its complete five-signature proof is required for recovery.
External peers retain final certificates and public packages for catch-up.

The 80 control-transaction budget reserves the final install or cancellation
write before any vote. Duplicate requests consume none. An implementation must
map each logical transaction, including interrupted transactions, to its physical
write and erase cost, reserve crash recovery capacity, and enforce the hardware's
remaining endurance before admitting the ceremony. Failure of that admission
preserves the existing journal. Power interruption cannot grant a fresh write
budget. Physical damage and deliberate power cycling can still deny service.

A full ledger, fourth closed withdrawal, sixteenth ballot, or exhausted hardware
budget cannot trigger eviction, wraparound, automatic cancellation, authority
transfer, or a counter reset. Exact retransmission, certificate learning, and
finishing the already reserved operation remain available when their required
storage is healthy. Further new work after setup limits are exhausted requires
a fresh setup and fund rollover. Recovery must preserve consumed counts.
The self-contained-ping candidate needs a separate byte and endurance budget
for encrypted checkpoints, without increasing ordinary signature counts.
Its complete snapshot inventory and holder-retention guarantees are not yet
specified; existing public-package limits cannot be treated as proof of capacity
for private recovery state. There is no continuous ST or backup recording budget.
Unresolved decisions and finite limits can expose forced-fallback risks; they
cannot be resolved by erasing locks or treating fallback as equivalent protection.

## Silence and retry admission

A deployment must publish finite budgets, in milliseconds, for the existing
`SAR_ACK_DELAY`, bounded WT and SAR processing, a complete routed round trip,
and chain observation. Define:

```text
MIN_SWITCH_SILENCE_MS = SAR_ACK_DELAY_MS
                     + WT_SAR_PROCESSING_BUDGET_MS
                     + ROUND_TRIP_ROUTING_BUDGET_MS
                     + CHAIN_OBSERVATION_BUDGET_MS
```

The host can issue a silence prompt only after a fully submitted valid request
has waited strictly longer than this sum, as well as satisfying the ordinary
block-height silence rule. Unknown or contradictory timing restarts advisory
observation; it grants no trusted state transition. A deployment with no
measured finite timing bounds cannot enable automatic silence prompts.
The same conservative lower bound applies to requests whose SAR duty was
already acknowledged, avoiding classification-dependent timer behavior.

An exact in-flight acknowledgment may arrive while frozen and discharge only
its matched duty. Retransmission through a candidate retains the inner IV,
approved ID, and original SAR replay deadline; it cannot select a faster release
path. The SAR's existing fixed-deadline failure policy applies. Freeze and retry
counts depend on public request state and user action, never decrypted safe or
duress status. Generic heartbeats and partial responses do not count as progress.
